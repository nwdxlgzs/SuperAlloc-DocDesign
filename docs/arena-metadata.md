# Arena 内自举、元数据与地址索引

本文规定arena内控制数据的布局原则、增长方式和预算契约，服务于[architecture.md](architecture.md)的A-01至A-20。本文仅为设计，所有字节预算和结构尺寸均需实现后核算及验证。

S-01已补充[逐字段布局](metadata-layout.md)与[容量预算](metadata-budget.md)。前者给出32/64位模型偏移，后者给出64KiB控制岛的条件性算术闭合；两者均不是编译或平台实测。

## 1. 计量与容量

统一使用二进制容量：16MiB 为 `2^24` 字节，4TiB 为 `2^42` 字节。arena capacity 包含控制区、分配有效负载、缓存、空闲区和隔离区；不是可全部交给用户的 payload 大小。

定义：

| 符号 | 含义 |
| --- | --- |
| `B` | arena 基地址，至少按 P 对齐；owned backend 可要求更高保留粒度 |
| `C` | arena 总逻辑容量 |
| `P` | 当前进程的系统实际页大小 |
| `T` | 地址索引 tile 大小，设计基线 `max(64KiB, P)` |
| `S` | 普通 slab 大小，首版设计基线 `max(64KiB, P)` |
| `M0` | 初始 bootstrap 及最初元数据的提交预算 |
| `M_limit` | 用户设置的全部 runtime 元数据预算上限 |

P、T、S 需要是可由平台支持的整数页倍数；平台不满足基线时创建必须拒绝或使用显式验证过的 geometry，不能按 4KiB 写死。range offset 和持久长度使用 u64，运行时每次换算为指针或 `size_t` 前执行上界及溢出检查。地址校验与相对 offset 计算使用已检查的 `uintptr_t` 数值运算，不依赖对无关 C++ 对象指针做减法或大小比较。

64 位支持 4TiB 仅是容量契约上限；保留成功依赖地址空间。32 位 Android 的单实例上限为向下按 P 对齐的 `PTRDIFF_MAX`，约 2GiB；多实例合计仍受实际进程虚拟地址限制，创建允许返回地址保留失败。不能把接近该上限的地址保留成功当成常见设备保证。

## 2. 自举含义与允许的外部状态

所有可增长、用于管理实例的数据都位于实例的 arena 内，包括范围节点、线程登记、slab descriptor、remote-free 控制数据、radix 节点、事务记录和统计存储。不允许元数据不足时回退到系统 malloc 或另一个默认 arena。

允许以下不随分配量增长的区外状态：

- 固定大小的 API 入口与平台适配代码、BSS 初始化锚点和锁。
- 可选 TLS 链表头；链表节点在对应 arena 内。
- 调用栈上的有界临时变量和 API out 参数。
- 操作系统拥有的内核页表、文件句柄及映射对象；这些仍应纳入句柄和 view 配额，但不能声称内核控制数据位于用户 arena 内。

进程内持久文件登记使用实例 runtime 控制块中的 intrusive 节点；区外只有固定头和锁。创建失败时不得把指向已解除映射控制块的登记节点遗留在全局链表。

C++20 实现使用原位构造、显式 pool 与 intrusive 结构；不能在控制路径直接使用会调用默认 allocator 的容器、字符串或异常构造作为隐式回退。平台适配函数与诊断路径也需审计分配依赖，错误报告采用固定 status 与调用方提供的输出缓冲，不为了报告 OOM 再申请普通 heap。

## 3. 控制区布局

普通 owned arena 的逻辑布局为：

```text
[ bootstrap/control island ][ ordinary metadata extents ... ][ payload/free/mapping extents ... ]
```

图中的后两部分可以交错，控制区不要求单独连续增长。bootstrap 位置和长度固定；后续 metadata extent 由 broker 分配并永久或阶段性标记为 `METADATA`。固定 map 请求不能覆盖任何 metadata extent。

bootstrap 至少容纳：

- arena header、geometry、incarnation、模式和状态。
- broker 锁、范围树根、空闲尺寸档头及非空 bitmap。
- 内部元数据池的根、统计计数和预算状态，以及无 ctx 入口所需的 shared fallback heap 根与锁。
- 稀疏索引根、首批范围节点、最少事务记录。
- 不向一般操作出借的紧急 descriptor/索引储备。

首版预算目标为常见 `P <= 64KiB` 平台上，最小普通 VOLATILE 配置的 `M0 <= 64KiB`；16MiB arena 中该值约占 0.39%。该预算不包含额外登记线程、RT pool、持久日志与 checkpoint，也不是运行全过程的元数据上限。这是需测量的验收目标，不能把它作为未核算结构尺寸的事实。若某平台不能满足，必须形成明确的预算变更记录，而不是偷偷向区外申请控制内存。

S-01将最小控制岛设计为64KiB，其中含一个shared heap及一份待使用ctx+heap库存；16MiB、64位4KiB页模型的已安排结构/库存为23424B，其余42112B仍属于metadata尾部。额外线程和后续增长另收费；平台原子、锁槽与实际sizeof仍需验证。

可恢复 arena 将持久 superblock/日志/账本与 runtime control island 分开。runtime island 在打开时重建，其地址范围由持久几何保留，但内容不当作持久分配真相。控制岛采用何种 backing 由平台与持久布局规定；不能让临时锁或进程指针因文件映射而意外成为恢复输入。

## 4. 元数据对象族

元数据分配器使用按固定尺寸族管理的内部 pool，避免控制路径依赖通用小对象前端：

| 对象族 | 生命周期与持有者 |
| --- | --- |
| Extent descriptor | 从 broker 分割生成，到兼容合并回收；持有自己的树/链节点 |
| Slab/large descriptor | 从对象 span 初始化到最后合法对象操作结束；不置于可重绑 payload 中 |
| Mapping descriptor | mapping handle 生命周期；记录 view profile、backing 身份及边界 |
| Radix node / PageTile | 按需建立；首版仅在 quiescent 维护阶段回收目录节点 |
| Thread context / owner record | 登记到退出移交完成；owner token 生命周期内不复用 |
| Range transaction | 从Pending建立至发布/完整回滚；终态隔离X仍保留关联范围及native责任证据，直至独立清理收口 |
| RT pool/slot 状态 | prepare 到 quiescent teardown；运行期不增长 |

持久账本记录不直接复用以上 C++ 结构布局。文件格式需要固定字段宽度、版本和校验，并与当前编译器 padding、原生指针宽度解耦。

descriptor 应采用 offset 或区内指针中的一种明确形式：runtime 原生指针用于效率，持久记录只用 offset64。两者不能通过原样 memcpy 相互序列化。

## 5. 稀疏地址索引

小对象 free 不能在范围红黑树中搜索。采用以 arena 相对 offset 定位的固定深度稀疏 radix：

1. 先检查指针是否落在 `[B, B + C)`，再计算 `offset = ptr - B`，所有算术均检查溢出。
2. `tile_index = offset / T`。radix 以 8 位为一级，分支数设计基线为 256；根到 tile 的最大深度由 C/T 在创建时固定。
3. radix 的 tile 条目指向一个 `PageTile`；其中有 `T/P` 个页级条目，使用 `(offset % T) / P` 选择稳定 descriptor。
4. slab descriptor 再用 slot geometry 判断对象边界与 size class；large descriptor 记录唯一有效起始位置。诊断模式可以增加分配位图检查，但不承诺廉价检测所有悬垂裸指针。

PageTile 的额外一层不能省略成“一 tile 只有一个 descriptor”：页粒度大对象和页映射可能共享同一 64KiB tile 的不同页。统一 slab 可在后续加入 uniform tile 编码优化，首版以显式页条目保证语义。

页条目只为合法对象起点需要的查找建立：slab和RT覆盖全部可能起点页；大对象仅登记合法返回指针所在页，内部指针不属于free/usable_size/realloc契约。普通空闲、metadata及仅由 mapping handle 访问的用户映射不建立完整对象索引。控制面任意地址所属查询使用受保护范围树。新 descriptor 初始化完毕后以原生指针宽度的 release 发布，读取方 acquire；32 位不要求 64 位双字段原子发布。

新增目录路径由 broker 的慢路径发布阶段串行化，节点在发布前原位初始化，子指针采用同样的原生宽发布规则。竞争者必须重新使用已发布路径并归还多余预留节点，不能互相覆盖根或丢失另一范围的 PageTile。节点只增不在线撤销的首版规则使合法 free 遍历不需要 hazard pointer；quiescent 回收时另行验证没有路径读取者。

4TiB、64KiB tile 时有 `2^26` 个潜在 tile，8 位 radix 最多需要 4 级。创建只建立实际需要的根/路径，不分配覆盖全部 tile 的数组。节点开销随涉及的目录分支和已使用 tile 增长，不随保留容量直接线性增长。

完全使用时页级条目仍有实际成本。例如 64 位、4KiB 页、每页 8B descriptor 指针的原始条目成本为 payload 覆盖范围的约 0.195%，未计 radix、descriptor 和 pool 碎片。该示例用于容量核算，不是总体开销承诺。

RT prepare 为所属 pool 完成全部路径、PageTile 和 descriptor 建立；运行阶段查找深度固定且不触发缺页准备工作。原生指针 atomic 的 lock-free 能力须在目标平台验证，不满足时不能宣称该平台提供所定义的 RT profile。

## 6. 自举创建顺序

自建 arena 的建立过程：

1. 校验 C、P、模式与配置，探测最小平台 capability，并计算 bootstrap geometry。
2. 保留C字节连续地址；仅提交bootstrap所需区域。若提交失败，清理已取得资源；清理仍失败时按操作协议以固定caller cleanup token交付剩余责任，不输出可分配arena。
3. 在 bootstrap 中原位建立 header、元数据 pool 根、首批 descriptor 和空闲索引，不调用任何普通分配 API。
4. 普通VOLATILE用已有节点登记bootstrap为METADATA，其余为FREE；根据后端边界可形成多个free extent，不预建逐页描述符。持久模式按固定几何把整个R和格式控制区排除于用户FREE，其专用流程见持久格式。
5. 建立空的对象索引root、首条路径库存和紧急储备；metadata范围只由broker与pool管理，不为自身递归建立对象PageTile。持久模式此时执行 lease、格式校验或恢复要求的专用顺序。
6. 所有最低预算、后端和恢复条件满足后才发布 arena handle。失败不得向用户返回半初始化实例。

借用 buffer 采用同一原位 bootstrap，但跳过拥有者的 reserve/commit 操作。调用者承诺 buffer 在 arena 生命周期中持续可写、独占子分配且不会自行解除映射。初始化失败可以改写已获授权的控制区字节；文档不承诺内容回滚，但不得解除调用者映射。

持久 open 的 lease 与恢复具体顺序由持久设计定义，不能为了复用普通创建流程先覆盖文件 superblock。

open 还必须根据持久分配记录数量、选定 geometry 与当前 runtime 配额，核对重建索引和 descriptor 所需空间。重建预算不足应在发布 handle 前明确失败，不能因为上一次运行曾成功创建该 store 就假设任何较低预算都能打开。checkpoint 的备用空间与 runtime reconstruction 空间分别计入预算。

## 7. 无递归元数据扩容

元数据扩容是独立维护事务，由 arena 内的 metadata-growth gate 串行。该 gate 不使对象热路径获取锁；它只约束控制区增长与需要增长的慢操作。

采用资源 credit 预检：每类操作先计算最大需要的节点、索引页、事务记录和恢复记录。一般操作只能消耗普通可用池，不能耗尽紧急储备。

扩容顺序：

1. 检查 `M_limit` 和可用地址；决定需要增加的内部 pool 族及页数。
2. 从紧急储备取得描述本次 metadata extent、最多必要分割和事务恢复的全部节点。若储备不足，不能开始扩容。
3. 在 broker 锁内选择空间并标记 Pending；锁外提交或映射 metadata backing。
4. 原位初始化新增 pool 和其自描述头，不能调用通用 malloc。
5. 保持Pending/pin，在broker锁外先于metadata pool叶子锁结算已确定的资源；再进broker发布范围为METADATA并撤Pending，解锁退Pending名额。之后在pool锁下先补足紧急储备，再把剩余新增节点放入普通池；growth gate保持本次扩容串行，broker与pool不嵌套。
6. 扩容失败恢复旧范围；恢复无法确认时，以预留记录完成隔离。若紧急预算因此不能恢复最低水位，拒绝后续需要扩容的操作，保留已分配对象的正常管理能力。

紧急节点数量不能拍脑袋固定为“几个”：实现前为每个会修改范围的原语列出最大分割数、回滚节点数及隔离描述数，再推导紧急水位。多 view 操作按实际 view 数预留事务数组并接受配额限制，不能在操作执行到一半时申请更多记录。

S-01把owned应急增长限制到一个无活跃backing的单元；borrowed例外沿用不可变的caller buffer BackendUnit，仅重划逻辑所有权。每组保守预留2个Extent、1个BackendUnit、1个RangeTxn和1个TxnStep；初始两组，64/32位净记录成本为1248/960B，已经计入bootstrap。无法补足下一组完整额度时停止增长，已有对象仍可通过已预留的归还结构free。多view操作使用独立预算，不消耗该专用储备。

持久runtime R从创建起整体属于METADATA，未启用部分为RESERVED子态，不进入用户FREE索引。相同增长原语可以在R内拆分、启用一个无backing单元；左右仍保留METADATA/RESERVED，失败不把它归给用户。R地址额度已预付，增长只增加实际后端与控制资源，仍使用上述应急上界。

完整 extent 释放复用其原有节点；合并只回收节点，因此合法普通对象释放无需新增 extent descriptor。页映射的部分解除/部分保护需要先拆分，可能因预检元数据不足而返回错误，但不能已经解除部分映射后才发现无法记录结果。

元数据全生命周期可以出现 pool 内部碎片。普通模式允许在 quiescent 维护阶段把完全空的 metadata extent 归还 broker；bootstrap、仍有引用的 descriptor 页和紧急储备不得被归还。首版不承诺在线压缩或移动活 descriptor。

## 8. 预算、配额与统计

`M_limit` 是显式配置，最低允许值由 bootstrap 和选定模式的预检计算；不能在运行时悄悄突破。配置字段及兼容规则以 [API 契约](api-contracts.md) 为准，本文不另设互相冲突的默认值。

预算应分项暴露：

- bootstrap、普通 descriptor pool、radix/PageTile、线程登记。
- 普通缓存与空 slab；这是 payload ownership，不能算作可立即提供给映射的 free。
- Pending/恢复工作区、紧急储备和隔离范围。
- 持久日志/账本容量及 runtime reconstruction 空间，分别核算。
- RT payload、slot 状态、预建索引与登记记录，分别核算。

RT 的每 slot 原子状态以 4B 为设计基线，16B 对象的原始状态开销约为 25%；实际还需 descriptor 和索引。不得默认对每个状态填充至 64B，也不能把该开销隐藏在 payload 配额外。是否使用原生 4B atomic 及是否 lock-free 均由平台验收确认。

统计至少区分：保留虚拟容量、broker 所有权字节、后端提交字节、元数据已用/已保留、缓存持有、可分配总空闲、最大已知连续空闲、隔离字节。最大连续范围统计只能按兼容后端边界解释；不能把尚需系统调用修复的洞直接算作即刻可分配范围。

## 9. 线程、descriptor 与回收安全

ctx 存储与 slab descriptor 分离。线程退出先把 owner 管理权按并发协议移交，仍有活对象的 slab descriptor 继续存在。原生宽度 owner token 在单个 arena 生命周期不复用，溢出时拒绝登记；不使用“ctx 地址相同”判断当前线程拥有旧 slab。

有效对象释放与 owner 回收之间遵守以下关系：

- 未发布的远程 free 仍计为 live；owner 不能因为尚未观察到该请求而提前把对应对象当作已释放。
- 发布者执行 release 后不再读取或写入 descriptor；owner acquire 后才把对象从 live 移到可复用集合。
- descriptor 释放要求无 live、无待处理发布、无其他受保护引用。扫描统计和调试查询必须持维护保护或读取快照，不能把自己伪装成仍有 live object 的合法访问。
- PageTile 槽位可在无合法对象访问后清空并供新 descriptor 发布；目录节点本身只在 quiescent 维护回收。

RT 使用每 slot 返回状态，pool descriptor 至 quiescent teardown 固定存活。owner 的有界扫描只影响重新分配进度，不影响合法 free 能否成功发布。

## 10. 必须验证的证据

实现阶段应产出以下证据，而不是仅运行几个成功案例：

1. 各平台与构建模式的 `sizeof` 清单，16MiB 普通配置的 bootstrap 分项账，所有模式的最低配置拒绝理由。
2. 4TiB 仅保留且少量分配时的创建时间、元数据提交量和节点数量，证明没有全域 dense 初始化。
3. 所有 split、merge、metadata-growth、目录建立和事务工作区申请的失败注入，核对 A-06 与 A-08。
4. 普通 pool 耗尽后，完整对象释放仍可登记；部分 page 操作在任何可见变更前拒绝不足预算。
5. 元数据扩容自身发生后端失败和恢复失败时，紧急记录足够描述最终状态，且无区外动态堆回退。
6. 两个 arena 使用相同地址偏移、不同 geometry 和不同预算时，ctx、descriptor、文件身份登记及统计不串扰。
7. 32 位 Android 的容量边界、全部加法/乘法/round-up 溢出、owner token 耗尽，以及非 lock-free u64 构建路径。
8. 随机线程退出、远程释放、span 归还与地址复用下，索引不引用已回收 descriptor；目录 quiescent 回收可被验证。
9. 持久重新打开时 runtime 元数据从持久事实重建，旧进程指针和锁字节从不参与恢复决定。

以上检查完成前，64KiB bootstrap、slab 尺寸、radix 分支和 RT 状态成本均保留“设计基线/验收目标”标记。
