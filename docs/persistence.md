# 映射存储与持久化

本文描述文件映射上的分配、正常重开和分配器元数据崩溃恢复。SuperAlloc 不是数据库事务引擎；应用数据一致性由应用协议负责。具体 API 使用统一状态码与输出参数，签名以 [API 契约](api-contracts.md) 为准。本文中的操作名是行为名，不冻结二进制布局或符号拼写。

## 1. 三种模式

| 模式 | 分配器保证 | 成本与限制 |
|---|---|---|
| `VOLATILE` | 本次运行期间的分配正确性；可在匿名内存或映射 buffer 上工作 | 无日志税；不恢复分配账本，文件里仍有字节不代表能重开成有效 heap |
| `CLEAN_REOPEN` | 完整 clean close 后保留分配权属和持久根，允许换基址重开 | 关闭需停用新操作、收敛线程状态并 checkpoint；非正常退出后拒绝按 clean store 直接打开 |
| `RECOVERABLE_METADATA` | 崩溃后恢复已提交的分配器元数据；显式 data persist 由调用方使用 | alloc/free/root 修改有日志和屏障成本；不自动为用户对象事务化 |

三种模式是创建时选择的契约，不在运行中静默切换。普通 volatile 路径不经过日志或持久化屏障。受限实时 profile 仅用于预备的匿名 volatile 池；文件映射、checkpoint、persist 和可恢复元数据提交在非实时阶段执行。

应用数据、allocator 元数据、运行时控制状态必须分开：

- 持久 allocator 状态：格式版本、arena 身份、容量、页/extent 几何、分配目录、对象代次、持久根、checkpoint 与日志位置。
- 用户数据：对象 payload；写入与发布次序由应用管理。
- 运行时状态：锁、线程记录、TLS token、remote free 信息、地址索引、OS handles、file views、等待者及统计缓存。它们位于 arena 的运行时控制区域，重开时重新初始化；不能从文件中的旧 bit pattern 恢复可用锁或指针。

## 2. 存储与进程所有权

首版每个 durable arena 使用一个独立主文件，持久 allocator 元数据与其管理的持久对象都属于该主文件。首版不提供同一主文件分区承载多个 durable arena，也不提供跨进程并发分配器。

同一进程可以打开多个独立 arena。对于每个 durable 主文件：

1. 依据系统提供的稳定文件身份，而非路径字符串，识别文件。硬链接、不同路径或重复 handle 不产生新的 store 身份。
2. 进程内登记表防止本进程重复打开同一主文件。登记节点嵌在各 arena 的运行时控制区，全局只保留常量大小的 root/lock；不能为登记表调用外部 heap。
3. 在验证、恢复或写入之前取得 OS 支持的非阻塞独占 owner lease。冲突返回 `BUSY`，持有至关闭完成。
4. POSIX 后端优先验证 `flock` 的 open-file-description 生命周期；Windows 验证 `LockFileEx`。不使用会因同进程关闭其他 FD 而意外释放的传统 POSIX record lock 作为未经说明的替代。
5. 进程退出后由 OS 释放 lease。header 中的 PID、启动标识只供诊断，不能据此判断 owner 已死或自动绕过锁。

这些锁防止遵守协议的第二个 allocator owner，不阻止任意外部程序直接改写文件。首版只承诺通过验收的本地文件系统；NFS、SMB、锁或 flush 合同不明的文件系统返回不支持，不能静默降级。

普通外部 `MAP_SHARED` payload 映射不受上述主文件独占策略扩大限制；多个 arena 可以按应用协议映射同一外部数据文件。共享数据不等于共享 allocator 的锁、账本或恢复协议。

## 3. 文件格式与地址无关性

字段、偏移、CRC、WAL framing与区域大小已在 [v1候选格式](persistent-format.md) 完成设计。它是未经平台spike和故障模型验证的候选，尚未发布冻结；本文保留行为与恢复责任，不另建第二套字节布局。

持久 superblock 至少描述：

- magic、格式 major/minor、必要 feature bits、字节序/整数编码、arena UUID。
- 主文件有效长度、arena 逻辑容量、系统页/持久页几何、extent/size-class 几何版本。
- 元数据控制区、运行时保留区、payload、checkpoint 和日志的非重叠范围。
- checkpoint generation、已纳入 checkpoint 的 LSN、日志有效边界、根目录位置和校验值。
- clean/dirty 状态及其代次；不能只用一个无校验字节决定是否安全重开。

所有持久位置用经范围检查的 64 位 offset 或带 arena UUID/代次的 handle，不能存运行时裸指针。分配记录需要身份/代次、起点、长度或 size class、状态；外部引用使用 handle 时必须检测已释放后重用造成的旧引用。

runtime control islands 与后续 runtime metadata shards 使用 arena 内独立匿名 backing，不使用主文件 payload view 中的旧锁/指针字节。持久几何为这类工作区保留地址范围，对应的主文件逻辑 offset 不可再分给持久对象，文件中的旧字节不参与恢复。持久 superblock、日志、checkpoint 与 payload 则按独立 backend extent 映射主文件；不得把整个主文件作为一个不能拆分的大 view 从而阻塞所有 runtime islands 和外部映射。

整个runtime区R从创建起属于METADATA，内部用RESERVED/ACTIVE/PENDING/QUARANTINED表示尚未启用、已启用、事务中和隔离，不进入用户FREE。1002元数据地址额度在创建时used=R；R内启用新单元不重复收费，实际backing与records另计。R按需提交，不因4TiB实例的R默认256GiB就预触整段。精确runtime镜像/辅助/scratch布局与open允许覆盖的配置见[v1候选格式](persistent-format.md)。

创建时明确 runtime 工作区上限与重建预算，文件中记录其保留范围。打开时可以在这些范围内重新布置 runtime 结构，但不能为增加工作区覆盖已有持久对象。重建前根据持久记录数量、geometry 和配置核算 descriptor/radix 等需求；不足则在发布 handle 前返回 `METADATA_LIMIT`。checkpoint 备用空间和 runtime reconstruction 空间分别计入容量，降低运行时配置后不保证仍能打开原 store。

generation 回绕不能默默重用：达到格式允许的极限时拒绝产生有歧义的新身份，要求离线迁移。日志与校验负责检测 torn write，不假设 sector、页或 64 位写天然原子。

`open` 必须验证格式、范围、文件长度、容量和页几何。4 KiB 独立边界可能无法直接映射到 16 KiB 主机；首版遇到不兼容返回 `INCOMPATIBLE_GEOMETRY`，不自动重新排布。32 位进程不能整体打开超出其寻址上限的 store。

offset64 也不意味着用户 payload 跨 OS、CPU ABI、编译器或 C++ 对象布局可移植。对象内裸指针、vtable、句柄和用户锁由应用处理；数据库式 offset 数据结构可在满足自身格式协议时使用。

公共API只有file_ref/FD，没有path入口。调用方负责以不覆盖语义创建新空文件及其路径/目录项持久性；库建立匿名bootstrap后取得自己的引用、身份登记和lease，再确认文件长度为0。任何非空文件都拒绝格式化，不截断既有有效或损坏数据。库随后建立长度、初始checkpoint并同步，再按双super发布协议提交有效描述。失败半成品保留明确状态及cleanup责任，不自动删除或截断调用方文件。库只提供文件内容屏障；请求库保证namespace durability返回UNSUPPORTED，不能用内容flush冒充路径持久化。

open为严格区内自举可先借用调用方FD有界读取两个候选super的固定头到栈，取得通过数值上界检查的候选容量；此读不信任身份/CRC、不写文件、不取得库拥有资源。建立bootstrap后才dup、登记、lease并完整重读验证，早期候选变化则失败。精确顺序与读取上界见候选格式，调用方原FD在整个调用期间保持有效。

## 4. Volatile 与 clean reopen 的流程

### 4.1 Volatile 文件 buffer

在映射 buffer 上运行 volatile allocator 只说明它把该内存当工作空间。关闭后保留下来的 payload 不构成有效持久 heap；再次初始化会创建新的分配状态。普通 free 不得因临时元数据不足失败，其所需记录必须在分配或 span 建立时预备。

### 4.2 Clean reopen

创建或正常打开一个 clean store 后，在允许任何修改之前将新一代 dirty 标记按配置的同步屏障提交。该步骤失败则不能进入可写运行态。

正常关闭前，调用方显式解除所有 session mappings、清理临时资源、停止新使用并按 API 完成线程 detach。close 不隐式替调用者解除仍存活的 session mapping；预条件不满足返回 `BUSY`。完成这些预条件后按以下顺序执行：

1. 阻止新操作并等待已进入操作结束，包括主文件 data persist、gate 等待者和 commit 已确认而 runtime 尚未发布的调用，要求登记线程退出或完成 quiesce。
2. 收敛缓存、remote free 和所有运行时分配状态，使每个存活对象在逻辑账本里恰好出现一次。
3. 确认应用在关闭 admission 前已经完成要求的数据 persist；close 不在关闭入口后再等待一个尚未进入的用户 persist，也不隐式把应用的逻辑事务变为原子事务。
4. 写入新的完整元数据 checkpoint，并执行其持久化屏障。
5. 向两个super槽先后发布带generation和校验、指向同一新checkpoint的CLEAN描述，并分别同步；只有两次发布确认后才认为close成功。
6. 销毁已静止的运行时状态、释放进程内登记和 owner lease。

异常退出、缺少有效 clean 标记、checkpoint 校验错误都返回需要恢复/不兼容状态；不能把 `CLEAN_REOPEN` 误当作支持崩溃恢复。应用可选择其自行管理的备份或离线修复流程，首版不自动猜测和修复损坏账本。

关闭允许存在持久 live allocations，它们正是重开的对象。关闭与“销毁所有对象/丢弃 store”是不同操作，后者必须显式表达。`WRITE_FAULTED` 下使用显式 abort-close 释放运行时资源并要求下次恢复，不将它伪装为一次成功 clean close。

## 5. 可恢复元数据的提交模型

采用 redo 日志与代次化 checkpoint。运行时索引可随时重建，持久权威状态是一个已验证 checkpoint 加其后的有效 committed 日志。禁止在日志提交前原地覆盖唯一有效 checkpoint。

每次 durable alloc、durable free、根更新构成一个独立的 allocator 元数据事务。首版不自动将多次调用合成应用事务；内部可以共用一次 group-commit 屏障，但每个调用只在自己所在事务达到提交合同后返回成功。

首版持久事务按实例串行化验证与提交，相关对象/root 在验证、Pending 与提交结果确定之间保持受保护状态。root_set 与 pfree 不能各自基于过期快照通过校验：pfree 检查直接 root 引用和 persist 控制 pin，root_set 检查对象不是待释放且代次仍有效；两者在同一提交序列中确定顺序。进入串行化 gate 前预取元数据；gate 可跨持久 I/O，并可短暂取得 broker 锁登记 Pending 或发布，但 I/O 时不得持 broker/heap 锁，也不得反向在 broker/heap 锁内等待 gate。普通实例与其他 arena 不受其影响。分组提交属于保持该顺序的后续优化，不是首版性能前提。

记录至少包含 record 类型、总长度、事务 ID、LSN、预期对象/范围代次、操作参数和校验；commit 记录还要绑定其事务记录集合的长度/数量与校验。LSN、record 长度、范围和状态转移均做边界验证，防止损坏文件诱导越界访问。

提交次序：

1. 进入 durable admission，在 gate 外预备日志额度、恢复记录和所需元数据；gate 内短持 broker 最终复核身份、root、控制 pin 后预占范围或标记待释放。尚未提交的 free 范围不能交给其他分配。需要的新 backing 在 WAL 前按单独阶段许可建立。
2. 在持久 control 的短 admission 锁下重查 health 并取得有限 redo 阶段许可，锁外追加完整 redo 并完成日志数据同步。
3. 再次重查并取得本事务 commit 阶段许可，锁外追加对应 commit 标记，并完成其完整持久化屏障。该许可涵盖本事务 commit 写入及所需屏障，不允许扩展到下一事务；仅记录存在但没有有效 commit 的事务不能被恢复为成功。
4. 保持Pending及既有pin，在不持broker时短取metadata_pool结算已预付资源；解pool后按已提交记录在broker下无失败地更新可重建的运行时账本并撤Pending；检查点可以稍后生成。
5. alloc 此时才能向调用者输出持久 handle；free 此时才能返回成功。健康实例才允许该空间后续复用；并发主文件错误已锁存 WRITE_FAULTED 时，成功事实保持但不再复用或开始新的破坏性物理回收。根发布此时才能报告新的根已生效。

步骤4到返回之间保留commit gate和admission责任；资源先结算再发布，避免并发pfree/物理清理扣减尚在reserved的used。Pending名额自身的ticket.reserved保守留到撤Pending后再退，既有pin最后释放或移交；全部责任收口后才退出gate/admission。不增结算状态字段。gate内仅允许结算已预取资源，不允许增长或新申请；pool不反向等gate，也不与broker嵌套。

日志与 commit 的同步可以在证明等价顺序的实现中合并，但必须用撕裂写和重排序故障测试证明恢复不会接受缺少完整 redo 的 commit。文档不能用“调用了 flush”替代此证明。

待提交 free 也不得提前把 freelist/remote next 写进用户 payload、清零对象或丢弃其 backing：如果崩溃发生在 commit 之前，恢复账本仍可能把它视为 live allocation，必须保留此前已持久化的数据。持久前端只能在提交后执行这些普通回收动作。

并发前端和持久前端的成功含义不同。volatile 的 remote-free 发布成功可以完成逻辑释放；recoverable free 不能在仅入队时声称持久释放成功。durable 操作的同步慢路径不进入受限实时 profile。

### 5.1 失败与 WRITE_FAULTED

在任何持久修改之前可确定的参数、容量或日志配额错误，可以返回已知无修改的失败。本调用结果与实例 health 分别反映事务事实和后续可写性，不能相互替代。若 commit 已可能生效但无法确认，或主文件 data persist 的所需持久阶段未知：

- 返回 `COMMIT_UNKNOWN`，arena 进入 `WRITE_FAULTED`。
- 停止签发新的 durable alloc/free/root 写入、checkpoint 覆盖、空间复用与破坏性回收阶段许可；不能尝试“回滚”并马上把待释放对象交给另一次分配。故障前已获许可的有限阶段仍可能完成或在故障后物理进入 OS，按其真实结果收口。
- 错误涉及的对象和范围保持隔离；不能告诉调用者“free 失败所以对象一定仍然存在”。恢复后由 handle 身份与恢复账本查询确定结果。
- 可提供只读诊断与受控关闭/重新打开；应用不能靠盲目重试同一个 free 推断提交结果。
- 其他独立 arena 不因此失效。VM 映射回滚失败导致的范围 `QUARANTINED` 与持久性不明导致的整个 durable arena `WRITE_FAULTED` 分别报告。

数据 persist 的 I/O 错误即使没有修改 allocator 日志，也不能继续暗示 store 完整可写。首版采取保守策略：持久主文件的 outcome-unknown 错误进入 `WRITE_FAULTED`；外部会话映射的错误只影响其映射/数据操作，不自动污染无关的 durable 主文件。

health 锁存和每个新阶段许可在同一短 admission 锁下排序，锁内不等待 gate/broker/OS。在 gate 内最终 broker 校验之后、执行后端阶段之前仍须取得许可；早期 health 检查不能授权后续无限次写入。commit 阶段许可覆盖完整 commit 写入/屏障协议，不能因 commit 写出后的某个屏障未启动而宣称事务未提交。已确认 commit 必须用预留资源完成 runtime publish 并返回 OK，即使同时 health 已故障；诊断/query 暴露该 health。仅 redo 已发且 commit 从未获许可、能证明没有提交时，调用返回 WRITE_FAULTED，或本调用自身已知 I/O 错误对应的 IO_ERROR，不能误报该事务 COMMIT_UNKNOWN。

主文件 redo I/O 的持久结果未知仍会关闭该实例的新修改，但“日志字节是否落盘未知”不等于“事务是否提交未知”：从未允许写 commit 的事务保持确定未提交事实。所有已进入调用留在 admission 中直到结果分类、runtime 发布和责任收口；执行中保持 Pending，唯一 executor 分类后把未知责任转为不可复用的终态记录，不留下无人执行的活动 Pending。诊断或超时不能抢走 executor。完整交错规则和资源处理见 [故障恢复](failure-recovery.md)。

## 6. Checkpoint、日志回收与恢复

### 6.1 Checkpoint

首版使用短暂停止 durable 元数据修改的 checkpoint；普通 volatile arena 不受影响。不承诺首版无停顿 checkpoint。

1. 确定一个已提交 LSN 截点，在不覆盖当前有效 checkpoint 的位置构造新快照。
2. 写完快照及其校验并同步。
3. 在较旧superblock槽中写入更高generation、快照位置、截点、新WAL epoch和校验，再同步发布。
4. 将另一super槽镜像为同一manifest、再次递增generation并同步。只有两槽都确认指向新checkpoint/epoch后，才可回收被覆盖的旧日志与旧快照，并允许新epoch从WAL起点复用。

checkpoint开始前先关闭durable admission并排空全部已进入调用，包括data persist、等待commit gate以及commit已持久但runtime尚未apply的操作。等待期间不持control/broker/gate；排空后重查health再开始快照。单次事务持commit gate，在gate内短持broker做最终验证/Pending与runtime发布，I/O期间不持broker。不能仅因journal当前无写者就拍摄遗漏已提交事务的快照。

第一super已确认、第二super可证明完全无效果地失败时，返回IO_ERROR并保持health=READY，但内部CHECKPOINT_HELD让admission继续关闭，新修改返回BUSY。现有control阶段/manifest字段记录责任；显式checkpoint或满足quiescence的正常close唯一claim并继续缺少的镜像，不重做已完成发布，不开始新checkpoint，不无限重试。确认镜像前不回收旧WAL/CP；重试结果未知则WRITE_FAULTED，abort-close可直接退出。普通写入/flush错误若不能证明无效果，直接走未知结果合同，不能用read-back猜测磁盘事实。详见候选格式和故障恢复。

至少保留两份代次化描述与足以完成一次 checkpoint 的备用空间。日志将满时先触发 checkpoint；无法在配置预算内继续且尚无 I/O 不确定性时返回资源不足，不覆盖恢复所需记录。日志/快照空间均计入 arena/主文件预算，不临时向外部 heap 借空间。

### 6.2 Open/recover

1. 获取进程内登记与 OS owner lease，校验文件身份和基本长度；失败不写文件。
2. 检查 superblock 候选的版本、geometry、范围、generation 与校验，选择最高有效代次。没有有效候选则报告损坏。
3. 载入其 checkpoint，扫描其后的日志。只有记录集合完整、校验通过且具有有效 commit 的事务才可 replay；重复 replay 必须幂等。
4. 合法的末尾未提交或撕裂追加可以被忽略；已经声明提交的中间日志损坏、LSN 断裂或非法状态转移不得当作普通尾部截断静默略过，应报告 `CORRUPT`。
5. 根据持久分配记录重新构建范围账本、空闲索引、对象代次和根；清零并重建锁、线程记录、remote 状态和 native handles。
6. 对交叠、越界、重复身份、root 指向失效对象进行完整一致性检查；恢复完成前不开放分配。
7. 完成必要的恢复 checkpoint/运行状态标记后才向调用方发布已打开 arena；若该持久化步骤不确定，保持失败且不开放可写入口。

崩溃可能发生在 commit 持久化之后、成功结果返回之前。这种操作在恢复后可以存在：alloc 可能成为未被应用记录的对象，free 可能已经生效。该事实需要由 orphan 枚举和 handle 查询处理，不能承诺调用方看到失败就等于磁盘上没有提交。

## 7. 数据 persist、根发布与 orphan

`persist` 针对明确的地址/handle 与长度执行写回和所选同步屏障，返回实际完成阶段。它不自动解析对象引用，也不把多个范围或文件变成原子事务。硬件断电保证受经验证的文件系统与设备 flush 合同约束。

主文件data persist进入durable admission，broker下最终验证存活identity/generation/范围并取得已有descriptor控制pin，锁外按阶段许可执行I/O；结果分类和必要health锁存完成后才退pin并退出admission。pfree/backing变更在最终验证时遇冲突pin返回BUSY。该pin只保护本次库调用，不替应用保护任意读者或与persist并发改写的业务数据。persist可以不持metadata commit gate，与不冲突事务并行，但checkpoint/close必须等待它退出。

推荐的持久对象发布顺序：

1. durable alloc 提交分配身份，得到 handle；此时对象内容尚未承诺有效。
2. 应用构造 payload，使用 offset/handle 表示需跨重开保存的引用。
3. 对 payload 执行显式 persist，并确认所需屏障成功。
4. 通过独立的 durable root 更新发布该 handle；root 更新本身属于 allocator 元数据事务。

上述每次调用的持久提交语义适用于 `RECOVERABLE_METADATA`。在 `CLEAN_REOPEN` 中，alloc/free/root 更新首先修改运行时权威状态，只有成功 clean close 的 checkpoint 才确认这些元数据可重开；data persist 不能单独把该模式变成崩溃可恢复。`VOLATILE` 不提供持久根。

root 只允许指向本 arena 主文件中的存活持久 allocation，且包含代次；不能指向外部文件映射地址或临时对象。首版以固定整数 root ID 定位根槽，最大根槽数在创建时配置，空根使用明确无效 handle；单个根槽更新是原子发布单位。多根业务原子性由应用用一个间接根对象实现。allocator 可以检查 handle 合法性，不能证明应用是否已经完整持久化 payload。

释放前应用需先移除根或其他应用引用；仍直接被持久根指向的 allocation 返回占用错误。allocator 不自动遍历应用对象图，因此不能检测全部间接引用，也不能替应用解决读者生命周期。

必须提供存活分配枚举和根枚举，包含 handle、大小/类别、代次等足以让应用恢复关联的信息。提供“没有被 allocator 根直接引用”的候选枚举时，明确这不是垃圾证明：对象可能被其他 payload 间接引用。应用确认无人引用后再 durable free，恢复不能自动回收这些候选对象。

## 8. 外部文件映射的首版边界

首版外部 `sa_mmap` 类映射仅在当前会话有效，不写入持久对象目录，不提供自动重开 resolver。持久根不能指向这些映射。

- 持久 ledger 记录主文件持久对象和固定控制区。会话映射只能取其中未分配、runtime ledger 可用且 backend unit 边界允许安全替换的范围，并进入 runtime ledger，阻止同会话的 durable alloc 与之重叠。主文件某个仍有 live 对象的大 view 内的逻辑空闲页，不因此自动变成可替换 backing 的区域。
- 调用方在正常 close 前显式撤销所有会话映射，否则 close 返回 `BUSY`；崩溃恢复重建账本时没有这些会话映射，其原占用范围重新按主文件/保留态后端规则建立为空闲。这里没有需要 replay 的持久文件 handle。
- 会话内对外部 SHARED 数据执行 persist，与主文件 allocator 元数据事务互不原子；必须分别返回各自结果。
- PRIVATE/COW 修改不会因 arena close 或恢复而持久化到原文件。
- 外部文件内容被其他进程截断或改写的风险由应用协调；allocator 不假设外部文件永远不变。

后续若增加显式 reopenable profile，需要稳定 backing key、文件身份/版本、区间、共享属性及调用方 resolver；只恢复验证匹配的 SHARED 映射，不重建 native handle bit pattern，不隐式恢复 PRIVATE COW 内容。resolver 缺失、文件变化或区间不匹配必须阻止该映射恢复。这仍不提供跨文件事务。该扩展不进入首版 API 验收范围。

## 9. 验收场景

- 各模式创建、关闭、换基址重开；clean store dirty 标记在允许修改前持久化；非正常退出的 clean 模式拒绝伪恢复。
- 同一主文件路径别名、硬链接、重复 handle、同进程重复打开和第二进程竞争都返回确定结果；进程崩溃后 lease 可重新取得。
- 多个不同主文件的 arena 并存；多个 arena 的外部 SHARED payload 映射可以共用一个数据文件。
- 在 redo 写入、redo 同步、commit 写入、commit 同步、运行时应用、返回调用者的每个边界注入中断和 I/O 错误，并交错主文件 persist 故障；验证确定未提交、确认提交且 health 故障、真正结果未知不会混淆。
- 在 checkpoint 数据写入、描述发布、日志回收各边界注入撕裂与乱序，验证总能选到完整代次，或明确报告损坏。
- durable free 的 outcome-unknown 不触发复用；重开后可查询该 handle 的最终状态；恢复不存在重复分配和重叠区间。
- 已提交但尚未返回的 alloc 可在枚举中找到；没有直接 root 的对象不被自动回收；root 永远不能解析到已释放代次或外部 map。
- 4 KiB/16 KiB geometry 不兼容、文件缩短、非法长度/offset、错误 checksum、未知 feature bits、32 位容量超限均 fail closed。
- 显式 data persist 后 root 发布；对未执行 persist 的 payload 不作虚假恢复保证；外部多文件部分同步失败有独立结果。
- 进程 kill、虚拟故障后端的 torn/reorder 注入与受控断电是不同验证层次。只有前两者通过不能宣称已证明所有设备断电持久性。

## 10. 尚待实机验证

实施前需要给出 POSIX/Windows 全文件 lease 生命周期、支持的本地文件系统、各系统 flush 屏障、文件创建/长度元数据同步次序、稀疏文件磁盘满行为的测试记录。锁与同步失败必须 fail closed。

Windows 的文件视图写回与文件 handle 同步不能混成一个未说明的调用；macOS 的普通同步与 `F_FULLFSYNC` 也不能被描述成相同设备保证。平台后端在 [平台文档](platforms.md) 的阶段化 capability 中公开实际合同，持久模式只在满足其所需阶段时可创建。
