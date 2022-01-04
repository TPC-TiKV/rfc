# TPC TiKV

## 团队介绍

- [youjiali1995](https://github.com/youjiali1995): TiDB 分布式事务研发工程师
- [sticnarf](https://github.com/sticnarf): TiDB 分布式事务研发工程师 

## 项目介绍

Better TiKV!!!

##  动机

TiDB 的写入性能和性能稳定性一直被人吐槽，主要原因在 TiKV。作为经常参与性能调优的人，常因为 TiKV 不好的 CPU、disk 资源使用情况感到痛心疾首，以致于不推荐用户给 TiKV 使用 16C 以上的机器，它“不配”。举几个例子：

- 某用户将 16C TiKV 替换成 2 个 8C TiKV，相同并发下性能大幅提升。
- 某用户将 TiKV 机型配置翻倍，即使增加并发吞吐也无任何提升。
- 上述场景，CPU、disk 均未到达瓶颈，且有很多余量。

除了好的机器利用不好外，资源紧张时 TiKV 的表现也不尽如人意：

- 4C/8C/16C 配置下的性能稳定性差距很大。
- 读、写、写的每个 pipeline stage、compaction、backup/restore 等任务互相争抢资源，导致性能不稳定。

除了资源使用问题外，TiKV 的性能调优也是门“技术活”，是只属于少部分 TiDB 研发的游戏，但即使是这些人，性能调优也充满不确定性。TiKV 也不够灵活，无法自动根据压力、workload、机器配置等进行适配，需要为每个用户手动调优，费时费力且无法应对变化。

本项目采用 bottom-up 的设计思路，从更好地利用 CPU、disk 等资源的角度出发，使用 TPC(thread-per-core) 的线程模型来优化 TiKV 的写入性能、性能稳定性和自适应能力。

![bottom-up-design](./media/bottom-up-design.png)

## 详细设计

TiKV 目前使用经典的 [SEDA](https://en.wikipedia.org/wiki/Staged_event-driven_architecture) 线程模型，它的缺点在 FAQ 里会提到，这里只介绍写入路径上最关键组件的问题 —— raft store。raft store 包含两个 thread pool：store pool 用于处理 raft message、append log 等，raft log 会写入 raft db；apply pool 用于处理 committed log，数据会写入 kv db，目前 raft db 和 kv db 均使用 [rocksdb](https://github.com/tikv/rocksdb)，之后 raft db 会切换到 [raft-engine](https://github.com/tikv/raft-engine)。

![tikv](./media/tikv.png)

rocksdb 无法很好的利用现代高速硬盘，因为它的 foreground write(WAL) 只能提供 1 个 I/O depth 且 write group 间同步、排队的消耗很大，而 NMVe SSD 等高速硬盘需要高的 I/O depth 来打满 IOPS，或者大的 I/O size 加上不那么高的 I/O depth 来打满 bandwidth，但大的 I/O size 不适合 OLTP 系统，因为攒大 batch 通常意味着高的延迟。

![rocksdb](./media/rocksdb.png)

为了优化 TiKV 的 disk 使用，raft engine 需要支持并发写 WAL 或者拆分 raft db 来并行写多个 WAL 文件，为了更公平的和 upstream TiKV 做性能对比，本次 hackathon 不会对数据模型做很大改动，会实现并行写 WAL，不会拆分 raft db。为了最大化 disk 的压力、更好的 CPU 使用率、更好的性能稳定性，选择使用 async I/O 来实现该功能。

store pool 实现了上述功能后，它的性能应该会大幅优于 apply pool，但可能会消耗更多的资源从而影响整体的性能，如消耗了更多的 CPU 和 disk I/O 资源导致 apply pool 变慢、积攒太多 committed logs 导致 OOM 等，且整个 pipeline 的性能受限于最慢的一个阶段，需要根据最慢的阶段做 back pressure，如调整 store pool 和 apply pool 的线程数量从而保证速度匹配。但拆分多个线程池实在是不易用、不灵活，为了避免手动调优，我们会将 store pool 和 apply pool 合并为单个线程池，为了实现这一目标，raft engine 使用 async I/O 也是必需的，kv db 同样需要使用 async I/O，但 kv db 理论上可以不写 WAL，因为数据可通过 raft log 回放且该功能已有方案，在 hackathon 上会强行去掉 kv db 的 WAL。除了 async I/O 外，还需要实现 CPU scheduler 来保证当 CPU 成为**瓶颈**时单个线程内不同任务成比例地使用资源，如原来 store pool 和 apply pool 的任务各使用 50% 的 CPU 资源。

有了 CPU scheduler 后可以把更多的线程池合并在一起从而实现真正的 unified thread pool，如 gRPC thread pool、scheduler worker pool、unified read pool、rocksdb background threads、backup thread pool 等，CPU scheduler 会给每个原先 thread pool 的任务分配一定比例的资源，且可动态调整，从而提升资源紧张时的性能稳定性、实现自适应和避免手动调参。

### raft-engine parallel log

目前 [raft-engine](https://github.com/tikv/raft-engine/commit/dce6c225bec148146749780dc4debcffb373990a) 类似 bitcask 的实现：

- 所有 raft group 的 log 都顺序写入当前 log file 中，当 log file 到达一定大小后会切换到新的 log file；在 memtable 中维护了所有 raft group 部分 raft log 所属的文件和文件地址。
- 写 log 的流程类似 rocksdb，会由 write group leader 写入所有 writer 的数据，但目前需要调用多次 `pwrite()` 和一次 `fdatasync()`。
- 需要主动调用 [`Engine::compact_to()`](https://github.com/tikv/raft-engine/blob/dce6c225bec148146749780dc4debcffb373990a/src/engine.rs#L254) 标记清理无用的 raft log、主动调用 [`Engine::purge_expired_files()`](https://github.com/tikv/raft-engine/blob/dce6c225bec148146749780dc4debcffb373990a/src/engine.rs#L208) 清理磁盘上无用的数据。

为了支持 parallel log，我们会将 log file 划分为固定 4KiB 大小的 page，每次写入以 page 为单位，数据格式不发生变化（已有 checksum 来保证数据的正确和完整性）。I/O 方式选择 `O_DIRECT | O_DSYNC` 以支持 async I/O，log file 会预先分配阈值大小来避免 `O_DSYNC` 每次写入都需要修改 metadata。并发的写 log 请求不会组成 write group，每个请求单独写入，会使用 atomic log page allocation（人话是单个原子变量）来分配不重叠的、连续的 page，当分配的 page 超出 log file 大小时，需要切换到新的 log file，使用锁来实现。数据恢复时会以 page 为单位遍历 log file 所有的数据以防止 log file 中有空洞（hackathon 可以不做）。

####  open questions

- `O_DSYNC` 可能需要是 write-through disk，而且只 `fallocate()` 足够吗？[commitlog: Add optional use of O_DSYNC mode](https://github.com/scylladb/scylla/commit/1e37e1d40c78cb3c86ff5ce33d3a58dce5670b1f#diff-58f71059b7c89ad959d4d27d9e921026bb0a2fd5c8d74773423df48139d9c69dR1267-R1269) 可能最好是复用 log file 来避免 `open()`、`fallocate()` 及 sync dir，但需要修改数据格式包含当前 active file number 来区分旧数据。

- 支持 write group logging 来增大 batch？使用 `O_DIRECT | O_DSYNC` 的 I/O 方式，write group logging 在当前 raft-engine 实现下没有任何帮助，不过可以优化为单次 write，但如果支持 async I/O 需要在 glommio 中跨线程 wake。

- [`Engine::fetch_entries_to()`](https://github.com/tikv/raft-engine/blob/dce6c225bec148146749780dc4debcffb373990a/src/engine.rs#L215) 仍使用 buffer I/O，会不会有奇怪的影响？但很容易实现为 direct I/O 且增加 entry cache 大小来缓解。

- 修改 gc 的 I/O 方式，不过 raft log gc 的工作在 TiKV 中由单独的线程完成，本次 hackathon 不需要解决。

### async I/O + async store batch system

raft-engine 依赖 [glommio](https://github.com/DataDog/glommio) 来提供异步接口，store pool 也会使用 glommio runtime。

#### 单个 raft group 的流程

有了 async ready 后理想的流程：

1. batch system 接收到 raft group，处理一批 messages。
2. 获取 raft group ready，发送 messages 给其他 peer、发送 committed entries 给 apply batch system、生成 `WriteTask` 并 `advance_append_async()`。
3. 生成的 `WriteTask` 会在 batch system end 中合并、异步写入。
4. 该 raft group 仍可以处理 messages、生成 ready 并异步写入，不需要等待第 3 步完成。
5. 对异步写入完成的 ready 调用 `on_persist_ready()`，该接口需要升序调用，但不需要顺序递增。

batch system 在接收到 raft group 后 spawn 出处理该 raft group 的 future，在 end 里 spawn 出 `join_all()` 这一批 future 并异步写入 log 的 future，写入完成会对这一批 raft group 调用 `on_persist_ready()`。

由于 batch system 需要先从 `fsm_receiver` 里获取到 raft goup 才会处理它的消息，而第三步异步写入 raft log 可能会将该 raft goup 绑定在当前线程，不会出现在 `fsm_receiver` 里，导致单个 raft group 的流程仍是串行模式。解决方案有：

1. 将 raft group 接收消息也改为 future，当有 ready 未写入完成时 spawn 出接收并处理消息的 future。

2. 保留异步写 log 完成发送 `PeerMsg::Persisted` 的机制，异步写 log 的 future 不包含 fsm 的所有权，就可以扔回 `fsm_receiver` 继续接收、处理后续消息。当 region 收到 `PeerMsg::Persisted` 消息后使用类似 TCP 滑动窗口的方式保存完成的 ready number，没有空洞后使用最后一个 ready number 调用 `on_persist_ready()`。

#### batch system 改造

batch system 会卡在 `fsm_receiver` 上，glommio 需要定期 poll I/O 而且有可能 sleep，需要改造 batch system 来适配。最理想的情况是 glommio 提供 `Poller` trait，batch system 实现该 trait 并由 glommio 定期调用，但目前无这样的机制。可以选择将 batch system 异步化作为 root future，有新的 raft group 要处理时就唤醒 glommio 防止它 sleep。

#### 未解决的同步行为

store batch system 会阻塞的地方：

- store fsm:
  - `StoreTick::SnapGc`：会遍历 snapshot dir，但我们不会生成 snapshot（用 1 或 3 台 TiKV）。
  - `on_raft_message`：会读 kvdb 获取 region state。
  - `StoreMsg::CreatePeer`：会读写 kvdb。
- peer fsm:
  - leader 复制 raft log 可能会读 raft engine，要改的话需要改 raft-rs。（理想情况下不会发生，包括 apply 里获取 committed entries）
  - snapshot 相关的元信息会写 kv db，但我们会把 kv db WAL 去掉。

### CPU scheduler

TODO：https://youjiali1995.github.io/scylladb/cpu-scheduler/

### unified raft store

1. kv db 去掉 WAL 变为纯计算逻辑，不需要 join write group 和 multi batch write。
2. glommio 中创建 2 个 task queue 分别给 store batch system 和 apply batch system 使用，通过 shares 控制 CPU 使用比例。
3. raft group 发送 committed entries 改为 spawn 写入 kv db 的 future 到 apply batch system task queue。（需要 peer fsm 里包含对应的 apply fsm）

### unified thread pool

TODO

### multi-tenant

TODO

## 缺点

- 追求极致的性能需要数据也按线程划分，但单线程热点不好解决，可以退一步到不划分数据，只是线程为 SMP 模型。 
- 工程难度太高，所有组件都要自己实现。
- `io_uring` 对 kernel 版本有要求，但 linux AIO 能达到相同的效果。

## FAQ

### 不改线程模型能缓解上面的问题么？

可以，但不最优，见 [seastar introduction P8-13](https://docs.google.com/presentation/d/1akbSXhToFicZe_h8hKLfXKerZzUfLpYEWawg96_FI7M/edit?n=seastar#slide=id.g106bde28062_0_55)。

### 为什么要使用 async I/O，raftdb 支持同步、并发写 WAL 不够么？

可以，但不最优，见 [seastar introduction P6-7,29](https://docs.google.com/presentation/d/1akbSXhToFicZe_h8hKLfXKerZzUfLpYEWawg96_FI7M/edit?n=seastar#slide=id.g106bde28062_0_43)。

## Task

- [x] raft engine parallel log + async interface @[sticnarf](https://github.com/sticnarf) [[Support async write #1]](https://github.com/TPC-TiKV/raft-engine/pull/1)
- [x] async store batch system @[youjiali1995](https://github.com/youjiali1995) [[async store batch system #1]](https://github.com/TPC-TiKV/tikv/pull/1)
- [x] kvdb remove WAL @[youjiali1995](https://github.com/youjiali1995) [[remove kvdb wal #2]](https://github.com/TPC-TiKV/tikv/pull/2)
- [x] unify store batch system and apply batch system @[youjiali1995](https://github.com/youjiali1995) [[unify store and apply batch system #3]](https://github.com/TPC-TiKV/tikv/pull/3)
- [ ] multi-tenant

## Benchmark

### 环境

- 机器：
  - TiDB/go-ycsb：1 * c5.9xlarge
  - TiKV：1(or 3) * i3.4xlarge(or i3.8xlarge，看 CPU 是否够用)，有时间再测 ebs。
  - PD ：1 * c5.xlarge
- file system：xfs（对 async I/O 支持最好）
- block device 配置：
  - `queue/scheduler`：none
  - `queue/nomerges`：2（disable merge）
  - `queue/write_cache`：确认是否为 write through
  - `queue/nr_requests`：调大
- RAID 0 或者 raftdb、kvdb 分盘部署，需要测试：`io_uring` 目前不支持 poll md，仍通过中断通知。([raid0 vs io_uring](https://lore.kernel.org/all/ee22cbab-950f-cdb0-7ef0-5ea0fe67c628@kernel.dk/t/), [md: add support for REQ_NOWAIT](https://lore.kernel.org/all/20211221200622.29795-1-vverma@digitalocean.com/))
- irqbalance?

### Config

TPC

```yaml
tikv:
    raftstore.store-pool-size: 8 # 它控制合并后的 raftstore 线程数量，可能越大越好，需要 tune
    raftstore.store-io-pool-size: 0 # TPC 实现依赖它为 0 
    raft-engine.enable: true
    rocksdb.disable-wal: true
```

master

```yaml
tikv:
    raftstore.store-pool-size: 4 # 需要 tune
    raftstore.apply-pool-size: 4 # 需要 tune
    raftstore.store-io-pool-size: 1 # async-io，需要 tune
    raft-engine.enable: true
    rocksdb.disable-wal: true
```

### Workload

- rawkv ycsb 纯写：期望效果最好的场景。
- txnkv ycsb
- sysbench

