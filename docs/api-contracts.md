# 公共 API 契约

状态：设计基线。这里定义行为、输入输出及所有权；[C ABI详细规格](c-abi-spec.md)给出候选字段偏移、编号与签名，不创建可编译头文件。发布冻结仍须经过实际ABI/平台验证。

## 1. 共同规则

公开接口采用 C ABI、不跨边界抛异常，使用固定宽度整数、opaque handles、调用者提供的输出结构。所有可能失败的调用返回 `sa_status`，结果通过输出参数交付，不使用隐式全局 `errno`、全局默认 heap 或隐式系统堆回退。

配置与查询结构以 `struct_size`、`abi_version` 开头。输入中未知的非零 flags 和不支持的版本明确拒绝；可省略的尾部字段按文档默认值处理；输出只写调用者声明的范围。公开接口不暴露 C++ 类型或布局，不使用 C++ `bool`/enum 大小作为二进制契约。

- 配置容量、文件偏移和持久偏移使用 uint64 表达，运行时对象长度使用目标 ABI 的 size_t 表达；转换和所有加法、乘法、round-up 必须先检查。
- 返回指针仅在所属实例当前生命周期有效；跨进程或重开时使用持久对象句柄转换到指针。
- 实例、映射和线程身份带生命周期标识。有效句柄的跨实例误用可诊断；已经销毁的裸 handle/指针不保证可安全解引用或检测。
- 默认失败不发布新对象；输出指针为 NULL、输出 handle 为无效值。`realloc` 保留旧对象。映射部分失败和持久结果未知有下文专门契约。
- 首版不提供用户callback，不启动后台线程；错误通过status、固定输出结构和显式查询处理。

所有“失败可重试”均以本次结果已判定、原输入身份仍有效为前提；operation_id是诊断编号，不是请求去重键。PARTIAL_FAILURE必须先处理完整报告，COMMIT_UNKNOWN必须经恢复确认，不能把同一map、pfree或root_set原样盲重放。API不承诺异步线程取消、signal handler重入或强制终止持锁线程后仍可恢复执行；应用超时也不撤销一个尚在运行的同步调用。

## 2. 实例与配置

| 接口族 | 必要输入 | 成功结果与边界 |
| --- | --- | --- |
| `sa_arena_create` | 容量、模式、配额、匿名或主文件配置 | 库持有一个 reservation；控制块位于区内 |
| `sa_arena_init_in_place` | P 对齐 base、P 整数倍长度、明确借用权 | VOLATILE 固定 backing；只做逻辑分配，不能自行 protect/unmap/remap |
| `sa_arena_open` | 已有专用主文件、打开策略、配额 | 独占文件 lease 后校验/恢复；不得自动格式化、截断或静默扩容 |
| `sa_arena_destroy` | 已静止的临时实例 | 无活对象、映射、RT 池、ctx 和进行中操作才成功，否则 BUSY |
| `sa_arena_discard` | 已静止的临时实例、显式整体丢弃意图 | 使全部对象失效；释放 owned reservation，借用 buffer 不解除映射 |
| `sa_arena_close` | 已静止的持久实例 | 允许活持久对象和 roots；完成必要 checkpoint 并释放运行时资源/owner lease |
| `sa_arena_query` | 调用者输出结构 | 能力、页大小、容量、模式、预算、健康状态 |
| `sa_platform_query` | 调用者输出结构 | 创建前查询平台P/G等基础能力；实例特定能力创建后再query |
| `sa_cleanup_retry` | 非空caller-owned cleanup token | 推进明确可重试的残留资源清理，不作为可分配arena |

创建容量要求页对齐，范围至少 16MiB；64 位至多 4TiB，32 位至多 `floor(PTRDIFF_MAX/P)*P`。精确请求无法保留时返回 NO_ADDRESS_SPACE，不自动创建更小实例。更大页、granularity 和 backing 对齐由能力查询与预估返回；不把 P 写死为 4096。

首版模式、容量、backing与资源额度在create/open求值后不变；[配置规格](configuration.md)给出版本化默认公式与覆盖项。每次维护调用的工作预算可调整，池K在prepare确定；首版不提供set_limits或运行中的模式转换。持久几何不能在open时被覆盖修改。

持久create接收调用方以不覆盖方式创建并授权初始化的新空file_ref，取得lease后复核长度为0；任何非空文件均不得格式化/截断。文件内容屏障由库执行，路径创建及目录namespace持久性由调用方负责；首版没有path API或库级namespace保证。

默认不锁页、不启动后台线程、不自动启用大页。RT 准备可显式要求锁页，要求时失败不得静默降级。

### 生命周期

调用方必须先阻止新进入、等候已有调用和访问退出，再 detach/销毁。区内引用计数不能使已释放的 arena handle 继续安全。`BUSY` 校验是诊断及状态保持契约，不替代调用方同步。

正常 close 前显式解除 session mappings、销毁 RT/临时资源、detach 全部 ctx；持久 live allocation 不阻止 close。文件擦除/重置不复用 close，首版通过离线管理流程执行。遇到 WRITE_FAULTED 时不把普通 close 当作 clean commit；使用显式 abort-close 释放运行时并要求下次恢复。

create/open/init及destroy/discard/close/abort-close接收初始化为空的固定cleanup输出。初始化失败但资源未完整释放时不返回可分配handle，而通过PARTIAL_FAILURE及token交付清理责任；销毁进入不可逆清理后也可移交token，旧arena不再可调用。只有明确仍由库持有、可安全重试的资源允许cleanup_retry；未知FD关闭结果不能盲重试。完整合同见[C ABI](c-abi-spec.md)与[操作协议](operation-protocols.md)。

## 3. 线程上下文与维护

| 接口族 | 行为 |
| --- | --- |
| `sa_thread_attach` | 为当前线程从指定实例元数据创建 ctx；额度不足失败 |
| `sa_thread_detach` | 阻止该 ctx 再进入，移交 heap/slabs，注销 ctx；不要求已分配普通对象全部释放 |
| `sa_thread_flush` | 按调用者预算归还缓存和接收远程释放，报告实际工作量及剩余提示 |
| `sa_arena_maintain` | 在最大对象数/页数/步骤数总预算内处理 shared heap、orphan、合并、回收等；不无限等待睡眠线程、不强抢私有heap |

ctx 只能由登记线程使用，绑定一个实例 incarnation；远程释放线程必须使用自己的 ctx 或普通无 ctx 入口，不能共享 owner 的 ctx。允许同一线程登记多个实例。普通无 ctx 接口使用预建共享 heap 的慢路径，可能取锁；不隐式注册线程。RT 必须使用预先登记的上下文。

owner token 在一次实例生命周期内不复用；即使并发 ctx 数未满，累计登记次数耗尽 token 空间也返回 LIMIT，不能回绕身份。

ctx仍有任何RTBinding时detach返回BUSY；先在quiescence销毁相应pool并撤销全部bindings。

默认远程接收预算为每次 64 个对象，是可配置设计基线。线程数量、metadata、heap 私有 span 容量、空 span 保留、细粒度 views 都有单独额度，不能用一个“缓存限制”替代所有限制。

## 4. 普通对象分配

接口族包括 `sa_malloc`、`sa_calloc`、`sa_realloc`、`sa_free`、`sa_aligned_alloc`、`sa_usable_size`，以及等价的 `*_ctx` 入口。普通入口显式接收 arena；ctx 入口通过 ctx 唯一定位 arena，避免全局查找。

| 情况 | 约定 |
| --- | --- |
| 默认分配 | 返回至少 16B 对齐、至少请求字节数的块；不自动清零 |
| 零长度 malloc/calloc | OK + NULL；calloc 任一因子为零时不分配 |
| calloc 溢出 | OVERFLOW，任何内存都不分配 |
| free(NULL) | OK，无副作用 |
| realloc(NULL,n) | 等价分配 n 字节 |
| realloc(p,0) | 等价释放 p，成功后输出 NULL |
| realloc 成功 | 至少保留 `min(old_requested,new_requested)` 字节，可换地址；旧块失效 |
| realloc 失败 | 输出 NULL，旧块保持有效，原内容和大小不变 |
| 扩展对齐 | alignment 为 2 的幂且 >=16；size 不要求是 alignment 的倍数；对齐与范围溢出明确失败 |
| usable_size | 返回该有效对象可用字节数，不能用于推断相邻对象或磁盘布局 |

合法的 VOLATILE 对象 free 不因元数据不足、远程队列容量或维护预算不足而失败；OS 回收可延后处理。对象逻辑释放成功不等于 RSS 或 committed 已即时下降。检测到后端回收错误时保留不可复用 backing 的隔离记录，不撤销已经完成的对象释放。

double free、对象内部指针、非本实例对象、free/mmap 交叉误用及仍被其他线程访问的对象均违反契约。诊断模式尽可能检测并返回错误；生产快路径不承诺对任意恶意地址安全探测。对象所有权跨线程转交需要应用建立 happens-before。

持久模式不通过本组隐式获得持久语义：持久对象使用第 7 节独立接口，误用普通对象入口返回 UNSUPPORTED。CLEAN_REOPEN 的持久入口表示逻辑更新，只有成功 clean close 才形成重开保证；RECOVERABLE_METADATA 每次成功表示该元数据事务持久提交。

## 5. 页租用与映射

### 映射描述

`sa_mmap` 接收 arena、长度、placement、backing、保护、共享类型和 split policy，返回 mapping handle、base 和实际长度。长度必须为正且为 P 的整数倍；文件 offset 和固定位置 offset 也必须 P 对齐。接口明确拒绝非对齐长度，不隐式扩大用户可见范围。

placement 为任意位置或固定 arena-relative offset。固定位置默认 no-replace；显式替换只允许指定有效 mapping handle 当前仍持有的范围。没有可任意覆盖对象的裸 MAP_FIXED。

backing 为匿名或调用方可访问的原生文件 handle/FD。库在成功创建文件映射前取得自己的 handle 引用，调用方成功返回后可关闭原引用；库最后使用结束后关闭自己的引用。失败时不关闭调用方原 handle。文件不得被外部并发截断，访问期间的介质错误/外部截断异常不可能全部转换为分配器状态码。

匿名页是会话资源，重建匿名映射应遵循零页语义；借用 buffer 中的 `sa_page_alloc` 仅提供页对齐 lease，默认不清零，显式 ZERO 标志才要求清零。页租用与 OS 文件映射能力必须区分。

| 接口族 | 语义 |
| --- | --- |
| `sa_page_alloc` / `sa_page_free` | 逻辑页区间租用/归还，适用于借用 buffer；不能用对象 free 释放 |
| `sa_map_estimate` | 计算地址、metadata、views 和 backing 资源需求；估算不预留资源 |
| `sa_mmap` | 创建匿名或外部文件映射；返回完整句柄 |
| `sa_munmap` | 解除句柄所有或允许的子范围；地址范围仍归 arena |
| `sa_map_replace` | 仅替换调用者句柄仍拥有的明确范围，新 backing 按能力检查 |
| `sa_mremap` | 范围缩放；默认仅原址；显式 MAY_MOVE 时可返回新 base，成功后旧范围失效 |
| `sa_mprotect` | 按能力变更保护，仅针对映射/页 lease，不接受普通对象地址 |
| `sa_map_discard` | 显式允许丢弃内容的页回收；不等于 release 地址，也不保证即时回收 commit |
| `sa_map_persist` | 刷新 SHARED 文件映射范围；PRIVATE/COW 写回原文件请求返回 UNSUPPORTED |
| `sa_map_query` | 查询存活片段、holes、backing、保护和真实资源状态 |

为确保明确的数据保持语义，`sa_mremap` 首版只处理无洞的完整映射。匿名 MAY_MOVE 必须保留原范围有效内容；文件 SHARED 可重新建立相同文件偏移视图；PRIVATE/COW 只有后端能保留全部私有修改时支持，否则预检查拒绝。缩小丢弃尾部，增长不能越过文件有效长度；不自动修改外部文件大小。映射 resize 和 move 要求整个目标范围无并发访问。

### 普通和可拆映射

普通模式以实际 backend unit 为解除/替换边界；可拆模式初次创建就保证 P 粒度的解除/替换。Windows 使用多个 views 实现可拆模式；原有大 view 不支持无扰动升级。

默认细粒度 view 上限为每映射 4096、每实例 16384，均为项目可调配额，不是 OS 上限；普通 views 单独统计、仍受 metadata/总资源预算限制。超限返回 LIMIT，不静默降级。可拆匿名 backing 回收的实际 commit 状态按平台文档单独报告。

部分 unmap 后 survivors 保留原 mapping 身份与 generation，holes 回归 broker 空闲；后续洞中建立的映射有新身份。原 handle 的操作必须逐片验证所有权，不因原始大范围覆盖洞中新 owner。最后片段消失时 handle 失效。

### 失败和同步

多页调用不是单个原子 OS 动作。调用方停止访问正在改变的范围；邻居范围不得被临时解除。操作先检查和预留全部管理资源，再执行 OS 调用。

首版同一mapping handle同时只允许一个改变其fragment目录、保护、backing或长度的事务，即使两个请求的子范围不相交；冲突操作返回BUSY。MappingDesc变更权与范围Pending共同保护最终发布所需的目录几何，pin单独只保证存储生命周期。不同mapping仍可在各自范围并发，调用方继续同步被访问或改变的payload。

map_persist在受保护的存活身份上取得覆盖其I/O的backend/descriptor pin；与该pin冲突的replace/unmap等变更在最终校验时返回BUSY，persist也不能绕过已取得的mapping变更权。普通query在broker内复制值后返回，不隐式为调用方长期固定映射。

创建失败且完整回滚时不返回可用 mapping。对现有映射的部分变更失败：只有能够证明原 backing/保护/内容契约全部恢复，才报告普通失败并保留原契约；否则返回 PARTIAL_FAILURE，结果结构记录已完成、保持原状和隔离的片段，调用方必须查询并处理。不得只恢复地址却谎称匿名内容也已恢复。

后端范围状态不确定时进入 Quarantined，直到显式维护证明可恢复或实例整体丢弃；不会自动作为空闲空间重用。查询/错误输出由调用者提供存储，容量不足时返回所需条目数，不借用外部堆拼接结果。

所有可能产生多片段变更的正式操作在Pending及目标范围OS修改之前检查caller report容量，按步骤计划预留最坏片段数；不足返回BUFFER_TOO_SMALL及required_count。因此PARTIAL_FAILURE总能输出完整本次报告。估算不预留；query分页绑定ledger revision，变化返回STALE_CURSOR，不能拼接不同代次快照。

并发导致预检计划在最终校验时失配，普通慢路径最多内部重新计划一次；再次失配返回BUSY且尚未改变用户目标范围，调用方可以稍后重新申请。此限制不把普通CAS、锁等待或OS操作变成有界延迟，也不允许回滚已发布free或已提交持久事务。

分页在存在Pending时保守返回BUSY；事务begin与最终publish/rollback分别推进revision。诊断revision达到饱和哨兵后分页统一STALE_CURSOR，合法释放/已有清理继续且不回绕；持久身份/排序计数仍按自身上限拒绝新变更，详见ABI与操作协议。

## 6. 受限实时 API

`sa_rt_prepare`仅适用于库自行保留且具有完整VM能力的VOLATILE匿名实例。输入固定pool owner、slot类别与数量、登记线程集合、维护预算K和可选锁页要求；输出预建pool、容量及开销。首版无通用provider导入，不在借用buffer、持久模式或外部文件映射上提供RT。

`sa_rt_try_alloc`、`sa_rt_free`、`sa_rt_maintain` 不调用 OS、不扩容、不建立 ctx、不执行不受限搜索。分配只由 pool owner 执行，free 可由任意已登记线程执行。页块 RT lease 也来自预建固定槽，不执行真正 mmap。

- `try_alloc` 找到块返回 OK，否则 WOULD_BLOCK，不通过有限扫描断言整个 pool 全满，也不向普通堆回退。
- 合法 free 返回 OK，唯一释放者进行归还状态 release-store；不因资源不足失败。
- 维护每次最多检查 K 个 slot，默认 K=64；报告本次检查和回收数量，不扫描全池生成精确剩余量。
- prepare为允许的ctx/pool关系建立固定RT binding；run入口携带或使用预先选定的binding，校验固定身份，不遍历动态线程列表。binding与pool一起在quiescence后回收。
- N 个 slot 的完整一轮需要 `ceil(N/K)` 个维护步；对象在其位置刚扫描后归还时，下一完整轮会再次检查该位置。若归还通过应用同步 happens-before 这一轮，则一轮内接收；一般并发的 release/acquire 不给出缓存传播的墙钟或指定轮数保证。该工作量保证以实际维护调用发生为条件。
- pool销毁必须quiescent且无活slot/未接收归还；可先维护接收归还。首版不导出活pool owner迁移函数，改变owner须清空、销毁后重新prepare。

详见 [实时设计](realtime.md)。

## 7. 持久 API

持久对象句柄包含 arena UUID、分配 ID、offset 和必要的尺寸信息；主文件格式中的引用使用 offset64。调用者可以解析为本次打开期间的指针，但不能把该裸指针作为重开的定位依据。

接口族为 `sa_palloc`、`sa_pfree`、`sa_persist`、`sa_root_get/set`、`sa_allocation_enumerate`、`sa_checkpoint`。首版不提供通用持久 realloc 事务；应用通过新分配、复制、persist、发布 root 和解除旧引用组合完成。

- root 表有创建时声明的最大条目预算，以整数 root ID 定位；空 root 使用明确无效句柄。
- root_set 只接受本主文件的持久对象，不接受 session map、RT slot 或外部文件地址；root 更新的持久性遵循实例存储模式。
- 对象数据 persist 成功不自动提交 root 或应用多对象事务。
- pfree 前应用必须先解除所有持久引用；不能在提交前把对象 payload 改写为 freelist/remote next。
- 仍被 allocator root 表直接引用的对象执行 pfree 返回 BUSY；库不遍历应用对象图，间接引用和读者生命周期仍由应用负责。
- RECOVERABLE_METADATA palloc/pfree/root_set 返回 OK 表示对应提交完成；不能把普通远程入队视为持久释放成功。
- 提交已开始但结果无法判定时返回 COMMIT_UNKNOWN，实例进入 WRITE_FAULTED，停止新的分配/释放/root 写入和地址复用；保留诊断读取、abort-close 和后续恢复路径。
- 崩溃后可存在未发布到 root 的已分配对象；枚举只是辅助应用恢复，不能把所有非 root 对象自动删除。
- 外部文件映射首版仅会话有效，不恢复 native handles；主文件恢复后其范围回到可用地址集合。

[持久化设计](persistence.md) 定义日志、屏障、checkpoint 和恢复顺序。日志或 checkpoint 失败不能靠切换为 VOLATILE 继续操作。

### 并发故障与已经进入的调用

主文件data persist也进入持久admission并固定目标资源，避免close/checkpoint排空漏掉锁外I/O；pfree最终校验遇有效persist pin返回BUSY，应用的业务引用生命周期仍由应用管理。阶段许可和故障锁存在短控制锁下排序，故障后不签发新写入阶段；此前已获许可的单次动作属于在途，可能晚于故障发生才结束，按真实效果收口。

| 本次调用的提交事实 | 返回与实例行为 |
| --- | --- |
| 尚无commit动作可生效，后续许可被故障拒绝 | WRITE_FAULTED或本调用已知的I/O错误；不产生新的持久分配事实 |
| commit可能生效且无法确认 | COMMIT_UNKNOWN；禁止应用根据失败返回直接重试或释放未知对象 |
| commit已确认且runtime发布完成，另一个调用同时令实例故障 | 本调用仍返回OK，error/query中的health可为WRITE_FAULTED；成功事实不撤销，实例仍禁止后续修改和地址复用 |

data persist的OK仅承诺本次要求阶段已确认完成，不替其他请求或业务事务背书。诊断输出的health反映采样时刻，不承诺返回后不会立刻改变；正常READY也不等于不存在局部QUARANTINED范围。

checkpoint第一份新super已确认、第二份镜像确认未生效的失败，保留内部CHECKPOINT_HELD及关闭的admission，health可以仍为READY；新持久修改返回BUSY。只有显式checkpoint或正常close按已保留阶段继续完成镜像，或者abort-close退出；期间不复用旧WAL/CP。未知发布结果仍进入WRITE_FAULTED，不伪装成可继续写入的已知失败。CHECKPOINT_HELD是内部流程状态，不增加公开health编号。

## 8. 状态码与诊断

| 状态类别 | 建议状态码 | 含义 |
| --- | --- | --- |
| 参数/兼容 | INVALID_ARGUMENT、OVERFLOW、UNSUPPORTED、INCOMPATIBLE_FORMAT、INCOMPATIBLE_GEOMETRY | 输入无效、无法表示或契约不适配 |
| 身份/所有权 | WRONG_ARENA、INVALID_HANDLE、COLLISION | 有效输入检查发现实例、句柄或区间冲突 |
| 生命周期 | BUSY、CLOSED、WRITE_FAULTED | 有效实例当前不能执行操作；不承诺访问已销毁地址可返回 CLOSED |
| 资源 | NO_ADDRESS_SPACE、OUT_OF_MEMORY、NO_CONTIGUOUS_RANGE、METADATA_LIMIT、COMMIT_LIMIT、LIMIT | 区分虚拟地址、总空闲、连续区间、元数据、后端预算和其他额度 |
| 实时 | WOULD_BLOCK | 本次有限工作未取得块，不暗示整个 pool 永久耗尽 |
| 后端 | OS_ERROR、IO_ERROR、PARTIAL_FAILURE、COMMIT_UNKNOWN | 确定失败、范围部分改变或持久结果未知 |
| 状态损坏 | CORRUPT、QUARANTINED | 检测出损坏或不可复用范围 |
| 查询/ABI | BUFFER_TOO_SMALL、STALE_CURSOR、NOT_FOUND、INCOMPATIBLE_ABI | 缓冲容量、分页失效、空目录项或版本不适配 |

错误细节使用调用者提供结构：operation ID、native error、相关offset/length、阶段、健康状态及阻塞资源键/额度。默认不格式化字符串、不日志I/O；诊断由上层处理。候选数值在[C ABI](c-abi-spec.md)列明，未经实现验收不称已发布枚举。

## 9. 统计与可控性

查询区分：reserved 地址、broker free、最大连续 free、用户 requested/live、slot 内部碎片、私有 span capacity、cached free、metadata、日志、RT 状态数组、mapped views、backing/commit charge、resident sample 和 quarantined。

普通小对象默认不保存逐slot requested size：精确requested live及由其推导的精确内部碎片标注不可用，不能用usable bytes或累计请求字节冒充。可选精确追踪使用已计费的区内侧表；大对象和持久记录保留各自精确长度。realloc在未追踪小对象请求长度时可以复制min(old_usable,new_requested)字节，仍满足有效数据保留合同；复制与sanitizer可读范围必须一致。字段有效性通过快照validity标识报告。

并发快照标注采样一致性；精确全局对象统计需要 quiescent 或显式协调，不默认在每次分配添加全局原子。resident 是 OS 采样，不与逻辑 committed 混称。锁页、NUMA、大页、物理预分配和强持久屏障均由能力查询报告，未支持时显式失败。
