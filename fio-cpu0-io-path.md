# fio 在 CPU0 上运行：blk-mq 软件队列、硬件队列与 IO Merge 时机详解

> 场景：博通 HBA，2个硬件队列（HQ0/HQ1），HQ0 的 MSI-X 中断 affinity 绑定 CPU0 + CPU1。
> fio 运行在 CPU0，使用 mq-deadline 调度器。

---

## 1. 整体结构全景图

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                        fio 进程（绑定 CPU0）                                     ║
║                                                                                  ║
║  fio线程发出 pwrite/pread 系统调用                                                ║
║       │                                                                          ║
║       ▼                                                                          ║
║  VFS/文件系统 → bio 构造 → blk_mq_submit_bio()                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝
         │
         │（CPU0 上下文，可抢占，但尚未切换）
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    【Merge 时机 ①：plug 层 bio merge】                           │
│                                                                                  │
│  current->plug  （struct blk_plug，per-进程/per-线程）                           │
│  ├── mq_list:  [bio_A] → [bio_B] → [bio_C]  ← 还没变成 request                 │
│  └── cached_rq: 预分配的 struct request（批量减少 sched_tag 分配开销）            │
│                                                                                  │
│  blk_bio_list_merge(q, &plug->mq_list, new_bio):                                │
│    遍历 plug->mq_list（最多8个），检查 bio 尾部扇区能否接上已有 bio               │
│    ✓ 成功：new_bio 直接合入已有 bio/request，扇区范围扩展                         │
│    ✗ 失败：new_bio 加入 plug->mq_list 等待后续处理                               │
│                                                                                  │
│  【触发时机】：blk_finish_plug() / blk_flush_plug()                              │
│    ─ fio 调用频繁 IO 时，plug 会在 task_work 或主动 flush 时下发                  │
└─────────────────────────────────────────────────────────────────────────────────┘
         │ plug flush（批量）
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    【Merge 时机 ②：bio → scheduler bio merge】                   │
│                                                                                  │
│  blk_mq_submit_bio(bio)                                                         │
│    └─ blk_mq_sched_bio_merge(q, bio, nr_segs)                                   │
│         └─ dd_bio_merge(q, bio, nr_segs)   【mq-deadline 回调】                  │
│              spin_lock(&dd->lock)                                                │
│              blk_mq_sched_try_merge(q, bio, nr_segs, &free):                    │
│                ① blk_attempt_bio_merge(q, q->last_merge, bio)                   │
│                   检查 bio 能否接在 last_merge request 的尾部                     │
│                ② elv_rqhash_find(q, bio_end_sector(bio))                        │
│                   在哈希表中找扇区匹配的 request（尾部 merge）                    │
│                ③ elevator->ops.request_merge() → dd_request_merge()             │
│                   elv_rb_find(sort_list, bio_end_sector)                         │
│                   在红黑树中找 request（前向 merge：bio 接在 request 前面）       │
│              spin_unlock(&dd->lock)                                              │
│                                                                                  │
│  ✓ bio merge 成功：bio 合入已有 request，无需分配新 request，直接返回             │
│  ✗ bio merge 失败：继续分配新 struct request                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
         │ merge 失败，分配新 request
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    request 分配（绑定 CPU0 的上下文）                             │
│                                                                                  │
│  data->ctx  = blk_mq_get_ctx(q) = per_cpu(blk_mq_ctx, 0)  ← ctx0 (CPU0)       │
│  data->hctx = ctx0->hctxs[DEFAULT] = hctx0                 ← HQ0 的内核对象    │
│                                                                                  │
│  从 hctx0->sched_tags 分配 internal_tag（调度器 tag，不是驱动 tag）              │
│  rq->mq_ctx  = ctx0   ← 记录"我是 CPU0 提交的"                                 │
│  rq->mq_hctx = hctx0  ← 记录"我属于 HQ0"                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    【Merge 时机 ③：insert merge（request 级别）】                │
│                                                                                  │
│  dd_insert_requests(hctx0, list, flags)                                         │
│    spin_lock(&dd->lock)                                                          │
│    dd_insert_request(hctx0, rq, flags, &free):                                  │
│      blk_mq_sched_try_insert_merge(q, rq, &free):                               │
│        elv_attempt_insert_merge(q, rq, free):                                   │
│          ① 检查 q->last_merge（上次 insert 的 request）                          │
│          ② elv_rqhash_find(q, blk_rq_pos(rq))  ← 找能接在 rq 后面的 existing   │
│          attempt_merge(q, existing_rq, rq):                                     │
│            ✓ 成功：rq 的 bio 链接到 existing_rq，rq 本身被释放（freed list）     │
│                     dd_merged_requests() 调整 fifo_time 取较早值                 │
│                     deadline_remove_request() 从红黑树+FIFO 删除被吞并的 rq      │
│      ✗ insert merge 失败：                                                       │
│        deadline_add_rq_rb(per_prio, rq)      ← 插入红黑树（扇区排序）           │
│        elv_rqhash_add(q, rq)                 ← 插入哈希表（加速后续 merge 查找） │
│        list_add_tail(fifo_list[dir], rq)     ← 插入 FIFO（超时排序）            │
│        q->last_merge = rq                                                        │
│    spin_unlock(&dd->lock)                                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
         │ request 成功进入 deadline_data 的 sort_list + fifo_list
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    mq-deadline 全局调度状态（单份，全队列共享）                   │
│                                                                                  │
│  deadline_data                                                                   │
│  ├── per_prio[RT=0]                                                              │
│  │   ├── sort_list[READ]  (红黑树：sector有序)                                   │
│  │   ├── sort_list[WRITE] (红黑树：sector有序)                                   │
│  │   ├── fifo_list[READ]  (链表：fifo_time有序，超时先出)                        │
│  │   ├── fifo_list[WRITE] (链表：fifo_time有序)                                  │
│  │   └── dispatch         (链表：被 requeue 的请求，优先下发)                    │
│  ├── per_prio[BE=1]       （同上结构，fio 默认 BE 优先级）                       │
│  ├── per_prio[IDLE=2]     （同上结构）                                           │
│  └── lock                 （单把锁，序列化所有 insert 和 dispatch）              │
│                                                                                  │
│  注：CPU0 和 CPU1 提交的请求都在这一份数据结构里混合排序！                        │
└─────────────────────────────────────────────────────────────────────────────────┘
         │ dispatch（由 kworker 在 hctx0->cpumask 的 CPU 上运行）
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    dispatch → 驱动 → 博通 HBA HQ0                               │
│                                                                                  │
│  dd_dispatch_request(hctx0)                                                     │
│    → 从 sort_list/fifo_list 按优先级+扇区顺序+超时选出 rq                       │
│    → blk_mq_get_driver_tag(rq)  分配真正的硬件 tag                              │
│    → hba_ops->queue_rq(hctx0, rq)  写入 HQ0 ring buffer                        │
│    → doorbell 通知博通 HBA 芯片                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. CPU0 提交 IO 时的软件队列和硬件队列详细图

```
CPU0（fio 进程运行在此）
══════════════════════════════════════════════════════════════════════

  进程上下文（per-task）
  ┌─────────────────────────────────────────────┐
  │  struct blk_plug  (current->plug)            │
  │  ├── mq_list: [rq1]→[rq2]→[rq3]            │  ← ① Plug 层暂存
  │  │   (plug 期间积累，尚未进调度器)           │
  │  └── cached_rq: [预分配rq池]                │
  └─────────────────────────────────────────────┘
         │ blk_finish_plug() 触发 flush
         │ 或单个 bio 无法 plug 时直接下发
         ▼

  per-CPU 软件队列（无调度器时才使用 rq_lists）
  ┌─────────────────────────────────────────────┐
  │  struct blk_mq_ctx  ctx0  (CPU0 专属)        │
  │  ├── cpu = 0                                 │
  │  ├── index_hw[DEFAULT] = 0  → hctx0         │
  │  ├── hctxs[DEFAULT]   = hctx0               │
  │  └── rq_lists[DEFAULT]: [空]  ← 有调度器时  │
  │                                不用这个列表！ │
  └─────────────────────────────────────────────┘
         │ dd_insert_requests(hctx0, ...)
         │ （有 mq-deadline 时，请求直接插入调度器，不经过 rq_lists）
         ▼

  调度器全局队列（deadline_data，全队列唯一一份）
  ┌─────────────────────────────────────────────────────────────────┐
  │  per_prio[BE]（fio 默认优先级）                                  │
  │                                                                  │
  │  sort_list[READ]（红黑树，按扇区号排序）                         │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  sector:100  sector:108  sector:200  sector:500  ...     │   │
  │  │  rq_A(cpu0)  rq_B(cpu1)  rq_C(cpu0)  rq_D(cpu1)  ...   │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │  （注：cpu0 和 cpu1 的请求混合在同一棵红黑树里！）               │
  │                                                                  │
  │  fifo_list[READ]（链表，按 fifo_time 即到期时间排序）            │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  rq_A(T+500ms) → rq_C(T+500ms) → rq_B(T+500ms) → ...   │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │  （哪个先插入，哪个先到期，不区分来自哪个 CPU）                  │
  └─────────────────────────────────────────────────────────────────┘
         │ blk_mq_run_hw_queue(hctx0)
         │ kworker 运行在 hctx0->cpumask 中的某个 CPU（CPU0 或 CPU1）
         ▼

  硬件队列（hctx0，对应博通 HQ0）
  ┌─────────────────────────────────────────────┐
  │  struct blk_mq_hw_ctx  hctx0                │
  │  ├── queue_num = 0                           │
  │  ├── cpumask = {CPU0, CPU1}                  │
  │  ├── nr_ctx  = 2                             │
  │  ├── ctxs[] = [ctx0, ctx1]                  │
  │  ├── sched_tags  ← internal_tag 池          │
  │  │   （request 分配时就占用，dispatch 后    │
  │  │    才换成 driver tag）                    │
  │  ├── tags        ← driver tag 池            │
  │  │   （queue_rq 前调用 get_driver_tag 分配）│
  │  └── dispatch    ← bypass/requeue 暂存列表 │
  └─────────────────────────────────────────────┘
         │ hba_ops->queue_rq()
         ▼

  博通 HBA 物理硬件
  ┌─────────────────────────────────────────────┐
  │  HQ0  ring buffer（提交队列 SQ）             │
  │  [cmd_0][cmd_1][cmd_2]... ← 写入命令        │
  │  doorbell 寄存器 ← 通知 HBA                 │
  │  HQ0 完成队列 CQ ← HBA 填写完成条目        │
  │  MSI-X IRQ #0  → CPU0 或 CPU1 处理中断      │
  └─────────────────────────────────────────────┘
```

---

## 3. IO Merge 时机对比：CPU0 vs CPU1

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        IO Merge 时机全览                                        │
│                                                                                 │
│   时机①           时机②              时机③                  时机④             │
│  Plug层merge    bio→sched merge     insert merge          dispatch时无merge    │
│  (per-task)     (调度器层)          (调度器层)                                  │
└────────────────────────────────────────────────────────────────────────────────┘

【CPU0（fio 线程直接运行在此）】
────────────────────────────────────────────────────────────────────────────────

时机① Plug 层 bio merge（CPU0 独享，其他 CPU 无法干涉）
  │
  │  current->plug->mq_list: [bio_A: sector100-107]
  │                            ↑
  │  fio 提交 bio_B(sector108-115):
  │  blk_mq_bio_to_request() 之前先调用：
  │  blk_bio_list_merge(q, &plug->mq_list, bio_B)
  │    ─ bio_A 尾部 = bio_B 起始（108 == 108）？YES
  │    ─ bio_B 直接合入 bio_A，无锁，无 dd->lock
  │    ─ bio_A 扇区范围变为 100-115
  │
  │  ✓ CPU0 专属操作，不涉及任何锁竞争
  │  ✓ 发生在 request 分配之前，最高效
  │  ✓ fio 顺序 IO 场景下，大量 bio 在这里合并，显著减少 request 数量

  结果：多个连续 bio 合成一个 bio，后续只分配一个 request


时机② bio → 调度器 bio merge（CPU0 调用，需要 dd->lock）
  │
  │  plug flush 或单个 bio 直接提交时触发：
  │  blk_mq_submit_bio(bio_X) → dd_bio_merge(q, bio_X, nr_segs)
  │    spin_lock(&dd->lock)   ← 竞争！CPU1 的 insert 可能同时持有此锁
  │    blk_mq_sched_try_merge():
  │      检查 q->last_merge（最近 insert 的 request，任意 CPU 都可能设置）
  │      elv_rqhash_find() → 在 hash 表里找 sector 匹配的 rq
  │      dd_request_merge() → elv_rb_find(sort_list) → 前向 merge 查找
  │    spin_unlock(&dd->lock)
  │
  │  ✓ 可与任意 CPU 提交的已有 request 合并（例如与 CPU1 的 rq_B 合并）
  │  ! 需要 dd->lock，与 CPU1 的 insert 操作互斥
  │  ! 找到的是调度器全局 sort_list 中的 request（已经进入调度器的）

  合并成功：bio_X 合入已有 request（可能是 CPU1 提交的！），不再分配新 request
  合并失败：进入下一步，分配新 request


时机③ insert merge（CPU0 分配了新 request 后，insert 时再尝试）
  │
  │  dd_insert_requests(hctx0, rq_X, flags)
  │    spin_lock(&dd->lock)
  │    blk_mq_sched_try_insert_merge(q, rq_X, &free):
  │      elv_attempt_insert_merge():
  │        找到可合并的 existing_rq（任意 CPU 提交的均可）
  │        attempt_merge() → dd_merged_requests():
  │          取 min(rq_X->fifo_time, existing_rq->fifo_time)
  │          deadline_remove_request(q, per_prio, rq_X)
  │          ← 从红黑树、FIFO、hash 全部移除 rq_X
  │          rq_X 放入 free 列表，blk_mq_free_request() 释放
  │    spin_unlock(&dd->lock)
  │
  │  ✓ CPU0 提交的 rq_X 可以被合入 CPU1 已有的 existing_rq（或反之）
  │  ! 仍然需要 dd->lock


【CPU1（其他 IO 线程，kworker，或中断处理时的 workqueue 线程）】
────────────────────────────────────────────────────────────────────────────────

时机① Plug 层 bio merge（CPU1 独享，与 CPU0 完全隔离）
  │
  │  CPU1 上运行的线程有自己的 current->plug
  │  current->plug->mq_list: [bio_E: sector300-307]
  │                            ↑
  │  CPU1 提交 bio_F(sector308-315):
  │  blk_bio_list_merge(q, &plug->mq_list, bio_F)
  │    ─ 完全在 CPU1 的进程上下文中操作，无锁
  │    ─ CPU0 的 plug 和 CPU1 的 plug 是完全独立的两个 blk_plug 结构
  │
  │  ✓ CPU1 专属，CPU0 和 CPU1 的 plug merge 并行进行，互不干扰
  │  ✗ CPU1 的 plug bio 不能与 CPU0 的 plug bio 合并（不在同一 plug）


时机② bio → 调度器 bio merge（CPU1 调用，需要 dd->lock，与 CPU0 竞争）
  │
  │  与 CPU0 的时机② 完全对称，但：
  │  blk_mq_get_ctx(q) 返回 per_cpu(blk_mq_ctx, 1) = ctx1
  │  ctx1->hctxs[DEFAULT] = hctx0  ← 同一个 hctx0！（两 CPU 共享 HQ0）
  │
  │  dd_bio_merge(q, bio_E, nr_segs):
  │    spin_lock(&dd->lock)  ← 与 CPU0 的 insert/merge 竞争同一把锁！
  │    blk_mq_sched_try_merge():
  │      在 sort_list 中可以找到 CPU0 插入的 request，并将 bio_E 合入！
  │    spin_unlock(&dd->lock)
  │
  │  ✓ CPU1 的 bio 可以合入 CPU0 已插入调度器的 request
  │  ! 锁竞争：CPU0 和 CPU1 并发时，一方需要等待另一方释放 dd->lock


时机③ insert merge（CPU1，需要 dd->lock）
  │
  │  与 CPU0 的时机③ 完全对称
  │  CPU1 的 rq_F 可以被合入 CPU0 的 existing_rq（两者在同一 sort_list）
  │  dd_merged_requests() 调整 fifo_time，移除 rq_F，释放
  │
  │  ✓ CPU1 提交的 request 可以与 CPU0 提交的 request 合并
  │  ! 需要 dd->lock，仍然有竞争
```

---

## 4. 时序图：fio(CPU0) 连续提交 3 个 IO，同时 CPU1 也有 IO

```
时间轴 →

CPU0 (fio)    CPU1 (其他线程)    dd->lock    deadline_data        博通 HQ0
────────────  ─────────────────  ─────────   ──────────────────   ──────────

t1: fio提交bio_A(s=100)
    → plug: bio_A 存入 mq_list
    （无锁，极快）

t2: fio提交bio_B(s=108)
    → plug: bio_B连续？YES
    → bio_B合入bio_A（plug merge①）
    → mq_list: [bio_AB: s=100-115]
    （无锁，极快）

t3: fio提交bio_C(s=300)
    → plug: 不连续
    → mq_list: [bio_AB, bio_C]

                  t4: CPU1提交bio_D(s=116)
                      → 无plug（或plug flush）
                      → dd_bio_merge():
                      → TRY LOCK dd->lock ──────→ [LOCK获取]
                      → last_merge? 此时还没有
                      → UNLOCK ────────────────→ [UNLOCK]
                      → merge失败
                      → 分配rq_D(s=116)
                      → dd_insert_requests():
                      → LOCK dd->lock ──────────→ [LOCK获取]
                      → insert_merge? 无         sort_list[READ]:
                      → deadline_add_rq_rb()  →  [rq_D:s116]
                      → fifo_list_add_tail()  →  fifo=[rq_D]
                      → last_merge = rq_D
                      → UNLOCK ────────────────→ [UNLOCK]

t5: fio调用blk_finish_plug()
    → flush plug: bio_AB, bio_C
    → 处理 bio_AB(s=100-115):
    → dd_bio_merge():
    → TRY LOCK dd->lock ────────────────────→ [LOCK获取]
    → last_merge = rq_D(s116)
    → bio_end_sector(bio_AB) = 116
    → 116 == blk_rq_pos(rq_D) = 116? YES!   sort_list[READ]:
    → bio_AB 合入 rq_D（merge②）         →  [rq_D:s100-131]
    → rq_D 扇区范围从116扩展到100-131        （rq_D 在红黑树中位置更新）
    → UNLOCK ──────────────────────────────→ [UNLOCK]
    → bio_AB 不需要新分配 request！

    → 处理 bio_C(s=300):
    → dd_bio_merge():
    → LOCK dd->lock ────────────────────────→ [LOCK获取]
    → hash/rbtree 查找：无匹配              sort_list[READ]:
    → UNLOCK ──────────────────────────────→ [UNLOCK]
    → 分配 rq_C(s=300)
    → dd_insert_requests():
    → LOCK dd->lock ────────────────────────→ [LOCK获取]
    → insert_merge? s=300 无邻接           sort_list[READ]:
    → deadline_add_rq_rb(rq_C) ──────────→  [rq_D:100-131] [rq_C:300]
    → fifo_list_add_tail(rq_C) ──────────→  fifo=[rq_D, rq_C]
    → last_merge = rq_C
    → UNLOCK ──────────────────────────────→ [UNLOCK]

t6:                                          dispatch触发（kworker on CPU0/1）
                                             dd_dispatch_request(hctx0):
                                             LOCK dd->lock
                                             选出 rq_D(s=100-131, 最早insert)
                                             或 rq_C(s=300, batch/sector顺序)
                                             deadline算法决策...
                                             UNLOCK
                                             get_driver_tag(rq_D)
                                             queue_rq(hctx0, rq_D) ──────────→ HQ0[cmd_0]
                                             doorbell ────────────────────────→ 通知HBA

t7:                                                                              HBA处理完成
                                                                                 MSI-X IRQ#0
                                                                                 → CPU0或CPU1
                                                                                 处理中断
                                             dd_finish_request(rq_D):
                                             atomic_inc(completed)
                                             driver_tag 释放
                                             触发新的 dispatch
```

---

## 5. 关键结论：CPU0 vs CPU1 的 Merge 差异

```
┌──────────────┬─────────────────────────────────┬─────────────────────────────────┐
│  Merge 时机  │         CPU0 (fio 线程)          │         CPU1 (其他线程)          │
├──────────────┼─────────────────────────────────┼─────────────────────────────────┤
│ ① Plug merge │ ✓ current->plug 独享             │ ✓ 自己的 current->plug 独享      │
│              │   无锁，极高效                   │   无锁，极高效                    │
│              │   CPU0 的 bio 之间可合并          │   CPU1 的 bio 之间可合并          │
│              │ ✗ 不能与 CPU1 的 plug bio 合并   │ ✗ 不能与 CPU0 的 plug bio 合并   │
├──────────────┼─────────────────────────────────┼─────────────────────────────────┤
│ ② bio merge  │ ✓ 可与任意 CPU 已入调度器的 rq   │ ✓ 可与任意 CPU 已入调度器的 rq   │
│  (调度器层)  │   合并（含 CPU1 的 rq）           │   合并（含 CPU0 的 rq）           │
│              │ ! 需要 dd->lock（与 CPU1 竞争）  │ ! 需要 dd->lock（与 CPU0 竞争）  │
│              │   全队列共享一把锁               │   全队列共享一把锁                │
├──────────────┼─────────────────────────────────┼─────────────────────────────────┤
│ ③ insert     │ ✓ 可与任意 CPU 的 rq 合并        │ ✓ 可与任意 CPU 的 rq 合并        │
│  merge       │ ! 需要 dd->lock                  │ ! 需要 dd->lock                  │
│              │   仅在分配了新 request 之后       │   仅在分配了新 request 之后       │
│              │   尝试（比 ② 代价更高）           │   尝试（比 ② 代价更高）           │
├──────────────┼─────────────────────────────────┼─────────────────────────────────┤
│  本质区别    │ CPU0 和 CPU1 共享 hctx0，         │ 共享 hctx0，但 ctx 是独立的       │
│              │ 但调度器层完全无视 CPU 边界，      │ plug 层完全隔离，调度器层完全混合 │
│              │ 只看扇区是否相邻                  │ 只看扇区是否相邻                  │
└──────────────┴─────────────────────────────────┴─────────────────────────────────┘

fio 顺序 IO 场景下 merge 效率：
  CPU0 顺序写 → Plug merge 合并大量 bio → 每次 flush 只产生少量 request
  → 调度器层 merge 机会减少（已经合并好了）
  → sort_list 中请求不多，dispatch 快

fio 随机 IO 场景下 merge：
  CPU0 随机写 → Plug merge 几乎不发生（扇区不连续）
  → 每个 bio 单独进入调度器
  → 调度器层 merge 偶尔发生（概率低）
  → sort_list 中大量分散请求，mq-deadline 靠 FIFO 超时保证延迟
```

---

## 6. 为什么 CPU0 和 CPU1 共享一个 hctx0 但仍用独立 ctx

```
                      博通 HBA
                      ┌──────────────────────────────────────────┐
                      │  HQ0 (硬件提交队列)                       │
                      │  capacity = 256 commands                  │
                      │  MSI-X IRQ #0  affinity → {CPU0, CPU1}   │
                      └──────────────────────────────────────────┘
                              ↑                    ↑
                   queue_rq(hctx0, rq)      queue_rq(hctx0, rq)
                          ↑                         ↑
              ┌───────────┴───────────┐             │
              │    hctx0 dispatch     │             │
              │  (kworker轮流用CPU0/1)│             │
              └───────────────────────┘             │
                              ↑ dd_dispatch (全局顺序)
              ┌───────────────────────────────────────────────┐
              │           deadline_data (全局唯一)             │
              │   sort_list[READ]: ...rqA(cpu0)...rqD(cpu1).. │
              │   fifo_list[READ]: rqA → rqB → rqC → rqD     │
              └───────────────────────────────────────────────┘
                    ↑ insert(dd->lock)    ↑ insert(dd->lock)
              ┌─────┴──────┐        ┌─────┴──────┐
              │    ctx0    │        │    ctx1    │
              │  (CPU0专属)│        │  (CPU1专属)│
              │ index_hw=0 │        │ index_hw=0 │
              └─────┬──────┘        └─────┬──────┘
                    │ 分配request时确定     │ 分配request时确定
              ┌─────┴──────┐        ┌─────┴──────┐
              │  plug(cpu0)│        │  plug(cpu1)│
              │  bio merge │        │  bio merge │
              │  无锁独立  │        │  无锁独立  │
              └─────┬──────┘        └─────┬──────┘
              CPU0 fio线程           CPU1 其他线程
              submit bio             submit bio

设计原因：
  - ctx 独立：保证 plug 层 merge 无锁（per-CPU 天然隔离）
  - 共享 hctx0：博通 HBA 的 HQ0 是一个物理队列，CPU0 和 CPU1
    都通过同一个 ring buffer 下发命令
  - 共享 deadline_data：mq-deadline 使用全局视角做 IO 排序，
    才能保证跨 CPU 的相邻 IO 被合并，以及全局的 deadline 公平性
  - dd->lock 是代价：单锁限制了高并发时 CPU0/CPU1 同时 insert 的吞吐，
    但保证了全局调度正确性
```

---

*Linux 6.6, block/blk-mq.c, block/blk-mq-sched.c, block/mq-deadline.c*
