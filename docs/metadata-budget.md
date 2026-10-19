# S-01：自举与元数据容量预算

状态：**设计算术核对完成，尚无sizeof或实机测量**。布局依据为[metadata-layout.md](metadata-layout.md)。以下推导回答“这些结构能否在给定区域内放下”，不把管理开销模型当作吞吐、RSS或平台兼容结果。

## 1. 容量分账与计量公式

C是整个arena容量，包含metadata、payload、缓存、RT、持久工作区、空闲和隔离区。对任一时刻：

`C = M_owned + H_owned + L_owned + Map_owned + RT_owned + Free + Pending + Quarantined`

这些是互斥的地址所有权集合。pending期间原owner的字节不能同时计入最终owner合计；backend charge、resident和文件磁盘块是另外的资源轴，不能叠加到上式当作额外地址。

持久模式M_owned包含整个R及super/WAL/checkpoint等固定控制区域。R内部的Pending/Quarantined是METADATA子态，仍只计入M_owned，作为诊断明细时不能再加到上式顶层Pending/Quarantined。runtime_metadata option1002单独只计R；日志/CP对应自己的额度，地址和配额维度不混同。

必须分开报告：

- **M0**：创建成功时已经为bootstrap保留/启用的metadata区域。本基线为64KiB，包含未使用尾部。
- **L0**：bootstrap内部固定控制块、记录库存、头部和对齐已经安排的字节。本次可以逐项精确计算。
- **M_reserved**：当前全部metadata extents的地址容量，包括内部未用空间，受M_limit约束。
- **M_live_records**：已被在用控制记录消耗的槽位；预制但空闲的库存不能算作live，却仍占M_reserved。
- **M_emergency**：不能出借给普通操作的应急记录，已包含在M_reserved内，不能再从C扣第二次。
- payload内部浪费、线程私有空slab、metadata pool内部空槽和OS retained commitment分别统计。

一般metadata family chunk的成本为 `align64(align(header, record_alignment) + n × stride)`。增长为OS页/单元时还需整体按P和后端G取整；不能对每条小记录单独按页取整，也不能忽略每个独立extent的最终取整。

## 2. 64KiB自举基线

### 2.1 覆盖条件

普通VOLATILE、未启用RT/持久格式/精确请求统计，P=4KiB或16KiB，后端允许在一个64KiB控制岛中建立原位状态。控制岛也按已验证的G对齐；G不兼容64KiB或native控制结构超出预算时必须先调整设计/拒绝该配置，不伪称已满足。

创建预建一个shared heap，并预留一份ctx+heap记录作为首次显式登记库存；这不等于自动登记线程。对象payload尚未分配。radix root存在，但metadata本身不占对象PageTile；最大容量情况下另备首条路径需要的D-1个node。

### 2.2 初始库存

- Extent普通库存32条，应急4条；初始2条描述控制岛与余下FREE。
- SlabDesc 8条，LargeDesc 4条，MappingDesc 4条。
- BackendUnit普通8条，应急2条；bootstrap active backing使用普通库存。未替换的FREE placeholder/PROT_NONE reservation以Extent边界和状态描述，backend指针为空，不另建重复BackendUnit。
- RangeTxn普通4条，应急2条；TxnStep普通16条，应急2条。
- ThreadCtx 1条，Heap 2条（shared已用，另1条待登记）。
- RadixNode共D条（root 1条，路径库存D-1条）；PageTile 8条。
- 共11个有初始记录的family，每个有自己的PoolChunk头。RT、持久日志和额外线程记录初始不分配。

这些是**自举库存**，不是线程数、映射数或同时Pending数量的公开硬上限。库存不足可以在额度内从控制岛尾部或新metadata extent扩容；实际默认额度由S-02冻结。

### 2.3 16MiB布局逐项表

字节均为十进制B。偏移是64位、4KiB页模型中的arena-relative起点；各行已经包括必要的chunk头与64B边界取整。表不重复收费应急库存。

| 控制块/库存 | 数量依据 | 偏移：64位4KiB | 占用：64位4KiB | 占用：64位16KiB | 占用：32位4KiB |
| --- | --- | --- | --- | --- | --- |
| ArenaHeader | 固定控制块 | 0 | 256 | 256 | 192 |
| ControlLocks | 固定控制块 | 256 | 320 | 320 | 320 |
| Limits | 固定控制块 | 576 | 448 | 448 | 448 |
| Stats | 固定控制块 | 1024 | 192 | 192 | 192 |
| FreeIndex | 固定控制块 | 1216 | 2112 | 2112 | 1088 |
| MetaPoolControl | 固定控制块 | 3328 | 1088 | 1088 | 832 |
| BackendControl | 固定控制块 | 4416 | 256 | 256 | 256 |
| SizeClassGeometry | 固定控制块 | 4672 | 384 | 384 | 384 |
| ExtentDesc | 36 × 144B，另计chunk头/对齐 | 5056 | 5248 | 5248 | 3520 |
| SlabDesc | 8 × 192B，另计chunk头/对齐 | 10304 | 1600 | 1600 | 1600 |
| LargeDesc | 4 × 96B，另计chunk头/对齐 | 11904 | 448 | 448 | 384 |
| MappingDesc | 4 × 112B，另计chunk头/对齐 | 12352 | 512 | 512 | 448 |
| BackendUnit | 10 × 112B，另计chunk头/对齐 | 12864 | 1216 | 1216 | 1024 |
| ThreadCtx | 1 × 128B，另计chunk头/对齐 | 14080 | 192 | 192 | 192 |
| Heap | 2 × 1536B，另计chunk头/对齐 | 14272 | 3136 | 3136 | 1728 |
| RangeTxn | 6 × 144B，另计chunk头/对齐 | 17408 | 960 | 960 | 768 |
| TxnStep | 18 × 80B，另计chunk头/对齐 | 18368 | 1536 | 1536 | 1536 |
| RadixNode | 1 × 2080B，另计chunk头/对齐 | 19904 | 2176 | 2176 | 1152 |
| PageTile | 8 × 160B，另计chunk头/对齐 | 22080 | 1344 | 576 | 832 |
| L0合计 | 记录与控制块 | — | **23424** | **22656** | **16896** |
| 控制岛未分配尾部 | 留给metadata后续切块 | — | **42112** | **42880** | **48640** |
| M0 | 一次64KiB控制岛 | 0 | **65536** | **65536** | **65536** |

64位4KiB模型：23424 + 42112 = 65536。32位和16KiB模型也分别闭合；不同字段对齐并未被“指针减半”粗略替代。

16MiB中M0占 **0.390625%**；创建后其余地址为15.9375MiB。控制岛尾部不是用户payload，后续切出metadata记录时M_live/库存变化，但M_reserved不会再次增加。该账证明按当前布局模型可以在64KiB中自举；不替代真实编译器和平台适配验收。

首个普通64KiB slab至少需要已有的SlabDesc、Extent拆分额度、BackendUnit和PageTile。16MiB的D=1只使用root；4TiB的D=4最多使用已备的3个非root节点。库存足以完成此路径，payload提交仍可因OS/配额失败。

## 3. 应急额度从操作边界推导

本设计把metadata自扩容限制为一个明确原语：在**单个已知可替换的FREE、backend为空范围**中取得一个完整后端单元，标记为METADATA；不跨活跃view、不建立用户对象索引、不在中途再申请控制记录。

上述backend为空条件针对owned增长。借用buffer的FREE可以指向同一不变的borrowed BackendUnit；对应操作只是对已映射可写字节重划内部所有权，不执行OS替换，也不增加backend charge。下面B=1控制信用仍可保守预留后原样归还，不能由地址/对齐推断OS权限。

持久模式是另一明确特例：R已整体归METADATA，未启用部分为RESERVED子态且backend为空。增长在R内拆出单元，左右仍为META_RESERVED，已知回滚回原RESERVED，未知结果隔离；不经过用户FREE。记录上界不变，但1002地址额度已按R预付，启用时仅新增backend/view/Pending等费用，不能重复收费。详见[操作协议](operation-protocols.md)和[持久格式](persistent-format.md)。

| 所需资源 | 单次增长最坏需要 | 理由 |
| --- | --- | --- |
| 额外ExtentDesc | 2 | 原记录复用为目标，最多新建左右FREE两片；失败时同一组三片改状态，不增加切分 |
| BackendUnit | 1 | 目标新匿名单元；左右纯reservation无需额外active backing记录 |
| RangeTxn | 1 | 跨Pending、OS操作和发布/隔离维持身份 |
| TxnStep | 1 | 后端单元整体记录结果；不确定时隔离整个目标，不在线细分其未知内部状态 |
| Radix/PageTile | 0 | metadata不接入用户对象free索引 |
| PoolChunk头 | 从新单元内原位建立 | 成功前尚未发布；失败不依赖读取该单元的头部 |

因此一组控制额度：

- 64位：2×144 + 112 + 144 + 80 = **624B**。
- 32位：2×96 + 96 + 112 + 80 = **480B**。

初始预留两组：1248/960B，已经包括在表内Extent/Backend/Txn/Step库存。metadata-growth gate串行化增长；第二组是一次故障长期占用控制记录后的后备，不允许两个增长同时消耗完整储备。

成功增长时，在新metadata单元内先建立补充这四类记录所需的chunk，再给普通请求分配目标family。不能只扩充调用者请求的Radix family，却忘记把描述此次增长而永久占用的应急Extent补回。最小新单元必须容纳缺口记录、各family头和本次目标记录；检查后再执行OS操作。

失败分支：

1. 确认无效果/完整回滚：还回所借额度。
2. 目标隔离且左右分裂仍存在：保留这些记录，不把它们当作已归还。
3. 剩余库存不足一组时：停止新的metadata-growth；继续服务已有记录支持的操作与合法free，明确返回METADATA_LIMIT/故障状态。
4. 没有满足本原语条件的FREE单元：允许先由独立维护清理retained backing；不得扩大应急原语去跨view重映射。不满足就失败。

两组额度不保证无限次故障后仍能增长；它保证**进入任何一次增长之前，已有额度足以描述该次所有结果**。应急储备不负责多view用户操作，那些操作使用自己的完整预检额度。

## 4. 正常增长和请求成本

### 4.1 对象slab与线程

一个新64KiB slab，在新tile、新backend、无需额外radix分支的简化情况下，64位记录增量为：

`Extent144 + Slab192 + Backend112 + PageTile160 = 608B`（4KiB页）。

16KiB页时PageTile为64B，总计512B；32位4KiB时为96+192+96+96=480B。这是**记录量**，不是实际metadata extent提交量；地址拆分额外记录、共享radix、chunk头、初始库存抵扣和OS取整另计。

每登记ctx+heap为1664/960B；新线程并不立刻获得44个slab。若每个64位线程真的激活44类各一个64KiB slab，私有payload capacity是 **2.75MiB/线程**，远大于ctx记录本身。[S-02](configuration.md)通过private span预算及完整移交/共享路径限制这种保留，不能只限制ctx数量。

未启用诊断时不为每个小对象保存requested_size：

- calloc仍按调用时请求长度清零。
- realloc可复制 `min(old_usable,new_requested)` 个可用字节，满足至少保留旧请求有效部分的合同；不能读到slot之外。
- usable size与requested live是不同指标；没有逐对象追踪时requested live标为不可用，不报告虚构精确值。
- 可选精确请求表为每slot u16（小对象最大32768B），另按16B取整。16B类一slab有4096个slot，该表8192B，相当于payload的12.5%；诊断分配位图再加512B。启用时预检并从metadata计费。
- 大对象已有requested_size，持久对象目录也需精确长度；不能把普通小对象的省略策略直接套到持久格式。

可用字节、poison/unpoison和realloc复制范围的实现必须一致；启用精确requested追踪时可以只复制旧requested有效范围。

### 4.2 多view事务

k表示此次变更涉及的独立backend步骤数；大范围先形成步骤计划。首版保守预留上界：

- 新Extent记录至多3k+2：覆盖边界切分、旧状态保留和逐步隔离身份；只有明确证明可复用的原记录才能抵扣。
- Backend记录至多2k+1：新/替代单元及共享section父记录的保守额度；旧记录被pin并不因此再复制。
- TxnStep k条、RangeTxn 1条、新MappingDesc最多1条。
- 可能需要的索引记录、OS handle配额和用户内容备份另计；本模型不宣称持有Step就是备份了内容。
- 单步不能产生未预先计数的任意数量子区间；后端若不满足，必须先拆为更细步骤再计算k。

64位仅上述控制记录的保守公式：

`M_tx(k) = (3k+2)×144 + (2k+1)×112 + k×80 + 144 + 112 = 736k + 656`。

因此“4096个细粒度views”的配额不意味着metadata只需要几KB；该上界为3,015,312B，未计已持有旧记录和页取整。它是失败前完整预留的保守工作集，不是永久开销或对每个map强行实际分配这么多记录；backend在完整证明后可按计划抵扣无需的新记录，S-03审查其资源向量。

普通/可拆映射的资源预估必须同时返回地址、记录、views、handles和retained backing，不只检查请求payload是否小于C。

## 5. 4TiB与稀疏索引

T=64KiB时，4TiB有2^26个潜在tile、D=4。创建只预建root与首条路径库存共4个RadixNode。相比16MiB模型增加3×2080B记录，同时原Radix chunk尾部的32B填充消失：该chunk从align64(64+2080)=2176B变为align64(64+4×2080)=8384B。因此控制岛L0为23424−2176+8384= **29632B**，仍小于65536B；不创建2^26个PageTile。

对于n个实际需要索引的tile，内部节点是已涉及的前缀集合。4TiB时安全上界：

`N_radix <= 1 + min(n,4) + min(n,1024) + min(n,262144)`。

单个孤立tile的一条完整路径最多4×2080+160=8480B（64位4KiB）；以后相邻tile共享前缀。旧路径只在quiescent回收意味着索引成本也取决于历次涉及的tile前缀，而不只取决于当下live payload；长期稀疏churn需要metadata上限与显式维护。

| 使用情形 | 对象快速索引需求 |
| --- | --- |
| 只reserve 4TiB | root与小库存，没有全域PageTile |
| 单个超大对象 | 返回指针所在页/tile的一条路径；全范围所有权仍由Extent管理 |
| 大文件mmap | 由mapping handle访问，默认不建立对象free目录 |
| 普通slab密集铺满 | 每个被使用tile有PageTile，slab所有合法起点页可定位 |
| RT池 | prepare为全部合法slot起点页建立路径，运行期无目录增长 |

为展示密集小对象的上界，假想4TiB全部作为64KiB slab覆盖（真实arena还要扣metadata，**不是可交付payload承诺**）：

- tile/slab数：67,108,864。
- radix节点：1+4+1024+262144=263,173；2080B/节点，共547,399,840B，约0.509806GiB。
- PageTile：160B/tile，共10GiB，其中8GiB是页指针、2GiB是tile头。
- SlabDesc：192B/slab，共12GiB；ExtentDesc144B，共9GiB；假定每slab一个BackendUnit112B，共7GiB。
- 合计约 **38.509806GiB（相对假想payload约0.940181%）**，未计额外碎片、chunk头、空库存、保护切分和debug表。实际slab数受容量方程约束，不能同时宣称可用payload为4TiB。

此例同时说明稀疏初始化与满载管理开销是不同问题；仅报告创建很省内存不足以证明满载空间效率。

## 6. RT预算

固定N个slot的状态数组为4N B，没有归还队列条目或队列扩容。记录成本按布局文件另加：

`M_rt_records = RTPool + class_count×RTClass + binding_count×RTBinding + 4N + index增量 + range/backend记录`。

控制记录64位为RTPool192B、RTClass80B、RTBinding80B；32位为128/80/64B。绑定成本按允许归还的线程与pool关系数计算，不是每次free动态产生。

每类payload范围按P对齐，slot数量N_i与stride_i预定；padding为alignP(N_i×stride_i)-N_i×stride_i。状态数组、class表和binding表也有metadata对齐/分块费用。pool的用户额度不能把这些成本藏在另一个实例。

例：65536个16B slot需要1MiB payload，单是状态就需要256KiB；其余记录/索引另计。因此“1MiB RT总预算”绝不能承诺65536个slot。忽略其他开销时也最多floor(1048576/20)=52428个，真正prepare可接受值更低。

free只写自身a32；不能为了精确实时统计增加跨线程全局计数更新。owner计数的是已观察并接收的归还，不能等同于墙钟时刻所有producer已完成的free。

## 7. 持久模式与本次证明边界

64KiB自举证明仅对应普通VOLATILE。持久实例还需在同一个C内独立划出：

- runtime控制与恢复重建的metadata范围；
- superblock/根目录/持久对象目录；
- 至少两代checkpoint及发布描述；
- 日志与一次可完成checkpoint的备用工作集。

[S-05持久格式](persistent-format.md)已给出64B目录记录、256B WAL槽、R内镜像/辅助区/流式scratch与16MiB几何候选；不能把目录记录大小当作每对象全部开销。默认固定保留1.875MiB，payload候选14.125MiB，实际还受N/运行资源约束。open按记录数和索引前缀核算重建需求，额度不足在发布handle前失败；实际sizeof与正式格式冻结仍须I阶段证据。

VOLATILE文件buffer适用普通自举算式，但借用模式不拥有OS映射变更权限；不能由算式通过推导完整VM或RT支持。

## 8. 本轮核对与后续验收

本轮使用临时算术计算核对字段对齐、槽位取整、表格相加、4TiB节点数、线程/RT和事务公式；**未创建源代码、未编译结构、未调用OS映射或运行基准**。

I-03/I-04实施前必须取得：

1. 64/32位真实sizeof/alignof/offsetof与本表逐项对应；记录大小不同就重算，不只修改一个总数。
2. 目标平台LockSlot/BackendControl原位空间和原子布局验证；ARMv7明确u64对齐不代表u64 lock-free。
3. 最小16MiB创建不调用外部heap；M0、库存、应急和控制岛尾部分别测量。
4. 首次ctx attach/普通分配、metadata尾部用尽后增长、两次连续失败后拒绝增长的定向测试。
5. 大对象只索引合法返回页，slab/RT覆盖合法起点页；释放、realloc、诊断地址查询分别验证。
6. requested_live未跟踪时不可用；启用u16表的额外开销被计费。
7. 4TiB只reserve与稠密payload的结果分别报告；一般设备仅验证模型不冒充实建成功。

S-02已使用这些单价完成默认额度、预估器及配置组合的拒绝规则。v0.3专项对HeapHeader维护游标、owned-ring、中央/退休队列和终态X链只做同宽字段重命名或生命周期互斥复用，以上槽位、L0及应急公式不变。arena维护每类最多8条、两类合计16指针的固定栈工作区为64位128B、32位64B；这是已允许的有界调用栈成本，不是arena内新增metadata，也不包含其余普通栈局部变量，实际总栈占用仍须I-03/I-07检查。终态X/J保留会减少可用记录库存，不能因为名额已退而把记录费用抹掉。

