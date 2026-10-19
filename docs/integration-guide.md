# 集成指南：语言运行时、游戏引擎与数据库

本文是基于[C ABI草案](c-abi-spec.md)的调用流程设计，不是可运行示例；当前没有库、头文件、构建结果或性能数据。三类使用者共享独立函数、多arena、自举和明确预算的合同，各自选择适合的模式。

## 1. 所有集成都先固定的事项

1. 用`sa_abi_query`确认版本和目标ABI，用`sa_platform_query`查询创建前的P/G及基础能力；按文档初始化结构头、保留字段及输出buffer。所有status都处理，不把非零结果当空指针的一种普通情况。
2. 为不同生命周期或故障域创建独立arena；创建配置显式选择defaults_version=1，保存求值后的`sa_arena_query`结果用于诊断和基准重放。额度创建时固定，需要改变则在安全生命周期边界重新创建。
3. `sa_arena_create`、`sa_arena_open`、`sa_arena_init_in_place`均准备EMPTY cleanup token。失败时没有可用arena；token非空时由唯一负责人执行`sa_cleanup_retry`，不能丢弃清理责任或复制token并发重试。
4. 高频普通分配线程调用`sa_thread_attach`并仅使用自己的ctx；低频控制线程可用显式arena的无ctx入口。不同arena需要不同ctx，不存在全局默认heap。
5. 在应用队列、任务图或其他同步原语中明确对象所有权移交。远程free不允许把owner的ctx交给释放线程；释放线程使用自己的ctx或普通无ctx入口。
6. 普通free成功后不再访问该对象，即使实际OS回收延后。root、mapping、page lease、RT对象和普通对象使用各自释放接口，不混用。
7. 退出前停止新进入并同步所有调用/访问。teardown一旦把责任移交cleanup token，旧arena不再作为有效调用入口，只能通过token继续清理。

默认16MiB并不意味着16MiB净payload；metadata、缓存、RT状态、日志和隔离范围均占容量。4TiB是64位设计地址上限，不代表设备一定能reserve或commit；Android32遵守独立上限。

## 2. 语言运行时

### 推荐划分

每个相互隔离的runtime实例使用独立VOLATILE arena。普通语言对象使用malloc族；运行时自行管理的栈、JIT辅助区或按页数据使用page/map族。GC压缩、对象追踪、写屏障和语言级引用语义仍由runtime负责，allocator不会自动搬动普通对象。

JIT执行权限只通过平台支持的`sa_mprotect`变更专属页范围，先停止相关执行与访问。不要给含普通malloc对象或allocator metadata的页更改保护，也不要依赖所有OS允许同样的可执行映射策略。

### 调用顺序

1. `sa_arena_create`选择VOLATILE；记录`sa_arena_query`中的页大小、预算和能力状态。
2. worker启动后各自`sa_thread_attach`。使用`sa_malloc_ctx`、`sa_calloc_ctx`、`sa_aligned_alloc_ctx`；扩容时通过`sa_realloc_ctx`接收独立输出指针，失败保留旧对象。
3. 跨worker传递对象先完成应用同步。接收者最终通过自己的`sa_free_ctx`释放；不是把原ctx也传过去。
4. safepoint可让各worker调用`sa_thread_flush`，控制线程再按预算调用`sa_arena_maintain`。不能假设控制线程能强行收走睡眠worker的私有freelist。
5. 请求失败时区分metadata、commit、其他额度和连续范围问题。最多执行应用明确允许的维护/降载策略，不向系统heap隐式补一块，也不无限循环分配直到成功。
6. 普通对象都释放后，各worker`sa_thread_detach`；显式解除所有页映射/lease，再`sa_arena_destroy`。若业务明确整体放弃所有临时对象，quiescent后可用`sa_arena_discard`。

ctx detach允许仍有普通live对象，它们由orphan接管；但arena destroy不因此允许忽略这些对象。runtime仍需保证最终释放或明确整体discard。

### 页范围和错误处理

先通过`sa_map_estimate`获得资源与报告容量上界。为`sa_mmap`、`sa_munmap`、`sa_map_replace`、`sa_mremap`、`sa_mprotect`准备caller report buffer；BUFFER_TOO_SMALL发生在用户目标OS修改前，可增大缓冲后重试。

PARTIAL_FAILURE不是普通无效果失败。只访问报告明确保留的片段，使用`sa_map_query`获取当前状态；遇STALE_CURSOR从头查询或先quiesce，遇BUSY不读取正在变更的范围。旧mapping handle不能授权访问已解除hole中的新owner。

## 3. 游戏引擎与受限帧线程

### 普通资源与RT资源分开

资源加载、解压、关卡切换和文件映射使用普通arena/控制线程；受限帧循环、音频或固定任务池仅使用已prepare的RT pool。两者可属于同一VOLATILE arena并共享统一broker，但RT运行阶段不会向普通heap借资源。

RT的“有界”指allocator检查数量和无慢路径，不是操作系统调度、缺页或缓存一致性的绝对时间承诺。若要求锁页，prepare失败必须处理，不能把预触页成功视为锁页成功。

### Prepare与run

1. 所有未来参与free的线程先`sa_thread_attach`。为每个pool选择一个alloc/maintenance owner，准备固定class_index、slot_size、slot_count和允许participant集合。
2. 控制阶段调用`sa_rt_prepare`，给出全部payload+metadata的total_budget、K及可选REQUIRE_LOCK；输出各participant的binding。只有整个prepare成功后才把binding交给对应线程。
3. owner调用`sa_rt_try_alloc(binding, class_index, …)`；所选类不通过动态任意size搜索。其他登记participant通过自己的binding执行`sa_rt_free`。
4. 合法跨线程free必成功，不存在queue-full回退。free发布后对象立即归还责任，producer不得继续读写；owner可能尚未扫描到它。
5. try_alloc返回WOULD_BLOCK时立即执行应用级策略，例如放弃本次非关键工作、使用应用事先安排的独立预备对象，或退出本次任务。不得偷偷切换普通malloc、等待producer或在一调用内无限maintain。
6. owner按预先分配的帧预算调用`sa_rt_maintain`；每次最多K检查。记录WOULD_BLOCK比例、接收归还数和维护调用数，不把“已free”直接等同“下一次alloc可用”。

一轮`ceil(N/K)`维护步覆盖slot位置。并发release-store不保证某次缺少应用同步的扫描立即看到；如果应用先建立归还完成与后续维护的happens-before，一轮可以接收这些已同步归还。

### 关卡或线程池退出

先停用旧任务和binding，确认所有对象释放，owner执行足够维护接收Returned，再quiescent调用`sa_rt_pool_destroy`。有RTBinding时`sa_thread_detach`会BUSY；不能先销毁ctx留下悬空binding。需要改变owner、类表、参与者或配额时，清空并销毁旧pool后重新prepare，不运行中偷偷改表。

大页、NUMA绑定和padding只能按能力与实测选择。默认RT每slot至少4B状态，16B对象的状态成本已达payload的25%；容量规划必须把这项及索引、对齐计入。

## 4. 数据库与持久数据结构

### 选择模式

- VOLATILE适合查询临时内存或可重新构建的缓存；文件映射上保留字节不等于可重开heap。
- CLEAN_REOPEN适合应用能接受异常退出后无法直接恢复的场景；正常clean close才形成重开保证。
- RECOVERABLE_METADATA恢复allocator已提交分配权属与roots；业务payload事务、读者生命周期及多对象一致性仍由数据库承担。

持久arena的主文件独占管理，一个主文件首版只承载一个durable arena。多个不同主文件可同时打开。外部文件子区间映射只是会话资源，不被自动重开、不作为持久root目标。

### 创建、发布与读取

1. 应用按自己的文件路径/命名空间协议以不覆盖方式创建新的空专用主文件，提供有效native file引用；库取得lease后复核length=0，`sa_arena_create`建立所选持久模式。已有非空文件用`sa_arena_open`，不得用create覆盖或自动修复。路径和目录项持久性由应用负责；库仅保证声明的文件内容屏障。
2. 用`sa_palloc`获得phandle和当前地址，构造payload。对象内部跨重开的引用使用sa_ref逻辑身份或应用自己的经验证offset格式，不能持久化裸指针、vtable或线程锁。
3. 用`sa_persist`把payload推进到应用所需的受支持屏障阶段；成功后再`sa_root_set`发布byte_delta=0的ref。root事务不自动补做遗漏的数据persist。
4. 读取时`sa_root_get`取得ref，再`sa_ref_resolve`验证身份、代次和访问长度。返回指针不pin应用读者；数据库的epoch/锁/事务系统负责与pfree协调。
5. 更新可构造新对象、persist并替换root，再在应用证明旧对象无直接/间接引用和读者后`sa_pfree`。直接root仍引用旧对象时pfree返回BUSY；库不会扫描所有payload替应用证明无间接引用。

首版没有通用持久realloc事务。不能把普通`sa_realloc`用于持久对象；需要按上面的新建、复制、persist、发布、回收流程组织业务更新。

### 未知提交与恢复

COMMIT_UNKNOWN意味着调用结果不能被解释为“失败所以没有生效”。该arena进入WRITE_FAULTED后停止写入与空间复用，数据库先停止业务访问并保存诊断，quiescent后`sa_arena_abort_close`，再`sa_arena_open`按磁盘事实恢复。

- 对已知ref用`sa_allocation_query`判断同代对象是否仍LIVE或已FREE；如果该目录ID已经被其他代次复用，返回INVALID_HANDLE，不能把新对象当旧对象。
- palloc在commit后、返回前崩溃可能没有已知ref；用`sa_allocation_enumerate`辅助重建关联。
- 未出现在allocator root表中的对象只是候选孤立对象，可能被其他payload间接引用；不得批量自动释放。
- 枚举revision变化返回STALE_CURSOR。恢复时在开放并发业务之前完成需要一致视图的枚举；不让分页结果混合不同代次。
- 多文件的persist部分失败不能被解释成跨文件原子提交；外部映射与主文件metadata事务分别处理。

正常close允许live持久对象。应用先按事务协议persist需要的数据，显式解除session mappings、detach全部ctx、停止访问，再`sa_arena_close`。它不能把半成品业务事务变成原子事务。WRITE_FAULTED不走正常clean提交；使用abort-close及恢复路径。

## 5. 集成验收与常见误用

| 验收点 | 应看到的结果 |
| --- | --- |
| 线程携带其他线程ctx | 非法使用；不会因为同arena而合法 |
| 多arena并发与独立故障 | 一个实例的额度/WRITE_FAULTED不污染其他实例 |
| 普通free后OS回收失败 | free逻辑成功，物理资源继续计费；不再次free同对象 |
| RT大量跨线程归还 | 每次合法free成功，分配可暂时WOULD_BLOCK |
| partial unmap后旧handle访问hole | 所有权检查拒绝，不触及hole中的新映射 |
| report容量不足 | BUFFER_TOO_SMALL、required有效、没有用户范围OS修改 |
| clean模式异常退出 | 拒绝假装恢复；由应用备份/离线策略处理 |
| durable提交未知 | 进入WRITE_FAULTED，恢复后查询，不盲目重试pfree |
| teardown清理token非空 | 旧arena入口停用，唯一cleanup责任仍需处理 |

应用性能验收分别使用runtime对象谱、整帧延迟、数据库请求延迟/吞吐及内存峰值，遵守[基准规范](benchmarking.md)。接口名字和流程存在并不代表当前项目已经实现或证明了这些结果。
