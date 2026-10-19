# S-03：操作级流程、资源预留与失败协议

状态：设计协议 v0.1，未实现、未执行平台 spike。本文件把 [架构不变量](architecture.md)、[S-01 字段布局](metadata-layout.md) 与 [S-01 容量预算](metadata-budget.md)落实到每类调用。公开输入输出以 [API 契约](api-contracts.md) 为准，持久提交以 [持久设计](persistence.md) 与 [格式规格](persistent-format.md) 为准；数值配额来自版本化配置，不在本文件复制默认值。

所有“成功点”“最多节点数”和“先后关系”均为后续实现必须满足的设计合同，不表示已取得运行时证据。平台不能满足某操作的后端前提时，预检返回 `UNSUPPORTED`，不能在实现时自行减弱合同。配额和运行策略见 [configuration.md](configuration.md)，公开结构与状态编号见 [c-abi-spec.md](c-abi-spec.md)，完整转移表与可重放反例见 [state-machines.md](state-machines.md)。

## 1. 状态、责任与成功点

### 1.1 四类状态不得混用

| 状态轴 | 权威持有者 | 含义 |
| --- | --- | --- |
| 地址所有权 | broker 的 ExtentDesc 树 | FREE、METADATA、MALLOC_SLAB、LARGE_OBJECT、PAGE_MAPPING、RT_POOL、PENDING、QUARANTINED |
| 后端状态 | BackendUnit 与 TxnStep | reservation/view/private allocation、backing、保护、提交成本和实际变更结果 |
| 资源额度 | LimitBlock 的 used/reserved/limit | 已占用资源与正在预付的资源；不是另一份地址账本 |
| 持久提交 | 有效 checkpoint 与已提交 WAL | 跨崩溃分配事实；不包含线程锁、session mapping 和 runtime 指针 |

对象可以已经逻辑 free，但对应 extent 仍在退休队列中占有 backing。它不能同时出现在可用 FREE 集合，也不能因为回收失败恢复成调用方仍可使用的对象。

持久格式的整个 runtime 区域 R 自创建/打开建立账本时就归 METADATA 所有，不进入用户 FREE 索引。其内部以现有 ExtentDesc 状态/flags 表达 RESERVED、ACTIVE、PENDING、QUARANTINED；这些是 METADATA 域内子状态，不新增 S-01 字段，也不把同一地址重复计入另一个顶层所有权。未启用的 RESERVED 仍被持久几何独占，只有 metadata-growth 可以取得。统计该域内部 Pending/隔离量时是子集明细，不能再叠加进 arena 容量等式。

`PENDING` 表示一次范围操作独占目标，不是对调用方内存访问的锁。replace、protect、unmap、resize 与 teardown 仍要求调用方先停止访问被改变的范围；相邻未涉及范围不得临时失效。

上表列出四类权威账本，不把 RangeTxn 执行阶段、对象 live/retired 生命周期、arena 准入与健康混成其中一个字段。OS 成功但未发布、持久 commit 已确认但未 runtime apply、逻辑 free 但 backing 仍计 used 都是合法的中间组合。QUARANTINED 表示 broker 拒绝复用或授权该范围；若 reservation 维持失败，它不证明 native 地址仍归库拥有，也不承诺物理 no-access 一定成功。

### 1.2 可观察成功点

| 操作 | 成功/线性化点 | 不意味着什么 |
| --- | --- | --- |
| 小对象 alloc | owner 从空闲集合取 slot、记为 live，并在返回前完成规定初始化 | RAM 永驻、内容清零（非 calloc） |
| 小对象本地 free | owner 完成逻辑归还 | span 已归 broker |
| 普通远程 free | release CAS 成功发布节点，且这是最后一次对象/descriptor 访问 | owner 已接收、空间已重新分配 |
| 大对象 free | 原有 descriptor 被标记退休并进入已有 intrusive 清理链 | OS decommit 已完成 |
| map/页 lease 创建 | backend 和索引准备完成后，broker 发布最终 owner | 多个 OS 调用物理原子执行 |
| 现存 map 修改 | broker 发布各片段的最终状态；完整成功才能返回 OK | 失败必然无效果 |
| RT free | 对专属 slot 状态执行 release-store Returned | 下一次 try-alloc 必定发现该 slot |
| RECOVERABLE_METADATA 修改 | 事务 commit 屏障确认完成；runtime 发布完成后才返回 OK | 应用 payload 或多对象业务事务已经原子提交 |
| CLEAN_REOPEN 修改 | runtime 状态发布 | 异常退出后可恢复；该保证仅在 clean close 成功后成立 |

只读诊断与精确快照不得无保护解引用即将退休的 descriptor。合法对象 free 的 live 保证不能被任意地址查询借用。

## 2. 资源向量与额度预付

### 2.1 两种预留分别计账

操作同时持有两类资源：

1. **记录 credit**：从已有内部 pool 取出的 ExtentDesc、BackendUnit、RangeTxn、TxnStep、对象/映射 descriptor、索引节点和内部 buffer。节点在事务私有链上，不再可被其他请求取得。
2. **配额 ticket**：LimitBlock 中增加的 reserved 数量，覆盖新增 backend charge、metadata extent、ctx/private span/retention、views、Pending、RT、root/journal/checkpoint 等资源。

记录位于已收费的 metadata extent 时，只改变 pool 库存与在用/暂持状态，不再次增加 metadata 地址成本。VOLATILE/borrowed 新取得 metadata extent 或独立内部 buffer 的地址所有权才增加 `M_reserved`。持久模式在建立实例时将 1002 的 quota used 按整个 R 预付，后续 R 内增长的 metadata 额度增量为零，只预付实际新增 backend/view/Pending；不能重复扣一份 R 内地址，也不能把 R 已计费理解成已物理提交。PoolFamily 的 `reserved_records` 按 S-01 专指应急库存，不能复用成普通事务预留计数。

同步调用用固定大小的栈 ticket 保存 12 维 quota 增量和事务身份；不增加 S-01 RangeTxn 槽位、不向外部 heap 分配 ticket。函数返回前所有 ticket 必须转成最终 used 或归还；需长期保留的隔离/backing 记录通过原有区内 descriptor 及 used 额度继续计账，不能把栈地址留下。

LimitBlock 的所有慢路径读改写由 metadata_pool 锁串行化。整向量按 `used + reserved + requested <= limit` 一次检查并预付；不能逐维检查后分别无锁增加。所有加法先检查溢出。首版 limit 在 create/open 固定，运行中没有修改 limit 的接口，维护调用的工作量预算可独立调整。

### 2.2 预留与发布次序

1. **只读预检**：验证模式、输入、身份、几何和 capability；在 broker/控制保护下统计后端步骤及最坏所需记录，保存目标 generation 和范围状态。此时不取得用户范围。
2. **检查调用方输出**：可能产生部分结果的映射变更检查完整报告缓冲容量，不足先返回 `BUFFER_TOO_SMALL` 和 required 数量。
3. **额度预付**：在 metadata_pool 锁内预留整 quota 向量，释放锁。
4. **记录预取**：取得普通 pool 记录、内部 buffer 和句柄预算。已有库存不足时，释放本次用户 ticket 及记录，执行独立 metadata-growth 后重新预检；不能握住用户 Pending/backend 额度等待 growth 使用同一额度。
5. **最终校验与占有**：broker 锁内重新检查目标身份、边界及实际需求是否不超过预取值，并按第 7.3 节检查 begin 与最终 publish/rollback 的 revision 信用；成立则递增代次、建立 Pending，并固定旧 descriptor/backend 的控制引用。目标冲突时释放资源并返回 COLLISION/BUSY；需求增长时撤销本次预留并从第 1 步重算，不部分执行。需要新身份的请求若代次空间不足，在 OS 前返回 LIMIT；合法释放及已有责任的清理适用饱和例外。
6. **执行、结算与发布**：锁外执行 OS 及必要补偿，确定全部最终责任；保持 Pending 和所有 pin，在 metadata_pool 保护下把实际资源 reserved→used、扣减已确认释放的旧资源并退还多余资源信用。之后进入 broker，无失败地发布成功、完整回滚或隔离状态，推进当前 revision 并撤销 Pending；解锁后归还单独保留的 Pending 名额，完成记录/责任移交后才放 pin、输出结果或把新 slab 交给 heap。begin 时取得的 operation ID 不作为结束发布时的旧 revision 写回。

新 owner 对外可见前，其资源额度必须已经完成结算；不能只以 used+reserved 总量保守为理由先发布，导致另一线程退休新 owner 时从尚未建立的 used 扣账。Pending 名额是本次调用单独持有的控制信用，始终留在 ticket.reserved，直到 broker 撤 Pending 后才退还；该短间隔仅保守占用名额，不与新 owner 的资源账混用。反向资源释放也只有在 backend 确认资源已消失后才扣减额度；不得先给新请求释放预算，再发现旧 commitment 仍在。

所有可能失败、需要新配额或 native 调用的工作都在上述最终结算之前结束；结算不再申请资源。随后的 broker 发布使用已有记录、revision 收口信用及仍持有的变更权，不允许重新失败并回到 OS 执行。用户指针/handle 和 heap 可分配引用不得提前输出。合法逻辑 free/完整 page_free/RT destroy 只移交已有退休责任，不在其成功点假装后端费用已经释放；后续清理另走此结算协议。

新旧 view、备份和目的范围同时存在时，按峰值总占用预付，不按替换后的净增量预付。backend 的真实上界无法从能力与步骤计划确定时，拒绝该操作，不执行可能越过用户上限的试探分配。

metadata-growth 自身使用第 5 节的独立 ticket 和应急 credit。普通慢路径可以在一次联合 growth 中补齐多个不足 family；若增长失败或重试时仍无法取得完整 credit，返回明确的元数据/地址/后端错误。普通路径不属于 RT，无条件成功或严格步数不由预检重试承诺。

最终校验仅因并发变化导致计划失配时，首版最多进行 **1 次内部重新计划**：释放本次 ticket/记录，重新预检并重新取得资源；第二次仍失配则返回 BUSY，用户目标范围尚未进入 Pending/OS 修改。独立完成的 metadata-growth 可以保留，不能为制造“零副作用”把其合法库存拆回去。原本就是目标冲突则按 COLLISION/BUSY 返回，不以内部循环等待该目标变空。这是计划重算次数上限，不是 broker 等锁、普通 remote CAS、OS 调用或全部 malloc 步数上限。

### 2.3 记录符号与通用上界

| 符号 | 含义 |
| --- | --- |
| E / B | 新 ExtentDesc / BackendUnit 数，旧被 pin 的记录不重复计算 |
| X / J | RangeTxn / TxnStep 数 |
| Sl / L / M | SlabDesc / LargeDesc / MappingDesc 数 |
| Ctx / H | ThreadCtx / Heap 数 |
| R / T | 未存在的 RadixNode / PageTile 数 |
| RP / RC / RB | RTPool / RTClass / RTBinding 数 |
| Buf | 已计费内部可变 buffer，包括 steps 数组、RT 状态和可选诊断侧表 |

对于一个已有 FREE extent 内的单后端单元获取，原 E 复用为目标，最多额外 E=2 形成左右 FREE；B=1、X=1、J=1，另加最终 owner 与索引。backend 为空的保留范围不凭空收费一个 active BackendUnit。

多 view 用户事务采用 S-01 的保守上界：`E <= 3k+2`、`B <= 2k+1`、`X=1`、`J=k`、新 `M<=1`，其中 k 是预先枚举的独立后端动作数。替换、回滚及跨源/目的 resize 的动作均须纳入计划；共享 section 父记录另按实际所有权计划核对是否包含在 B 上界内。未证明可复用的记录不得抵扣。64 位记录成本上界为 `736k+656 B`，内部 buffer、索引、备份和页取整另计。

一个 TxnStep 只允许产生有限且预先计数的状态片段。后端动作会产生更多片段时，先细分 k，再计算 credit；不能在 OS 执行一半时扩充工作区。

大对象的快速索引只覆盖合法返回地址所在页，普通 slab 覆盖全部合法 slot 起点页，RT 按类覆盖全部合法起点页；metadata 与会话 map 不建立对象 free 索引。R/T 按本次缺少的前缀计算，发布时复用已经存在的目录并归还多取的节点。

## 3. 锁、pin 与 callback

| 保护 | 可以做什么 | 禁止做什么 |
| --- | --- | --- |
| lifecycle | 登记/关闭状态变化、身份检查 | 持锁进入 OS、journal 或普通对象路径 |
| metadata_pool | quota 向量、pool 库存、记录 credit 转移 | 与 broker、heap、central 嵌套；持锁调用 OS |
| metadata_growth | 串行化一个增长原语，短暂进入 broker | 进入 heap/central；执行另一次递归 growth |
| broker | 所有权树、FREE 索引、Pending、范围 pin、目录发布 | 申请新记录；OS/file/flush；用户 callback |
| heap/shared | owner 私有或共享 heap 状态、摘取/发布 slab | 嵌套 broker、metadata_pool、OS 或 callback |
| central | orphan/retire 的唯一接管与 intrusive 链 | 持锁进入 broker/OS；夺取未协作的活动 owner |
| journal/commit gate | 一个持久事务的最终验证、WAL、短暂 broker Pending/publish | I/O 时持 broker；持有 heap/central；反向从 broker 等待 gate；申请新元数据 |

metadata-growth gate 可以跨自身 OS 阶段保留；broker 在 OS 阶段始终释放。journal/commit gate 可以覆盖本实例的持久 I/O，并在 I/O 前后短暂进入 broker，唯一顺序为 gate→broker；不能在 I/O 时同时持有范围锁。生命周期 admission 和 checkpoint gate 是控制状态，不要求普通快路径逐次更新全局原子。

跨锁交接由已预留事务、descriptor pin、RETIRING/PENDING 等状态保持责任。refill 在 broker 完成后再进 heap 发布；退休先从 heap 摘除并固定状态，解锁后进 central/后续 broker。回滚需要回到 heap 时重新取得其锁，不保留原锁等待 OS。

首版 C ABI 不提供用户 callback，不通过用户回调生成日志、补充空间或决定 rollback。错误使用调用方输出结构和显式 query。任何内部诊断后端也不能在锁内调用用户代码；RT 完全不调用诊断 I/O。操作 ID 用于诊断，不建立无限增长的历史结果目录。

### 3.1 唯一完成责任与控制 pin

同步调用栈持有本次 RangeTxn 的唯一执行权；broker 挂接 Pending、固定 target generation 及 Extent/Mapping/Backend control pin 后才可解锁执行。其他线程能观察 active list，不代表可以调用尚在执行的补偿、关闭其 backend 或代替其发布。超时、owner 休眠、OS 阻塞和 watchdog 报警都不能授权清除 Pending。首版不支持异步终止线程或 longjmp 越过执行中的分配器调用；应用应在调用完成后的安全点协作停用。

pin 只保证记录存储不回收，Pending 保证范围不被冲突修改，调用方 quiescence 保证 payload 不被并发使用。三者不能互相替代。control pin 在所属控制保护下取得，先检查计数上限；查询把可变字段复制成值后使用，不以 pin 为由无锁读取变化中的链。最后一个 pin 释放前，最终报告值、统计所需值和错误值必须已经复制；释放后不能继续解引用旧记录。

RangeTxn.state 与 Pending/终态发布由 broker 保护；TxnStep.effect/native_error 和执行中的 completed_count 由唯一执行者锁外更新，普通诊断不得仅持 broker 锁就读取这些字段。flags 在 Pending 前形成只读步骤计划。运行中只读诊断使用 broker 所保护的阶段/目标，具体步骤结果由执行者在受保护安全点或最终发布时复制。逐步内部编码及非法组合见 [状态机](state-machines.md)。

执行者完成 OS 与必要补偿后，保持同一 Pending 与 pin，先完成第 2.2 节的资源结算；随后 broker 内验证责任不变量并一次性发布全部片段、索引和身份，撤销 Pending/推进结束 revision。该验证是既有独占责任的内部断言，不能变成“额度已经结算，才发现普通竞争需要重试”；该阶段不申请记录、不执行 OS。只有完成这次单向交接后，既有 retired/quarantine 责任才可被维护者独占领取。原执行者与维护者不能同时推进同一 native 责任。

首版同一 MappingDesc 的片段目录、保护、backing 与长度变更串行，即使请求范围不相交。最终预检在 broker 下用既有 state 取得变更权并 pin，其他同 handle 变更返回 BUSY；最终发布时解除变更权。不同 mapping 与不相交范围仍并行。该规则防止另一事务在预取后改变 fragment_count/目录，迫使最终发布临时申请记录；pin 本身不排斥这些修改。

map_persist 等锁外使用现存 backing 的控制读取在 broker 下确认目标未被修改并取得相关 pin，冲突 Pending/变更权存在时返回 BUSY。mutator 最终校验发现其他读取/persist 仍持有冲突的 backend/映射 pin 时同样返回 BUSY，不把自身刚取得的事务 pin误当外部冲突；只有互不冲突的单元可继续。普通 query 在 broker 内复制值后返回，不隐含供调用方长期使用的 pin。

## 4. 创建、自举、借用与销毁

| 操作 | 预检及资源向量 | 流程和成功点 | 失败与资源责任 |
| --- | --- | --- | --- |
| owned VOLATILE create | C/P/G、64/32 位上限、模式与默认配置；S-01 的 M0、L0、普通库存和两组应急 | reserve C→启用 64KiB 控制岛→原位 header/locks/pools/shared heap/root→登记 METADATA+FREE→验证最低额度→输出 handle | 控制岛不可用时仅用固定栈/native 状态清理 reservation；无法收口按下节交付 cleanup token；不输出可分配半成品 |
| borrowed init | 可写独占 P 对齐 buffer、容量；与 owned 相同原位记录预算，backing 不归库 | 原位建立控制块和逻辑范围，不执行 reserve/commit/protect/remap；完成后输出 handle | 可以改写获授权控制字节，不承诺初始化失败恢复原内容；不得 unmap 或关闭调用方 backing |
| thread attach | 非关闭状态、ctx 数量、累计 owner token、Ctx=1/H=1 | 先预付和取记录，锁外初始化；lifecycle 下复核后登记，发布 ctx；未公开 token 可消耗但不得回绕复用 | 初始化/配额失败归还记录；不隐式创建 slab；不改动现有线程 |
| destroy | 调用方 quiescent；无 live 对象、map/lease、RT、ctx、Pending 和运行中控制操作 | 标记 CLOSING→处理已有退休/隔离记录的最终清理→释放 owned reservation 或撤销 borrowed 控制身份→handle 失效 | 前置不满足 BUSY 且不开始销毁；后端清理失败由 cleanup token 接管仍可定位的控制岛与责任；旧 handle 禁止继续调用 |
| discard | 调用方明确同意全部指针失效且已 quiescent | CLOSING 为对象整体失效点；逐项清理 handle/view，最后解除控制岛 | 错误不能使已 discard 的对象复活；未完成责任交 cleanup token，原 handle 不再可调用 |

销毁不得先解除控制岛，再试图从已解除内存读取待清理链。owned arena 的原生范围由多个后端单元组成时，先清理非控制单元，最后释放控制岛。每次已完成的清理从账本移除，重试不重复关闭 native handle。borrowed buffer 不执行这些 OS 清理步骤。

持久创建不复用会覆盖现有文件的普通 init；它遵循第 9 节的独占、格式化和持久发布流程。

### 4.1 自举失败后的 cleanup token

reserve 成功、bootstrap 启用失败、随后释放 reservation 再失败时，不能假设已经存在可写的区内控制块。create/open 的结果因此带调用方持有的固定大小 cleanup token，正常成功或完整清理后该 token 无效；清理未完成返回 PARTIAL_FAILURE，不输出可分配 arena。

控制岛建立前只允许取得一个顶层 reservation 和其一个固定 bootstrap 单元，不打开文件、不建立任意数量的 views、locks 或后台资源。token 的早期形态只记录 base/length、bootstrap geometry、原生阶段和经后端证明安全可重试的资源身份；此限制保证 token 固定大小，不发生溢出后转外部 heap。

控制岛已经可用后再取得文件和其他可增长资源，失败时 token 可以引用保留的 cleanup-only 控制岛及已有区内记录。该 token 不是普通 arena handle，不能分配、查询对象或注册线程，只能由其唯一调用方执行 cleanup retry/查询。一旦 cleanup token 接管，旧 arena handle 即不可调用；即使控制岛仍被映射，也不能以旧 handle 重试 destroy/close。token 可移动交接，不能复制后并发或重复清理。

`cleanup_retry` 使用固定栈和仍有效的区内记录，只重试后端能明确证明仍由库拥有、身份未复用的资源。POSIX close 返回某些错误时，FD 可能已经关闭并被其他线程复用；这类未知身份不得盲目再次 close。token 的 `UNKNOWN_NATIVE_ID` 标志保存该诊断事实，可以继续清理其他已知资源，但存在该标志时返回 OS_ERROR 而不清空为“全部确认成功”。token nonce 只供诊断，不证明未知 native 身份仍归库所有。

最终释放控制岛前，将最后清理所需的固定信息转入调用方 token；即使控制岛解除结果不明，也不得再解引用可能已失效的内存。native 资源成功清理后立刻撤销相应责任，token 只有在全部责任已确定结束时才无效。普通 destroy/close/discard 在同类最后控制岛失败时使用相同 cleanup 结果合同。

同样的身份规则适用于 native 地址：相同 base/length 或查询到当前位置有映射，不足以证明它仍属本库。局部 reservation 失证后 broker 隔离该范围；cleanup 不用 fixed replacement 抢回洞，也不以整段 unmap(C) 覆盖可能已被第三方复用的部分。只清理后端可证明仍自有的单元；未知身份保留诊断并继续处理独立责任。若连受影响范围都不能界定，或控制元数据已被破坏，属于后端核心不变量失效，停止使用该实例，不能伪称其余 FREE 仍安全可用。

## 5. 元数据尾部切块与增长

### 5.1 已有控制岛/extent 的切块

在 metadata_pool 保护下检查未切分尾部，按 family alignment 建立 PoolChunk 或独立内部 buffer；空间足够时无需新增 broker 范围或 backend charge。新增库存仍属于既有 metadata extent 容量，即 quota 的既有 used，不增加 quota reserved；不能把 64KiB 控制岛的未用尾部当作用户 FREE。S-01 的 `M_reserved` 表示已划归 metadata 的地址总量，与 LimitBlock.reserved 的在途信用含义不同。

对象记录/目录路径的准备全部在用户范围 Pending 之前完成。metadata 自身不接入对象 PageTile，不产生“先有元数据索引才能分配元数据索引”的递归。

### 5.2 一个可证明的增长原语

增长有三个明确输入域，记录上界相同：

- owned VOLATILE：单个 `FREE + backend为空`、可独立替换且符合 G 的范围，新增 metadata 地址额度。
- borrowed VOLATILE：单个 FREE 范围可以引用该 caller buffer 的不变 borrowed BackendUnit；只改逻辑所有权，不新建 backing、不增加已按 C 计费的 backend charge，也不推导 OS 权限。新增 metadata 地址额度，保守预取但未用的 B 信用归还。
- 持久实例：R 内单个 `METADATA/RESERVED + backend为空` 且符合 G 的范围；保留 METADATA 域所有权，仅改变内部启用子状态。1002 used 已经是 R，不增加 metadata 地址额度，只预付本次实际 backend、ordinary view 和 Pending。

跨现存活 view 的 owned 增长不属于该原语；先由独立维护把相应单元恢复成所属域的可启用 reservation，否则失败。VOLATILE 普通 FREE 和持久 METADATA/RESERVED 不可相互混用；持久 R 耗尽不能向 payload 偷借控制空间。

一次增长固定借用 **E=2、B=1、X=1、J=1**；R/T=0。64 位一组 624B，32 位一组 480B。S-01 预置两组，第二组用于第一次故障长期占用记录后的后备，不允许并行增长各占一组。

| 阶段 | 动作 | 必须保留的责任 |
| --- | --- | --- |
| G0 预检 | growth gate 下计算缺少 family、补应急记录、chunk 头、目标记录和取整，按三个输入域预付新增额度 | 单元必须同时装下应急补充和普通目标，不能先 commit 再发现装不下；持久 R 不重复计 M |
| G1 借额度 | 从 emergency 专链取得完整一组 | 应急不能出借给一般用户 map/slab 请求 |
| G2 占有 | broker 下复核所属域/backend/G，原 E 改目标，至多建两边 E，标 Pending | 整个目标由同一事务控制；持久两边仍是 METADATA/RESERVED，不进入用户 FREE |
| G3 后端 | 锁外提交一个完整单元 | 只使用已有 J 记录，无目录增长 |
| G4 成功 | 新单元原位建立缺口四族记录但暂不供其他请求取得；保持Pending/pin先结算资源，broker发布METADATA/ACTIVE并撤Pending，再退Pending名额；在pool保护下先补足两组应急，再发布普通目标family | 使用中的 E/B 仍计入 used；新记录补水位而非假装旧记录归还；持久两边保持 METADATA/RESERVED |
| G5 已知回滚 | VOLATILE恢复原FREE；持久恢复METADATA/RESERVED；归还本组记录/ticket | 必须确认原后端/所有权状态完整恢复；持久1002 used仍为R |
| G6 不确定 | 隔离整个目标，保留相关 E/B/X/J 直到诊断可收口 | 不把未知内部进一步细分，不虚报额度回收；持久目标仍属METADATA/QUARANTINED而非用户FREE |

当剩余库存不足一完整组时停止后续 growth，但仍允许已有库存支持的操作和合法普通 free。两组应急不承诺无限次后端故障后的增长能力；每次开始前都必须有足以描述所有结果的资源。

若 G6 已能把全部不确定状态固化到 E/B 等永久描述中，可归还不再必要的 X/J；未能证明无引用时保留。是否归还不影响“不足一组即禁止启动”的检查，也不得以强行释放故障记录补水位。

## 6. 对象分配、realloc、free 与 trim

### 6.1 alloc/calloc/aligned

| 路径 | 记录/配额 | 阶段与发布 | 失败 |
| --- | --- | --- | --- |
| ctx 热路径 | 无新增记录；span 配额已预付 | 本地 slot→live；calloc 在返回前清请求长度；可选请求表同步写入 | 没有本地 slot 转慢路径，不直接声明 arena OOM |
| shared 无 ctx | 与 ctx 相同，使用预建 shared heap 的锁 | 不隐式登记；锁只覆盖 owner 状态 | 不持 shared 锁进入 refill |
| 新 slab | E<=2、B=1、X/J=1、Sl=1、缺少的 R/T、可选侧表 Buf；S 大小 backend/private span 配额 | 预留→Pending→启用 backing→初始化 Sl/索引→broker owner→heap 发布 | 完整回滚则无对象；backend 不明则隔离并归还未用 quota，不输出指针 |
| 大对象/大对齐 | E<=2、B=1、X/J=1、L=1、返回页 R/T；目标范围及临时 alignment 空间预算 | 对齐与 round-up 全部先检查；原 FREE 中切目标及前后缀；初始化 L 与返回页索引后发布 | 目标未完整准备前无用户指针；不能先分配后发现对齐溢出 |

普通 malloc/calloc 的零大小及算术规则按 API 执行；calloc 不能把 entire arena 或未请求邻居当作清零范围。小对象无精确 requested 侧表时，统计不能假装有 requested-live 数据。

private span 额度不足不等于全局没有对象容量。owner 在安全点可以把完整 slab 移交 central、归还 private 信用后再取得 span，或通过 shared heap 服务请求；移交不搬动 live payload。shared heap 同样受全局容量/backend/metadata/empty 限制，不能用命名变化规避预算。

### 6.2 realloc

1. `p=NULL` 使用 alloc；`new_size=0` 使用 free，成功输出 NULL。其他情况必须是所属 VOLATILE 对象的合法起始指针，调用方停止并发访问。
2. 小对象新长度不超过现有 usable 时，原址保留；可选 requested 表更新。首版不为缩小小对象强制迁移到另一 size class。
3. 大对象缩小时先更新逻辑长度；物理尾部回收是可延后 trim，不把拆分记录不足变成“已经缩小但 realloc 失败”。尝试物理拆分时另走完整资源预检。
4. 原址增长仅在邻接 FREE 及 compatible backend 满足时尝试；保持旧 descriptor、旧页和内容不变，Pending 只占新后缀。新后缀成功后再发布新大小；失败完整释放新后缀，旧对象仍有效。若后端只能通过破坏旧内容扩展，不采用该路径。
5. 原址无法扩展则独立分配目的块，成功后拷贝：有 requested 记录时复制 `min(old_requested,new_requested)`，无小对象 requested 表时复制 `min(old_usable,new_requested)`；sanitizer 的可读范围必须与之相符。
6. 拷贝完成后逻辑 free 旧对象，并输出新指针。因为合法 VOLATILE free 不会因回收资源不足失败，所以不存在“新对象已输出但旧 free 失败”的半成功合同。目的分配失败时输出 NULL，旧对象和大小完全保留。

### 6.3 free 与退休

| 路径 | 线性化与不再访问点 | 退休责任 |
| --- | --- | --- |
| 本地小对象 | owner 放回本地 freelist 并减 live | 达到空 slab 条件后按 retention 预算保留或摘入退休链 |
| 远程小对象 | 写自身 payload next，release CAS 发布成功；此后不再碰该 slot/SlabDesc | owner acquire exchange/backlog 接收才减 live；未发布者仍计 live |
| 大对象 | 验证起始指针，撤销用户可用身份并以原 L.retire_next 挂退休链 | L/E/B 不释放；没有 X/J 时延后 backend 清理 |
| 最后一个 slab 对象 | live=0、remote/backlog 空、无 control pin 后撤销索引并退休 | 原 Sl.retire_next 作为队列节点，不新建 retire record |

普通 free 不预留新的 RangeTxn，不等待 metadata-growth，不因额度满而失败。退休队列中的 span 在地址账本仍为原 owner 的 RETIRED/RETIRING 子状态，不允许新对象或 map 复用。对象逻辑释放返回 OK，OS 回收失败仅改变健康/隔离诊断。

最后一个 live 归还后，owner 先在自身保护下把全空 slab 标为 RETENTION_DECIDING 并从可分配列表摘下，然后在不持 heap/central/broker 时尝试预付 empty-span 信用；成功才重新发布缓存，失败直接挂既有 retire_next。已有 private-span 信用在实际离开该 private heap 后归还，backend/metadata 信用保留到真实回收完成。该决策不增加新记录，也不使 free 因保留额度不足而失败。

### 6.4 trim/maintain

在步骤/页数预算内从 central 摘取并 pin 已退休对象；释放 central 锁后，从普通库存取得 X/J 和必要范围记录，再经 Pending 回收 backend。无记录时把原 descriptor 放回退休链，报告本次未清理，不复活用户对象。

既有 `ArenaHeader.retire_tail` 指向循环单链 FIFO 的队尾，按 prefix.kind 选取普通 descriptor.retire_next 或终态 RangeTxn.active_next；central orphan 队列使用同样的 `central_tail` 语义。每次 arena maintain 对每类队列只开一个最多 8 项的小批，在 central 保护下独占摘取并固定责任，放入固定 16 指针栈工作区，解锁处理；未完成/预算不足项在批末回队尾，本次调用不再领取。按 steps 预算可少取；两类队列和 shared 冷类访问的最低服务机会、执行顺序及预算前提统一遵循 [并发协议](concurrency.md)，此处不另设调度。不能以易被并发摘取/回收的初始 tail 指针判断“本轮结束”。slab 维护使用既有 class/owned-ring 游标轮转。这些规则避免一次预算总落在同一失败资源上，不承诺 OS 阻塞时间有界或等待活跃 owner 响应。

公开维护预算只有 objects/pages/steps 三维：span 处理消耗步骤，backend 动作按实际覆盖页数同时扣 pages；没有第四个公开 spans 字段。不能在余下页预算不足以覆盖不可拆 backend 单元时仍执行该单元；跳过并返回 MORE_HINT，由调用方以更大预算推进。max_steps=0 不执行维护，其他维度为0时跳过相应类别。

已知成功恢复 reservation 后发布 FREE、兼容合并并归还 descriptor；已知 decommit/discard 未发生时保持退休 backing 可再试；后端范围不明则 QUARANTINED。持久对象的 free 先完成第 9 节提交，才进入此类 runtime 物理回收。

目录节点只在全 arena 相关读取者 quiescent 的维护阶段回收；普通 trim 只能清理无 live slot 的页条目，不在线释放遍历中的 radix/PageTile。休眠线程的私有 freelist 不能由 maintain 强夺。

## 7. 页 lease 与映射事务

### 7.1 预检与报告

所有 map 变更首先检查：长度/offset 为 P 倍数、算术、arena 来源能力、backend 单元边界、split policy、文件长度与访问权、mapping identity、old/new backing 生命周期，以及存储模式的 runtime/persistent FREE 约束。

在只读步骤计划中得到 k，保守要求调用方 report buffer 可容纳 `3k+2` 个本次涉及的结果片段；经证明可合并的计划可以报告更低的明确 required 值。空间不足返回 `BUFFER_TOO_SMALL + required_count`，不执行用户目标范围的 OS 变更。重试时必须重新计划，先前 estimate 不持有额度或地址。

结果包含本次的 operation ID、status、最终 base/length/handle 状态，以及 APPLIED、UNCHANGED、ROLLED_BACK、QUARANTINED 片段；只列本次涉及范围，不穷举整个 mapping 的所有历史 survivor。普通不足错误不能覆盖已经发生的 PARTIAL_FAILURE。正式调用在变更前已保证报告空间，不把错误详情丢到外部 heap。

### 7.2 操作矩阵

| 操作 | 资源向量与预检 | backend 与发布 | 失败/身份规则 |
| --- | --- | --- | --- |
| page lease | E 分割、M/lease 控制身份；borrowed 无 B/OS；可选 ZERO | broker 占有→必要清零→发布 lease | 不获得 borrowed protect/remap 能力；失败无 lease |
| 新 mmap | 多步骤上界、M=1、独立文件引用、view/section/charge 峰值额度 | FREE→Pending；建立 native 引用和 views；成功发布 M/E | 失败撤销已建部分；不能确认 reservation 完整时隔离；无有效新 mapping handle，报告隔离范围 |
| 全部 munmap | pin M/E/B，预取 X/J；新 E 按计划可抵扣 | 停用对象→后端恢复 reservation→片段变 FREE→最后片段消失则 handle 失效 | 未修改失败保留旧映射；部分无法恢复返回 PARTIAL_FAILURE，不宣称匿名内容仍在 |
| 部分 munmap | profile 与每个单元边界；最多保留两侧 survivor，完整 credit | 仅改请求部分；survivor 保留原 ID/generation；洞回 FREE | 不临时解除左右邻居；洞中后续新 owner 不属于原 handle |
| map_replace | 显式目标 handle，当前仍拥有全部目标；新旧双存和恢复能力预检 | 旧范围 Pending并 pin 旧B；受控替换；最终同 handle 管理被替换片段及原 survivors | 不覆盖 malloc/meta/RT/其他 owner；原内容无法恢复时 PARTIAL_FAILURE；不能只恢复地址就声称完整回滚 |
| mprotect | capability、native view 边界、所有权；每步 old/new protection | 逐步修改，成功后发布新保护 | 可证明全部恢复则普通失败；其余按片段报告并隔离不确定范围 |
| map_discard | 调用方明确允许丢内容，检查 backend 回收能力 | 所有权与 handle 保留，更新实际 commitment | 内容丢弃不等于解除地址；部分完成如实报告，不回滚伪造旧内容 |
| map_persist | SHARED、范围有效、请求同步阶段被支持；pin backing | 锁外 flush及文件屏障，返回实际阶段 | PRIVATE/COW 返回 UNSUPPORTED；外部文件错误不自动使主文件 WRITE_FAULTED |

page_free 首版只接受完整 lease，使用已有 MappingDesc.retire_next 保留归还责任，合法逻辑归还不申请新的 X/J；后台物理清理没有记录时延后。borrowed page_free 仅归还逻辑范围，不得解除调用方映射；owned 逻辑页 lease 的物理回收同样可由维护完成。真正 munmap 是 VM 合同，不能套用普通对象 free 的“逻辑成功后延迟全部操作”。

### 7.3 resize/move

首版仅接受无洞完整 mapping，整个源与目的范围由调用方停止访问。

- **缩小**：按 profile 检查尾部单元；保留前缀 identity，尾部采用 munmap 协议。无法完整执行或恢复时返回 PARTIAL_FAILURE 并报告有效范围，不返回含糊的新长度。
- **原址增长**：一次 Pending 同时覆盖源 handle 的变更权和相邻新范围。文件增长不得超过已验证文件长度。旧内容必须保持，新后缀失败则回滚后缀；旧范围需要破坏性重建时只有后端能证明其内容保持才支持。
- **MAY_MOVE**：预付源+目的同时存在的全部资源；目的先建立并完成内容保留，再撤销源。匿名数据通过后端保留或复制，SHARED 建立相同文件 offset，PRIVATE/COW 无能力保留私有修改时预检拒绝。
- **move 成功**：保留 mapping identity 与 generation，更新其现存范围/base；旧地址失效。generation 用于身份生命周期，不能因普通 move 随意改变使 handle 不可追踪。
- **move 后半失败**：若目的尚未接管可完整撤销则保留源；若源已部分解除、不可恢复，则 PARTIAL_FAILURE。所有确定仍有效的源/目的片段都挂在同一 M 下，报告并供 map_query 检查；原“无洞完整映射”条件不再成立时，后续 resize 必须拒绝，调用方先显式清理片段。

#### 分页 revision、Pending 与计数耗尽

map_query 和持久 root/allocation 分页读取同一实例的 ledger revision；并发变更导致 cursor 中的 revision 不匹配时返回 `STALE_CURSOR`，要求从头查询。首版使用已有 `ArenaHeader.next_operation_id` 保存当前 broker 变更代次，不给 MappingDesc 或 S-01 增加未经计费字段。普通 slot 热路径不更新该计数；无关 broker 变更也可以保守地使 cursor 过期。operation ID 不充当长期快照，也不赋予访问已经归还洞的权限。

每次分页调用在 broker 保护下检查实例/cursor、Pending 和 revision，并把本页数据复制到调用方 buffer；不得在解锁后依赖可退休的 descriptor。只要 arena 存在任一 Pending，分页保守返回 BUSY，不返回一页暂时仍为 LIVE 的旧目录。`UINT64_MAX` 为饱和哨兵：进入该状态后，全部上述分页调用均返回 `STALE_CURSOR`，包括要求第一页的新 cursor；此规则优先于 Pending 的 BUSY。标量诊断和完整的调用方操作报告仍可使用，不依赖分页成功。

正常 Pending 事务在 broker 临界区内完成两次不同的代次推进：**begin 建立 Pending 时一次，最终 publish/rollback 撤销 Pending 时再一次**。成功、完整回滚、部分结果和 quarantine 都执行结束推进，即使回滚恢复了原目录也不沿用旧代次。持久 root/allocation 在 commit gate→broker 下遵守相同规则；不能在 WAL commit 后沿用 begin 的旧代次发布目录。一个事务只在最后一次 broker 发布对查询暴露结果，中间 OS 步骤保持 Pending；如未来需要额外的可见中间发布，必须先扩展其代次信用规则。

预留的是**推进次数信用**，不是可稍后乱序写回的预分配数字。设当前代次为 `r`、现有 `pending_count` 为 `p`、最大正常代次为 `UINT64_MAX-1`：建立一个新受编号 Pending 前，以 checked arithmetic 验证剩余代次至少为 `p+2`，覆盖现有每个 Pending 的结束推进及本次 begin/end；`p+2`、剩余空间减法和 Pending 增减均先检查可表示性。begin 递增 `r` 并增加 `pending_count`，结束时递增当前 `r` 并减少 `pending_count`。此式适用于所有受编号事务，包括 durable begin/publish；不经 Pending 的一次可见修改需至少 `p+1` 次空间。`RangeTxn.operation_id` 保存本事务 begin 时得到的值用于诊断，结束发布必须推进当时的共享计数，不能把该 operation ID 当作当前 revision。上述信用由已有 Pending 计数推导，不增加持久 ticket 或 S-01 字段。

诊断代次耗尽不能拒绝合法的 VOLATILE free、完整 page_free、RT 逻辑 teardown、已经开始事务的收口或已有资源责任的清理。这些路径若无法在保留其他 Pending 结束信用的同时推进计数，就把 revision 置为 `UINT64_MAX` 并继续；已经饱和则保持哨兵，既有 Pending 仍可正常 publish/rollback。饱和后的清理/维护事务使用 `operation_id=0`，保留其既有“未编号”含义；事务责任始终由现有 RangeTxn 地址、descriptor 与 pin 确认，不能从该 operation ID 定位责任或把它作为可复用的唯一身份。仍有其他前置错误或资源不足时按各自协议返回，但不得仅以诊断计数耗尽为理由失败。非分页普通查询与诊断仍按自身状态规则工作。

需要建立新范围/映射等身份或普通 Pending 的 alloc/map/prepare/growth 请求，在信用不足或已饱和时于 OS 前返回 LIMIT；已有 slab 中无需新身份的普通 slot 分配不增加该依赖。代次禁止回绕或原地重置；关闭后建立新实例使用新的实例身份。该饱和规则仅适用于 runtime 查询 revision，不适用于句柄 generation、owner token、持久 allocation ID、LSN、journal epoch 或 superblock generation；后者耗尽仍必须在对应变更前拒绝，不能以诊断哨兵绕过唯一性或持久排序。

### 7.4 多 view 回滚的确定处理

每个 J 的 effect 只取未执行/已证明无效果、已改变、已恢复、不确定；开始步骤前保持 old/new unit 和共享父后端的 pin。native 调用前保守置不确定，返回后依据平台实际效果分类；不能把错误返回直接解释为没有副作用。completed_count 仅记录进度，不代替逐步 effect。回滚按预先安排的依赖逆序执行，只有 backing、保护与应保留内容全部满足原契约才归类“已恢复”。

已确认恢复的片段回原 owner；已经成功完成且无法无损撤销的片段保留最终状态；未知片段隔离。报告不把这三类混成一个普通 OS_ERROR。实际已关闭的 native handle 立即从 cleanup 责任中移除，剩余资源持续 used，不能双重关闭或释放 quota。

补偿方法必须在 OS 前已有记录、backup、峰值额度及完整 caller report；old protection、文件偏移和 descriptor 本身不是 payload 备份。一个动作拆成更多 native 子动作时先扩大 k 并重算记录与 `3k+2` 报告上界，不能在错误路径临时扩容。每个逆向子动作本轮至多尝试一次；再次失败后不递归“回滚回滚”、不无限重试。仍满足前提的独立责任可继续收口，依赖未知资源的动作跳过并保留。

请求成功发布、合法逻辑 free 或持久 commit 确认都建立各自不可撤销边界。发布后的物理清理失败转 retired/quarantine，不复活旧用户对象；持久 commit 未知也不能按“尚未提交”执行所有权反向补偿。已知原内容丢失、仅剩安全 reservation 的片段即使 native 地址状态明确，也不能声称 ROLLED_BACK，可归入 QUARANTINED 并在实际报告说明。

终态 QUARANTINED 必须退出进行中的 active Pending 链并降低 pending_count，再退本次 Pending 名额，避免永久阻塞全 arena 分页；X/J 证据、相关 pin/backend 及实际或未知费用仍保留，不能跟 Pending 名额一起释放。ExtentDesc.pending 可指向该终态 RangeTxn，表示恢复证据而非仍有业务执行者。Q extent 用既有 free_prev/free_next 组成 X.cleanup_list，绝不同时进入 FREE 索引，也不占用 mapping 的 owner 链。X 退出 active 链后复用 active_next 加入 retire_tail，依 prefix.kind 读取链字段，两链互斥；孤立 backend 责任继续由 steps.old/new_unit 持有，cleanup_list 不混记录类型。只有关联范围和全部后端责任解除后才能归还 X/J。completed_count 在终态清理阶段重置为扫描游标，逐项判断 effect 且计入维护 steps，不新增 refcount 或无界故障扫描。细节见 [状态机](state-machines.md)。

## 8. ctx 退出与 RT 生命周期

### 8.1 detach/orphan/adopt

| 阶段 | 必要条件与动作 | 不变量 |
| --- | --- | --- |
| D0 detach 预检 | 当前登记线程安全点、无正在执行的 ctx 调用；存在任何 RTBinding 则 BUSY，先 teardown 相关 pool | 不让固定 binding 指向已回收 ctx |
| D1 停止 owner | ctx 状态改为 DETACHING，停止新分配；按预算处理归还 | 远程 free 仍可使用稳定 SlabDesc |
| D2 发布 orphan | 留下区内 freelist/backlog/live，central 下逐 slab 转 ORPHAN | 不保存线程栈指针，不分配新队列节点 |
| D3 释放 ctx | 所有 slab 移交且无控制 pin后注销；quota 和 Ctx/H 归还 | owner token 永不回绕复用 |
| D4 adopt | central 下仅一个接管者claim仍为ORPHAN的slab；释放central后，在目标heap有效lease/锁下建立owned环与完整状态，最后release发布新owner token | 旧 owner 已停止；remote head 地址稳定；不能先公开shared token再安装状态 |

detach 不要求 live object 为零，不搬动 payload。可以逐 slab完成移交，但在全部移交完成前不返回 detach 成功。普通 detach 无严格 RT 步数承诺；维护不能对未 detach 的睡眠 owner 执行同样接管。

### 8.2 RT prepare/run/teardown

| 阶段 | 资源与锁 | 成功点/失败 |
| --- | --- | --- |
| prepare 预检 | owned VOLATILE 匿名能力、P 几何、owner/允许 ctx、原生 a32/aptr lock-free、固定类/slot/K、可选锁页 | 计算 RP/RC/RB、4N 状态、每类 P 对齐 payload、完整 R/T、E/B/X/J及 buffer；预付峰值 |
| prepare 执行 | Pending 保留全部类范围，锁外 commit/prefault/可选 lock；原位构造状态和固定 binding | 所有准备成功后发布 pool/bindings；失败解除已取得资源，无法恢复的范围隔离，不返回半初始化 binding |
| try_alloc | 固定 binding 身份校验；仅 owner；最多 min(K,N) 状态检查 | acquire 接收Returned→本地Free，再取目标类；无立即可用 slot返回WOULD_BLOCK |
| free | 固定深度索引到 RC，算 slot，合法唯一释放者 release-store Returned | OK；不申请资源、不写聚合统计、不检查共享队列、不CAS循环，发布后无对象/descriptor访问 |
| maintain | 固定 owner cursor、最多K检查 | 返回实际检查/接收数；一轮覆盖位置，不承诺并发store指定轮次可见 |
| teardown | 应用quiescent，阻止后续binding调用；完整检查无Allocated/Returned，非空BUSY且原pool仍有效 | 检查通过后撤bindings/索引，以已有RP.retire_next及RC/范围记录转交arena退休责任，逻辑destroy返回OK、旧pool/binding失效；OS清理延后失败仅留下retired/quarantine，不撤销成功 |

prepare 的全部上下文已经登记，run 不动态登记。首版不提供 owner 迁移函数；需要改变 owner 时先 quiescent、接收全部归还并销毁空 pool，再按新 owner 重新 prepare，旧 binding 随之失效。要求锁页时失败必须使 prepare 失败，不能以 prefault 替代。

RT teardown 可以进行完整检查和普通慢路径；它不是 RT 调用。检查发现 Allocated 或 Returned 均返回 BUSY；归还尚未接收时由调用方先使用 owner binding 显式 maintenance，再 quiescent 重试 teardown。不能仅凭 owner 旧计数判断全部 slot 已空，也不在 destroy 中悄悄扩大一次 RT maintenance 的预算。

逻辑 teardown 不申请新的退休记录，不要求取得 RangeTxn；pool 的现有控制记录保留到物理清理收口，不能先释放 RP 再把其 retire_next 挂链。后续 arena maintain 可以申请普通清理工作区；不足时继续保留责任。公开接口不提供 STOPPED pool 重试或 RT cleanup token，调用方不能在 OK 后再次 destroy 旧 pool。

具体使用 S-01 既有字段：RTPool.retire_next 在活动状态下充当 ArenaHeader.rt_pool_list 的单链 next；teardown 在 central 控制下先摘除活动链，再把同一字段用于 ArenaHeader.retire_tail 所指的循环退休队列，不能同时挂两链。RTPool.classes/states/bindings 继续持有原有区内数组，RTClass.pool 和 ExtentDesc.owner 继续定位这份退休责任；RC 不需要新增队列节点。RTBinding.next_in_ctx 从各 ctx 的登记链摘除后才归还 binding 记录，RP/RC/状态buffer则在物理清理收口后归还。该协议不改变记录槽位或字段数量。

## 9. 持久创建、打开、提交与关闭

### 9.1 主文件与 runtime 空间

首版持久实例只有一个主文件。进程内 stable file-ID 登记加 OS 非阻塞独占 lease，失败时不覆盖文件。运行时元数据位于独立匿名 control islands；主文件 payload 按 compatible backend extent 懒映射，不用一个巨型 active view 妨碍区内替换。

R 的整个地址域永久计入 METADATA ownership 与 1002 used；ACTIVE/RESERVED/隔离子状态和实际匿名 committed、记录 in-use 分别查询。R 内启用和回收只能在该域转换：可安全解除的空 runtime 单元回到 METADATA/RESERVED，不变成用户 FREE，不降低已按 R 计付的地址 quota；只有 backend/view 资源经确认释放后减相应 used。bootstrap、仍有引用的节点及应急记录不参与这种回收。

公开入口接收调用方 native file_ref/FD，不接路径。新文件由调用方以不覆盖方式创建并保持为空，明确授权库初始化；库复制引用、取得 lease 并核对身份后再次检查 length=0，拒绝所有非空文件，绝不通过 truncate 清空。路径/父目录的创建及 namespace durability 由调用方负责，要求库保证命名空间屏障返回 UNSUPPORTED；库只保证所选文件内容/元数据屏障。

open 在控制岛建立前可以借用调用方保持有效的原始 FD，只读固定上界的 superblock 候选到栈，以取得候选 C/runtime geometry；该预读不执行写入、恢复或取得库自有 FD。建立 reservation 和匿名 bootstrap 后才复制文件引用、取得 lease，并重新读取、核对文件身份及完整格式。前后数据变化导致预保留几何不再匹配时，清理此次创建资源并失败，不能跳过第二次校验。

持久 FREE 仅表明没有持久对象；若所在主文件 active view 仍与邻居 live 对象共用 backend，外部 session map 仍须符合可独立替换的边界。先满足 runtime ledger 与 backend 条件，不能为腾出 FREE 子页临时撤销邻居。

| 操作 | 预检/资源 | 完整流程及成功点 | 失败 |
| --- | --- | --- | --- |
| 新持久 create | 调用方授权的空专用file_ref；几何、root/WAL/checkpoint和runtime预算 | 建立匿名runtime控制→复制文件引用/取得身份与lease→复核length=0→建立长度及初始checkpoint→同步数据→发布并镜像有效superblock→重建runtime并输出handle | 未有效发布的文件保留为可识别半成品；返回错误，不自动打开为有效store；清理余留由token接管 |
| open | 只读文件长度/格式/版本/geometry/代次；可表示地址容量；完整runtime reconstruction预算 | 先lease→选择有效super/checkpoint→扫描已提交WAL→一致性检查→重建runtime→必要运行标记/恢复checkpoint→输出handle | 无有效代次或矛盾CORRUPT；geometry不兼容拒绝；预算不足METADATA_LIMIT；不格式化、不截断修复 |
| CLEAN_REOPEN open | 有有效clean描述且相应checkpoint完整 | 在允许修改前提交新一代dirty，并完成两个super槽的镜像发布屏障；然后开放runtime | 任一dirty镜像写入结果未知不可开放写入；异常退出的旧dirty store不按clean路径伪恢复 |
| palloc/pfree/root | gate外预取完整runtime、日志/目录、root与checkpoint资源；gate内最终验证 | commit gate→短持broker重检并标Pending→解broker→redo/commit屏障→保持Pending/pin结算资源→短持broker apply/publish/撤Pending→解broker→退Pending名额/责任交接→退出gate/admission→OK | commit已证明未发生时可保留旧durable ownership并收口；commit事实未知才COMMIT_UNKNOWN；确认commit必须apply |
| data persist | 合法主文件对象/范围、屏障阶段支持，调用方payload写入协议；durable admission及目标pin | 取得有限persist许可→锁外flush/文件屏障→结果分类及必要health锁存→退pin/退出admission，返回达到的阶段 | outcome-unknown主文件错误进入WRITE_FAULTED；不自动提交root |

metadata reconstruction 的控制空间来源由持久 geometry 预留，不从尚未验证的持久 FREE 记录任意取块。记录数、索引前缀、根表和工作区在输出 handle 前完整核算。旧锁、指针和线程记录从不作为恢复输入。

pfree 在 commit 前不改变 payload，不写 intrusive remote next，不清零、不 discard，也不允许新分配取得范围。仍被直接 root 引用返回 BUSY；间接应用引用由调用方处理。palloc 已提交但调用方未收到响应可能成为 orphan，恢复允许枚举，不能自动当垃圾删除。

palloc 如需新增主文件 payload view，在 gate 内最终范围验证和 Pending 之后、WAL commit 之前执行 backend 建立，期间不持 broker；建立失败时不提交分配记录。pfree 的物理回收在释放记录持久提交之后执行或退休延后。J、两个 checkpoint 和 H 个 root 槽的固定容量在初始化时已经计费，每次事务消费其内部日志/目录位置，不重复预付整份固定区域；日志槽不足先 checkpoint，目录可用槽不足返回 LIMIT。

commit gate/admission 持有到 runtime 发布或明确故障状态成立，且本次 ticket/pin 责任已经收口或单向移交。root_set 在 gate 内确认目标仍存活、代次匹配且非待释放；pfree 在同一顺序中检查所有直接 root。不能让两者分别使用 gate 外旧快照通过校验。最终校验发现资源计划不够时，不建立业务 Pending、不写WAL，释放gate及ticket后按第2.2节的一次重算上限重新计划；gate内禁止metadata-growth。gate→metadata_pool仅用于不申请新资源的有限最终结算，pool不反向等待gate，pool与broker仍不嵌套。

主文件 persist 也取得 durable admission 与目标控制 pin，checkpoint/close 的排空包括该调用，而不是只数取得 commit gate 的事务。pfree 最终校验发现目标存在该 persist pin 时返回 BUSY，不把目标释放后让旧 flush 继续作用于可能已重用的地址。后端准备、redo、commit、persist 各有限阶段由既有 admission/control 短锁签发许可；WRITE_FAULTED latch 后不签新许可，但此前已签出的单次阶段可以完成实际调用并分类。commit 许可覆盖 commit 记录写入及全部规定屏障，不在写过记录后因下一屏障未获许可就错误推断“未提交”。

health 与本次提交阶段分别判定：从未取得 commit 许可且可证明未提交的请求保留旧 durable ownership；commit 已确认的请求即使此时 arena 已被另一 persist 故障标为 WRITE_FAULTED，仍用预付记录完成 runtime apply/publish，返回本次 OK 并如实反映健康状态；commit 的事实未知才返回 COMMIT_UNKNOWN，冻结未决责任。不得因为全局 health 已坏而撤销已确认的事务，也不继续开启新的持久阶段。详细交错见 [持久设计](persistence.md)。

上述 redo/commit 屏障只属于 RECOVERABLE_METADATA。CLEAN_REOPEN 仍按同一 gate 完成对象/root 的最终验证及 runtime 发布，但不为每次修改写 WAL 或做持久屏障，逻辑 LSN 仅在 runtime 更新；只有正常 close 的 checkpoint 与双super CLEAN 发布形成重开保证。两种持久模式都不允许普通对象入口绕过对应前端。

### 9.2 checkpoint 的 admission

checkpoint 不能仅取得 journal 锁就开始复制 runtime：一个事务可能已完成 commit、正在等待 broker apply。采用以下控制协议：

1. 关闭本持久实例的新 durable 修改 admission；普通其他 arena 不受影响。
2. 等待全部已进入的 durable 操作完成其 runtime publish 或明确故障分类，并收口/移交 ticket 与 pin，包括gate等待者和data persist；Pending 清零到 checkpoint 可接受的稳定集合。等待期间不持控制锁、broker、heap或commit gate。
3. WRITE_FAULTED 则拒绝正常 checkpoint；健康时持commit gate，以已完成 runtime 权威状态和一致 LSN 截点写 inactive checkpoint，并同步数据及校验。
4. 第一个 super 槽发布更高代次、checkpoint引用、截点和新日志epoch，完成屏障；第二个槽镜像相同checkpoint/epoch/state并再提高generation，完成独立屏障。
5. 只有两份super均确认不再依赖旧journal/旧checkpoint后，才能复用其空间并开放 admission。第二份镜像失败不得开放写入或复用旧记录，避免回退到仍有效的旧super却发现其日志已被覆盖。

第一份新 super 已确认、第二份镜像证明无效果地失败时，使用既有持久 control 的 recovery_phase/admission_open/current manifest 记录内部 CHECKPOINT_HELD：health 仍为 READY，但 admission 关闭，新持久修改返回 BUSY。显式 checkpoint 或正常 close 在 quiescence 后独占接管，只继续未完成镜像，不重跑第一半、不另起新 checkpoint；本次无无限内联重试。镜像重试结果未知则转 WRITE_FAULTED，旧 WAL/CP 始终不复用；abort-close 可以退出会话。该内部阶段不新增 health 状态或布局字段。

checkpoint 使用创建时预留的备用区和有界区内 scratch，不为一次满日志临时申请外部堆。日志不够开始下一事务时，先执行该独立 checkpoint，再重算请求；不能在已经写了一半事务后等待挤出日志空间。

避免自等待：已经进入 durable admission 的请求若在 gate 内发现 WAL 满，必须在建立业务 Pending/写WAL前退出 gate、退出本次 admission 并归还其用户 ticket，再请求独立 checkpoint。一个协调者关闭新 admission；其他已进入请求要么完成、要么同样无变更退出，全部排空后协调者取得 commit gate。不得持 commit gate 等待包含自己的 active 数归零，也不得让等待该 gate 的旧请求因 admission 已关闭而永远无法退出。checkpoint 成功后原请求重新进入并从预检开始，配额/地址/目录选择不沿用旧报价。

### 9.3 close 与 abort-close

| 情况 | 预检 | 执行与结果 |
| --- | --- | --- |
| 正常 close | 应用quiescent；全部ctx detach，session mappings/leases和临时资源已显式释放，无进行中调用 | 关闭admission，收敛已提交状态；CLEAN_REOPEN完成新checkpoint与clean发布，RECOVERABLE按格式完成关闭标记；允许持久live对象和roots |
| close 前置不足 | 任意未停用资源/ctx/事务 | BUSY，尚未开始clean提交，不自动帮调用方撤销活跃session资源 |
| close 持久阶段失败 | 结果可确定或不确定按提交协议区分 | 未确认clean绝不报告clean close成功；未知进入WRITE_FAULTED，要求abort-close/恢复 |
| abort-close | 应用已quiescent，明确放弃本次会话继续写入 | 不checkpoint、不覆盖super或日志，不做“修复性flush”；只清理session/runtime native资源和lease，下一次open按磁盘事实恢复 |
| runtime清理失败 | 后端handle/view仍未确认释放 | cleanup token 接管可定位控制岛与记录，返回错误；原arena handle禁止继续调用，只能cleanup retry，不重复关闭已释放资源 |

owner lease 和进程内文件登记在不再有本会话运行时写入者后才撤销，控制岛最后解除。abort-close 不撤销已经持久的事务，也不把未知 pfree 声明为失败无效果；重新打开后查询 allocation identity 决定其状态。

## 10. 失败分类与审查矩阵

| 失败位置 | 地址状态 | 配额处理 | 用户可继续做什么 |
| --- | --- | --- | --- |
| 输入/输出预检 | 原样 | 未预留或已全归还 | 修正参数/报告buffer后重试 |
| quota/record不足 | 原样，metadata自身独立增长可能已完成 | 归还用户ticket；增长出的合法库存留在arena | 维护、稍后重试，或以另一创建配置重建实例；首版不在线调额度 |
| Pending后OS无效果/完整恢复 | 回原owner/FREE | 新资源确认释放后归还 | 按原对象合同继续 |
| OS部分改变且无法完整恢复 | 各片段确定最终状态，未知片段QUARANTINED | 实际及未知资源继续used | 查询报告、只访问被明确保留的范围，显式维护或discard |
| VOLATILE free后物理清理失败 | 用户对象已失效，retired/quarantined backing仍受控 | 保留used，不能把旧对象重新计live | 不重试free；通过维护/诊断处理 |
| 持久提交结果不明 | 未决对象不可复用，实例WRITE_FAULTED | 不释放不确定占用 | 只读诊断、abort-close、重开恢复后查询 |

下列审查项构成本文件的完成记录；实现测试以后按 [验证矩阵](validation.md)执行：

- P-01：每条公开变更路径均有 owner、成功点、quota ticket 和最终资源责任；估算不冒充预留。
- P-02：任何 OS 前已预留 split/rollback/quarantine 和 caller report；部分结果不被 BUFFER_TOO_SMALL 掩盖。
- P-03：普通 free 与退休队列无新增记录依赖；页 unmap 和持久 pfree 的错误合同单独保持。
- P-04：S-01 两组 emergency 仅用于单单元增长；不会跨 view 扩大原语或向对象 radix 递归；持久 R 的 RESERVED/ACTIVE/故障状态始终留在 METADATA 域且 1002 不重复收费。
- P-05：central/shared/broker 不嵌套，metadata_pool 仅叶子锁；commit gate→broker为明确单向例外，I/O不持broker；首版无用户callback，跨锁有稳定pin/状态。
- P-06：partial survivor、洞复用、move 部分失败和 stale query cursor 均不能给旧 handle 新权限；Pending 的 begin/end 各推进代次，分页遇任何 Pending 返回 BUSY，交错事务不乱序写回 revision，合法释放/清理在计数饱和时继续且分页统一 STALE_CURSOR，持久排序计数不适用饱和豁免。
- P-07：RT binding、slot状态、owner与teardown生命周期闭合，free不引入共享额度更新或动态登记。
- P-08：checkpoint 排空commit后尚未apply的操作，WRITE_FAULTED禁止覆盖恢复证据，close不要求持久live归零。
- P-09：所有后端未确认释放的资源继续计used；多实例故障和额度互不传播。
- P-10：上述推导不替代Windows/POSIX失败语义、原子代码生成、实际sizeof与持久屏障的实施验收；验证不通过即禁用对应能力或按变更流程修订设计。
- P-11：Pending 只有同步调用执行者可完成；pin不等于字段快照/范围授权；终态隔离退出active集合但保留全部清理证据，维护无超时强抢或无界故障循环。
- P-12：每个 native 子动作先计划再执行，错误与effect分离；逆向补偿再失败只发布实际结果/隔离，不撤销已公布逻辑成功或已确认commit；native身份失证时不盲重试同地址/句柄。
