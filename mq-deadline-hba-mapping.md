# Linux 6.6 mq-deadline 调度队列与博通 HBA 硬件队列对应关系

## 1. 核心数据结构层次

```
内核数据结构层次（Linux 6.6 blk-mq + mq-deadline）
═══════════════════════════════════════════════════

struct blk_mq_tag_set          ← 一块 HBA 卡对应一个 tag_set
├── map[HCTX_MAX_TYPES]        ← CPU→hctx 映射表（DEFAULT/READ/POLL 三张表）
│   └── struct blk_mq_queue_map
│       ├── mq_map[nr_cpu_ids] ← mq_map[cpu_id] = hw_queue_index
│       └── nr_queues          ← 该类型下硬件队列数量
├── nr_hw_queues               ← 硬件队列总数（≤ 系统 CPU 数）
└── tags[nr_hw_queues]         ← 每个硬件队列独立 tag 池

struct request_queue (块设备队列)
├── elevator → struct elevator_queue
│   └── elevator_data → struct deadline_data   ← mq-deadline 唯一实例（全队列共享！）
│       ├── per_prio[DD_PRIO_COUNT=3]
│       │   ├── [DD_RT_PRIO=0]   → struct dd_per_prio
│       │   ├── [DD_BE_PRIO=1]   → struct dd_per_prio
│       │   └── [DD_IDLE_PRIO=2] → struct dd_per_prio
│       ├── lock (spinlock，全局序列化 insert/dispatch)
│       ├── last_dir / batching / starved
│       └── fifo_expire[] / fifo_batch / writes_starved
│
└── queue_hw_ctx[nr_hw_queues] ← 硬件队列指针数组
    ├── [0] → struct blk_mq_hw_ctx (hctx0)
    ├── [1] → struct blk_mq_hw_ctx (hctx1)
    └── ...

struct blk_mq_hw_ctx (每个硬件队列一个)
├── queue_num          ← 本 hctx 的索引
├── cpumask            ← 该 hctx 服务的 CPU 集合
├── ctxs[]             ← 指向所属 CPU 的 blk_mq_ctx 数组
├── nr_ctx             ← ctxs 数组长度
├── tags               ← 驱动 tag（硬件 tag，dispatch 后分配）
├── sched_tags         ← 调度器 tag（有 scheduler 时，alloc 时就分配）
├── sched_data         ← mq-deadline 在此不使用（用全局 deadline_data）
├── dispatch           ← bypass list（调度器无法立即下发的请求临时存放）
└── ctx_map (sbitmap)  ← 标记哪些 ctx 有待处理请求

struct blk_mq_ctx (每个 CPU 一个，软件队列)
├── cpu                ← 所属 CPU id
├── index_hw[HCTX_MAX_TYPES]  ← 该 CPU 对应的 hw queue index
├── hctxs[HCTX_MAX_TYPES]     ← 快速指针缓存
└── rq_lists[HCTX_MAX_TYPES]  ← 无调度器时的请求暂存列表
```

---

## 2. mq-deadline 的 dd_per_prio 结构

```
struct dd_per_prio（每个优先级一份，全队列共享）
═══════════════════════════════════════════════
├── dispatch              ← list_head，优先下发列表（requeue 的请求落这里）
├── sort_list[DD_READ]    ← rb_root，按扇区号排序的读请求红黑树
├── sort_list[DD_WRITE]   ← rb_root，按扇区号排序的写请求红黑树
├── fifo_list[DD_READ]    ← list_head，按到期时间排序的读请求 FIFO
├── fifo_list[DD_WRITE]   ← list_head，按到期时间排序的写请求 FIFO
├── latest_pos[DD_READ]   ← sector_t，最近一次 dispatch 的读扇区位置
└── latest_pos[DD_WRITE]  ← sector_t，最近一次 dispatch 的写扇区位置

每个 struct request 同时挂在两个链/树上：
  rq->rb_node   → sort_list（红黑树节点，用于扇区顺序查找和前向 merge）
  rq->queuelist → fifo_list 或 dispatch（链表节点，用于超时/顺序出队）
  rq->fifo_time → 超时绝对时间 = jiffies + fifo_expire[dir]
```

---

## 3. 博通 HBA 卡 hctx 与 CPU/ctx 的对应关系

### 3.1 典型映射：2 个硬件队列，4 个 CPU

```
博通 HBA 卡（示例：2 个硬件提交队列 HQ0/HQ1，各绑定 2 个 CPU 中断）
═══════════════════════════════════════════════════════════════════════

物理层：
  HBA 卡
  ├── HQ0（硬件提交队列 0）── MSI-X IRQ #0 ── CPU0 亲和
  │                       └── MSI-X IRQ #0 ── CPU1 亲和（共享同一中断向量）
  └── HQ1（硬件提交队列 1）── MSI-X IRQ #1 ── CPU2 亲和
                          └── MSI-X IRQ #1 ── CPU3 亲和

内核层映射（blk_mq_map_hw_queues → irq_get_affinity）：
  tag_set->map[DEFAULT].mq_map[cpu0] = 0  ← CPU0 → hctx0
  tag_set->map[DEFAULT].mq_map[cpu1] = 0  ← CPU1 → hctx0（共享 HQ0）
  tag_set->map[DEFAULT].mq_map[cpu2] = 1  ← CPU2 → hctx1
  tag_set->map[DEFAULT].mq_map[cpu3] = 1  ← CPU3 → hctx1（共享 HQ1）

blk-mq 对象：
  hctx0 (queue_num=0)            hctx1 (queue_num=1)
  ├── cpumask = {CPU0, CPU1}     ├── cpumask = {CPU2, CPU3}
  ├── nr_ctx = 2                 ├── nr_ctx = 2
  └── ctxs[] = [ctx0, ctx1]     └── ctxs[] = [ctx2, ctx3]

  ctx0 (CPU0)  ctx1 (CPU1)  ctx2 (CPU2)  ctx3 (CPU3)
  hctxs[0]=hctx0            hctxs[0]=hctx1
  index_hw[0]=0              index_hw[0]=1

mq-deadline（关键点：只有一个 deadline_data，被所有 hctx 共享！）
  deadline_data（挂在 q->elevator->elevator_data）
  ├── per_prio[RT]  { sort_list[R/W], fifo_list[R/W], dispatch }
  ├── per_prio[BE]  { sort_list[R/W], fifo_list[R/W], dispatch }
  ├── per_prio[IDLE]{ sort_list[R/W], fifo_list[R/W], dispatch }
  └── lock（单把自旋锁，序列化所有 insert 和 dispatch）
```

### 3.2 完整对应关系图

```
                    ┌─────────────────────────────────────────────────┐
                    │              mq-deadline (全局共享)               │
                    │  ┌──────────────────────────────────────────┐   │
                    │  │          deadline_data                    │   │
                    │  │  per_prio[RT_PRIO=0]                     │   │
                    │  │  ├─ sort_list[READ]  (rb_root)           │   │
                    │  │  ├─ sort_list[WRITE] (rb_root)           │   │
                    │  │  ├─ fifo_list[READ]  (list_head)         │   │
                    │  │  ├─ fifo_list[WRITE] (list_head)         │   │
                    │  │  └─ dispatch         (list_head)         │   │
                    │  │  per_prio[BE_PRIO=1]  (同上结构)          │   │
                    │  │  per_prio[IDLE_PRIO=2](同上结构)          │   │
                    │  │  lock (spinlock)                         │   │
                    │  └──────────────────────────────────────────┘   │
                    └─────────────────────────────────────────────────┘
                          ↑ insert           ↑ insert
                          │ dd_insert_request│
              ┌───────────┴──────┐   ┌───────┴──────────┐
              │   hctx0          │   │     hctx1         │
              │ (HBA HQ0)        │   │  (HBA HQ1)        │
              │ cpumask={0,1}    │   │  cpumask={2,3}    │
              │ sched_tags       │   │  sched_tags       │
              │ dispatch(bypass) │   │  dispatch(bypass) │
              └────┬──────┬──────┘   └────┬──────┬───────┘
                   │      │               │      │
              ┌────┴─┐ ┌──┴───┐      ┌───┴─┐ ┌──┴───┐
              │ ctx0 │ │ ctx1 │      │ ctx2│ │ ctx3 │
              │(CPU0)│ │(CPU1)│      │(CPU2│ │(CPU3)│
              └────┬─┘ └──┬───┘      └───┬─┘ └──┬───┘
                   │      │              │       │
              应用进程/内核线程在对应 CPU 上提交 bio
```

---

## 4. IO 提交路径（bio → hctx → deadline_data）

```
用户态/内核 write()/read()
        │
        ▼
bio_submit → blk_mq_submit_bio()
        │
        ├─[plug 机制]─→ 请求被缓冲在 current->plug->mq_list
        │                （plug flush 时批量进入调度器）
        │
        ├─[尝试 bio merge]─→ dd_bio_merge()
        │   spin_lock(&dd->lock)
        │   blk_mq_sched_try_merge()  ← 查 q->last_merge 或 hash
        │   spin_unlock(&dd->lock)
        │   └─ 成功：bio 合入已有 request，不新建 request
        │
        ▼
blk_mq_get_request() → 分配 struct request
        │   ├─ rq->mq_ctx  = blk_mq_get_ctx() = this_cpu->ctx
        │   └─ rq->mq_hctx = ctx->hctxs[type]  （由 mq_map[cpu] 决定）
        │      （有调度器时，从 hctx->sched_tags 分配 tag）
        │
        ▼
blk_mq_insert_request() → dd_insert_requests(hctx, list, flags)
        │   spin_lock(&dd->lock)
        │   dd_insert_request():
        │   ├─ blk_mq_sched_try_insert_merge()  ← 再次尝试 request-level merge
        │   ├─ deadline_add_rq_rb(per_prio, rq) ← 插入红黑树（按扇区）
        │   ├─ elv_rqhash_add(q, rq)            ← 插入 hash（加速尾部 merge 查找）
        │   └─ list_add_tail(fifo_list[dir])    ← 插入 FIFO（按 fifo_time）
        │   spin_unlock(&dd->lock)
        │
        ▼
blk_mq_run_hw_queue(hctx) 触发 dispatch
```

---

## 5. IO 下发路径（dispatch）：关键：调用哪个 hctx，可能下发任意 CPU 的请求

```
blk_mq_run_hw_queue(hctx)
        │
        ▼
__blk_mq_sched_dispatch_requests(hctx)
        │
        ├─[先处理 hctx->dispatch bypass 列表]
        │
        ▼
__blk_mq_do_dispatch_sched(hctx)        ← 有调度器时走这里
        │  循环调用：
        ▼
dd_dispatch_request(hctx)               ← mq-deadline 的 dispatch 回调
        │   spin_lock(&dd->lock)
        │   ① 优先处理 priority aging（低优先级长时间未服务时提升）
        │   ② 按优先级 RT→BE→IDLE 依次查找：
        │      __dd_dispatch_request(dd, per_prio, now)
        │      ├─ 先检查 per_prio->dispatch 列表（requeued 请求）
        │      ├─ 检查 deadline_next_request()（扇区顺序，批量）
        │      └─ 检查 deadline_fifo_request()（超时，到期优先）
        │   spin_unlock(&dd->lock)
        │   返回 struct request *rq
        │
        │  ⚠️  重要：返回的 rq->mq_hctx 可能 ≠ 传入的 hctx！
        │      （因为 deadline_data 是全队列共享的）
        │
        ▼
__blk_mq_dispatch_rq_list() / blk_mq_dispatch_hctx_list()
        │   如果 rq->mq_hctx != hctx，按 mq_hctx 分组后分别 dispatch
        │
        ▼
blk_mq_get_driver_tag(rq)  ← 分配驱动 tag（硬件层 tag）
        │
        ▼
hctx->queue->mq_ops->queue_rq(hctx, &bd)  ← 调用 HBA 驱动
        │
        ▼
博通 HBA 驱动将请求写入硬件提交队列（HQ ring buffer）
doorbell 通知 HBA 硬件
```

---

## 6. 当一个 HBA 硬件队列中断绑定两个 CPU 时的详细分析

### 6.1 场景设置

```
HQ0 ← MSI-X IRQ #0，affinity mask = {CPU0, CPU1}
（博通 HBA 的中断可以 affinity 到多个 CPU，由 /proc/irq/N/smp_affinity 决定）

blk_mq_map_hw_queues() 的逻辑：
  for each cpu:
    if cpu ∈ irq_affinity(HQ0): mq_map[cpu] = index(HQ0)
  
结果：mq_map[0] = 0, mq_map[1] = 0   → CPU0 和 CPU1 都映射到 hctx0
```

### 6.2 IO 提交（两个 CPU 同时提交）

```
CPU0 运行的进程提交 bio：            CPU1 运行的进程提交 bio：
  ctx = per_cpu(blk_mq_ctx, 0)         ctx = per_cpu(blk_mq_ctx, 1)
  ctx->hctxs[DEFAULT] = hctx0          ctx->hctxs[DEFAULT] = hctx0
  rq->mq_ctx  = ctx0                   rq->mq_ctx  = ctx1
  rq->mq_hctx = hctx0                  rq->mq_hctx = hctx0
         │                                    │
         └──────────────┬─────────────────────┘
                        ▼
              两个请求都插入同一个 deadline_data
              （dd->lock 序列化 insert）
              ├─ per_prio[BE].sort_list[READ/WRITE] （红黑树）
              └─ per_prio[BE].fifo_list[READ/WRITE] （FIFO）
```

### 6.3 IO Merge（合并）

mq-deadline 的合并发生在两个层次：

```
【层次 1：bio merge（最早，bio 还没有变成 request）】

dd_bio_merge(q, bio, nr_segs):
  spin_lock(&dd->lock)
  blk_mq_sched_try_merge(q, bio, nr_segs, &free):
    ① 检查 q->last_merge（上一次 insert 的 request）
       bio_end_sector(bio) == blk_rq_pos(last_merge)?  → 尾部合并
    ② elv_rqhash_find(q, bio_end_sector(bio))          → 尾部合并
    ③ elevator->ops.request_merge() → dd_request_merge()
       elv_rb_find(sort_list, bio_end_sector(bio))      → 前向合并
  spin_unlock(&dd->lock)
  └─ 成功：bio 直接合入已有 rq，rq 扇区范围扩展
           不需要新分配 request
  └─ 失败：进入下一步，分配新 request

【层次 2：request merge（insert 时，两个 request 合并为一个）】

dd_insert_request():
  blk_mq_sched_try_insert_merge(q, rq, &free):
    elv_attempt_insert_merge():
      ① 检查 q->last_merge
      ② elv_rqhash_find()
      → 若找到可合并的 existing_rq：
         attempt_merge(q, existing_rq, rq)
         dd_merged_requests(q, existing_rq, rq):
           ① 调整 fifo_time（取较早的超时时间，优先满足 deadline）
           ② deadline_remove_request(q, per_prio, rq)
              ├─ list_del_init(rq->queuelist)  ← 从 fifo_list 删除
              ├─ elv_rb_del(sort_list, rq)     ← 从红黑树删除
              └─ elv_rqhash_del(q, rq)         ← 从 hash 删除
           → rq 被释放，existing_rq 扇区范围扩展
  
  ✓ 合并后只有一个 request 最终进入 sort_list + fifo_list

【前向 merge 后的红黑树重排】

dd_request_merged(q, req, ELEVATOR_FRONT_MERGE):
  elv_rb_del(sort_list, req)   ← 先删
  deadline_add_rq_rb(per_prio, req)  ← 再按新起始扇区重新插入
  （确保红黑树的扇区有序性不被破坏）
```

### 6.4 Dispatch 与中断的关系

```
中断触发（HQ0 完成请求，MSI-X IRQ #0 发生在 CPU0 或 CPU1）：

  irq_handler（CPU0 或 CPU1 处理，取决于 irq affinity 和当前负载）
      │
      ▼
  HBA 驱动 complete_rq():
      blk_mq_complete_request(rq)
      ├─ 若 rq->mq_hctx->cpumask 包含当前 CPU → 本地完成
      └─ 否则 IPI 到 rq->mq_ctx->cpu 完成

  dd_finish_request(rq):
      atomic_inc(&per_prio->stats.completed)
      （对 zoned 设备：解锁 zone，触发 blk_mq_sched_mark_restart_hctx）

  完成后触发新的 dispatch：
      blk_mq_run_hw_queue(hctx0, async=true)
          └─ 在 hctx0->cpumask 中选一个 CPU 的 kworker 运行
             blk_mq_hctx_next_cpu() 轮询选择 CPU0 或 CPU1
```

### 6.5 两个 CPU 共享一个 hctx 时的并发控制

```
并发点                     保护机制
─────────────────────────────────────────────────────
dd_insert_request()        dd->lock（spinlock）
dd_dispatch_request()      dd->lock（spinlock）
dd_bio_merge()             dd->lock（spinlock）
dd_merged_requests()       dd->lock（lockdep_assert_held）
hctx->dispatch list        hctx->lock（spinlock，在 hctx 匿名 struct 中）
sched_tags (tag 分配)      sbitmap_queue（无锁位图，per-CPU 批量分配）
driver tags (tag 分配)     sbitmap_queue（无锁位图）
```

**关键观察**：mq-deadline 使用**单把 dd->lock 序列化整个队列的调度决策**，不区分来自 CPU0 还是 CPU1 的请求。这是 `QUEUE_FLAG_SQ_SCHED`（SQ = "shared queue" scheduling）标志的含义：从全队列视角做调度，而非按 hctx 分片。

---

## 7. 总结：核心对应关系

```
博通 HBA 物理层          blk-mq 层                mq-deadline 层
────────────────────────────────────────────────────────────────────
HBA 卡                   struct blk_mq_tag_set     （一个）
  ├── HQ0 (硬件队列0)    struct blk_mq_hw_ctx[0]   deadline_data ←共享
  │   ├── CPU0 中断      struct blk_mq_ctx (cpu0)  per_prio[0..2]
  │   └── CPU1 中断      struct blk_mq_ctx (cpu1)       ↕ (同一份)
  └── HQ1 (硬件队列1)    struct blk_mq_hw_ctx[1]   deadline_data ←共享
      ├── CPU2 中断      struct blk_mq_ctx (cpu2)  per_prio[0..2]
      └── CPU3 中断      struct blk_mq_ctx (cpu3)       ↕ (同一份)

关系要点：
1. N 个 CPU → M 个 hctx (N ≥ M)，由 IRQ affinity 决定映射
2. 每个 CPU 有独立的 blk_mq_ctx（软件队列），持有自己的锁
3. 多个 CPU 的 ctx 可以映射到同一个 hctx（共享硬件队列）
4. mq-deadline 的 deadline_data 是整个块设备队列唯一一份
   └─ 所有 hctx 的 insert/dispatch 都竞争同一把 dd->lock
5. dispatch 时传入 hctx，但 mq-deadline 可返回任意 CPU 提交的请求
   └─ 上层（blk_mq_dispatch_hctx_list）负责按 mq_hctx 分组再分发
```

---

## 8. IO 合并决策流程图（单一硬件队列绑定两 CPU 场景）

```
CPU0 提交 bio_A (扇区100-107)      CPU1 提交 bio_B (扇区108-115)
        │                                    │
        ▼                                    ▼
dd_bio_merge(q, bio_A):             dd_bio_merge(q, bio_B):
  bio_A → 无法合并                    检查 last_merge / hash
  → 分配 rq_A, rq_A->mq_ctx=ctx0     → 发现 rq_A 尾部 = bio_B 起始！
  → dd_insert_request(hctx0, rq_A)   → bio_B 合入 rq_A（尾部合并）
    ├─ deadline_add_rq_rb(rq_A)       → rq_A 扇区范围: 100-115
    └─ fifo_list_add_tail(rq_A)       → 不再分配新 request！
    → q->last_merge = rq_A            → bio_B 直接完成（合并）

最终 mq-deadline 内部状态：
  per_prio[BE].sort_list[READ]:  rq_A (sector=100, len=16)
  per_prio[BE].fifo_list[READ]:  rq_A (fifo_time=jiffies+HZ/2)

Dispatch 时（hctx0 运行）：
  dd_dispatch_request(hctx0):
    → rq_A 出队，rq_A->mq_hctx = hctx0（匹配）
    → blk_mq_get_driver_tag(rq_A)
    → hba_driver->queue_rq(hctx0, rq_A)
    → 写入 HQ0 ring buffer，doorbell 通知博通 HBA 芯片
    → HBA 把合并后的 16 扇区 IO 作为单个命令提交给磁盘/SSD
```

---

## 9. 与博通 HBA 的 hcx（Host Context）对应

博通（Broadcom）SCSI/FC HBA（如 Emulex LPe 系列 / MegaRAID 系列）的驱动在 Linux 中通过 `blk_mq_tag_set` 注册其硬件队列：

```
博通 HBA 驱动（以 lpfc/megaraid_sas 为例）注册流程：

1. driver->probe():
   tag_set->ops        = &hba_mq_ops
   tag_set->nr_hw_queues = num_online_cpus() 或 num_msix_vectors
   tag_set->queue_depth  = hba_can_queue（硬件 tag 深度）
   tag_set->cmd_size     = sizeof(struct scsi_cmnd) + hba_private_size
   blk_mq_alloc_tag_set(&tag_set)

2. blk_mq_alloc_tag_set() 内部：
   为每个 hw queue 分配 blk_mq_tags
   调用 blk_mq_map_hw_queues()（若驱动注册了 map_queues）
     → pci_irq_get_affinity(pdev, vec) 获取每个 MSI-X 向量的 CPU 亲和掩码
     → 填充 mq_map[cpu] = queue_index

3. 博通 HBA 的对应关系：
   MSI-X 向量 0 → hctx[0] → CPU 集合 {0,1}
   MSI-X 向量 1 → hctx[1] → CPU 集合 {2,3}
   ...

   博通驱动的 hba->io_context[hctx_idx] 或类似结构 ←→ blk_mq_hw_ctx[hctx_idx]
   一一对应，hctx->driver_data 指向 HBA 内部的硬件队列上下文
```

**博通 lpfc 驱动的"hcx"（Host Context / Hardware Context）即对应 `struct blk_mq_hw_ctx`，通过 `hctx->driver_data` 链接到 HBA 私有的环形缓冲区和 DMA 描述符结构。**

---

*基于 Linux kernel v6.6，block/mq-deadline.c、block/blk-mq.h、include/linux/blk-mq.h*
