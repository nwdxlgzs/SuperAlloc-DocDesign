# 受限实时 Profile

本 profile 为需要控制分配器工作量的语言运行时、游戏帧循环、音频或其他受限线程提供独立 try 接口。它保证设计上的有界步骤和明确失败，不承诺通用操作系统上的硬实时墙钟期限。当前只有设计，尚无实现或实测数据。

最终选择是：**预备固定 pool，任意已登记线程对合法对象的 free 必须成功；以每个 slot 的原子归还状态替代远程队列。** 没有 queue-full 状态，不要求对象绑定固定归还线程。

## 1. 使用阶段与资源合同

### Prepare / 冷路径

prepare 在普通控制线程上完成所有可能缓慢或失败的工作：

- 从指定 arena 的统一 broker 预留独占 RT pool；显式配置 size classes、每类 slot 数、固定页块种类、owner context、允许的生产者/context 数量及维护预算。
- 分配全部 slot 状态、freelist、固定深度地址索引和 context 记录；资源全部计入 arena。除允许的少量 TLS/OS handle 外，不借外部 allocator 存储控制结构。
- commit 并预触匿名 backing；按配置尝试锁页。要求锁页时，锁页失败必须使 prepare 失败，不能把“已预触页”当成“今后绝不缺页”。
- 验证所用原生宽 atomic load/store 在目标平台为 lock-free，验证索引、计数及容量无溢出。
- 建立固定 owner 身份。每个 pool 在运行期只有一个线程执行 alloc 和 maintenance；任意已登记线程均可承担合法对象的最终 free。

首版RT只允许VOLATILE模式、库自行保留并拥有完整VM操作权的预备匿名pool；通用provider导入尚不提供，借用buffer不获得RT profile。文件映射、recoverable元数据提交、flush、动态protection和任意重映射不进入RT阶段。预触文件映射不能保证后续不发生I/O缺页。

### Run / RT 路径

运行期禁止：OS 调用、动态登记、元数据增长、broker/普通 heap 回退、阻塞锁、条件变量、CAS 重试循环、整 arena 搜索、后台等待或隐式日志提交。

所有操作使用显式 arena/pool/context handle；不根据指针搜索全局 arena，不修改隐式 errno。RT 不提供无 context 的共享 fallback 入口；归还线程使用自己的已登记 context，不能借用 owner 的 context。普通分配函数与 RT try 函数在接口层区分，具体命名和状态码以 [API](api-contracts.md) 为准。

prepare为每个允许的ctx/pool组合预建稳定RTBinding，run入口直接验证binding字段，不遍历可增长登记表。binding由所属arena计费，32/64位布局见[metadata-layout.md](metadata-layout.md)。

### Quiesce / 退出

先阻止新的pool使用并由应用同步所有持有者，再接收Returned、确认无活slot，才能逻辑销毁pool和撤销bindings。控制记录进入既有退休链，解除锁页与物理回收由维护完成；合法逻辑销毁不依赖新的清理记录。首版改变owner须销毁空pool后重新prepare。不能在仍可能free的线程运行时重建索引或撤销descriptor。

## 2. 每 slot 状态机

每个 slot 拥有独立、区域内侧置的 `uint32_t` 原子状态，候选值为 Free、Allocated 和 Returned。它的地址在 pool 整个运行期保持不变。

owner 的 freelist 采用每类一个预建 head，next 存在 Free slot 的 payload 内；owner acquire 观察 Returned 之后才可写入 next。运行中的合法归还线程只写侧置状态，不需要修改 payload 或链接其他对象，因此无需为每次归还分配额外节点。

| 转移 | 执行者 | 条件及内存序 |
| --- | --- | --- |
| Free → Allocated | 唯一 owner | 从 owner 私有 freelist 取出，owner relaxed store；对象交给其他线程必须经过应用同步 |
| Allocated → Returned | 唯一合法释放者 | release store；这次 free 此后不再访问对象及 descriptor |
| Returned → Free | 唯一 owner | acquire load 观察 Returned 后回收，owner relaxed store，再接入私有 freelist |

### 为什么 free 必成功

每个可分配 slot 已经有自己的归还状态位置；它既不争用队列槽，也不申请新记录。合法 free 只执行固定深度 slot 定位和一次 release store，所以不存在“归还空间不足”。

运行期 free 不用 CAS 从 Allocated 检验并改为 Returned。接口要求同一对象仅有一个有效释放责任者；重复 free、并发 double free、free 后访问及向错误 pool 释放均属于非法使用。调试检查不能被解释成生产版对这些错误的完整检测保证。

本地 owner 的 free 首版也采用同一 Returned 发布流程，使语义和证明一致；是否增加 owner-local 直接回收优化，必须在不改变成功合同的前提下另行基准验证。

### 可见性与复用

应用把对象交给归还线程时必须有合法 happens-before，例如受支持的同步队列；分配器不修复应用的数据竞争。释放者对 payload 的此前操作由 release store 排序，owner acquire 观察归还后才可重用该 slot。

owner 的重新分配写入 Allocated；后续同 owner 扫描受 atomic coherence 约束，不能把它自身此前那一代 Returned 再当成一次新归还。旧用户指针跨复用产生的 ABA 属于非法生命周期，裸指针接口不可能完整识别。

free 的发布是对 slot/descriptor 的最后访问。owner 可在看到 Returned 后立即重用 payload，即使原释放线程尚未从函数返回；原释放线程此后只能完成不再访问该对象的自身调用收尾。

## 3. 有界扫描与分配结果

### 默认策略

- 每个 pool 定义固定 scan-set，包含其 N 个 slot；为 owner 保存轮询 cursor。
- 每次 try-alloc 或显式 maintenance 最多检查 K 个状态，默认 K = 64，可在 prepare 配置为正整数；本次实际检查不超过 `min(K, N)`。
- 一次扫描最多回收同样数量的 slot，不遍历不相关 pool，也不在 RT 调用中不断扫描直到成功。
- 首版 try-alloc 先执行一段预算内维护，再查询目标类 freelist；有 slot 则分配，没有则立即返回 `WOULD_BLOCK`，输出指针保持空值。
- 显式 maintenance 返回本次检查数和归还数等有界结果，允许应用在帧间或固定预算内推进回收。

`WOULD_BLOCK` 仅表示本次有界工作后没有找到可立即分配的匹配 slot，可能是容量耗尽、目标类配额不足，或已有 Returned 尚未被扫描。不得用这一结果宣称整个 pool 或 arena 已无空间，也不为了区分原因增加全池扫描。

### 轮次界限的精确定义

固定轮询顺序下，对 N 个 slot 执行完整 K 预算的维护步，`ceil(N/K)` 步覆盖一轮。该界限描述“owner 检查了哪些位置”，不是以下任何保证：

- 归还线程调用 free 后若干微秒内可以再次分配；
- 没有 owner 继续维护时仍自动回收；
- 并发 release store 必定在某次缺少同步的扫描中立即可见；
- 某 size class 仍有 outstanding 对象时 try-alloc 必须成功。

若应用先建立归还完成与后续 owner maintenance 的同步关系，一次完整扫描能够覆盖这些已同步归还的 slot。一般并发情况下仍遵守 acquire/release 与硬件可见性约束，不虚构 C++ 内存模型未提供的墙钟传播期限。

### 页面能力的限制

RT page allocation 只发放 prepare 已建好的、按真实 OS 页对齐的固定页块。运行期不进行 OS mmap、任意大小拼接、固定地址替换、partial unmap 或保护变更。它是预备 backing 上的页块分配能力；普通 page API 的完整行为由 [平台合同](platforms.md) 定义。

若请求大小、对齐、页块类别不在 prepare 的静态集合中，按 API 参数/能力合同拒绝，不通过普通 allocator 隐式补足。

## 4. 成本、平台和可控边界

### 容量核算

RT pool 至少计入 payload、slot 状态数组、owner freelist 表示、page/slot 地址索引、context 和 alignment padding。状态数组本身为 `N × sizeof(uint32_t)`；对 16 B slot，仅 4 B 状态就相当于 payload 容量的 25%，尚未计算其他元数据。

不默认将每个状态 padding 到 cache line。不同生产者写相邻状态可能产生伪共享；布局分区、较大 slot 或专门 profile 可减少争用，但必须公开其额外内存成本。

每次维护检查至多 K 个状态，理论状态字段读取量约为 `4K` B，实际 cache-line 流量、coherence 和索引读取另计。K 越大，待回收资源通常越快被发现，但单次调用访问更多内存；K 越小，则更容易出现短暂 `WOULD_BLOCK`。

### 原子与硬件

32 位 Android 使用原生可 lock-free 的 32 位原子 load/store，不依赖 64 位或 128 位 CAS。prepare 和构建验证必须排除会隐藏调用锁实现的平台组合。

固定指令数量仍可能受到 cache miss、缓存一致性、抢占、CPU 频率变化、中断、内存控制器、缺页或系统回收影响。锁页权限及限制也因系统而异，所以只能宣称 allocator 算法工作量有界，不能宣称跨四平台的硬实时 WCET。

### 可选后续优化

信用预付的 SPSC return route 可以作为独立实验 profile：分配预付路由容量，归还不可能满；但它增加固定归还路由或路由容量成本，不是首版要求，不改变“任意已登记线程合法 free 必成功”的默认合同。未经单独 ADR 和等价正确性验证，不引入 queue-full 回退。

## 5. 必须通过的验收场景

- 任意已登记线程释放其他 owner 分配的对象；全部对象同时远程释放，所有合法 free 均成功且无需额外容量。
- 将生产者暂停在 release store 前后；owner 并发扫描及复用，不提前回收、不重复接入 freelist。
- K = 1、64、N 及 K > N；cursor 回绕正确，一轮覆盖每个 slot 且单次不超过预算。
- 归还集中在 cursor 后方，目标类 freelist 为空；try-alloc 返回 `WOULD_BLOCK` 而不扫描全池或误报全局内存耗尽。
- 不同 size class、固定页块、多个 owner pool、多个 arena 并行；free 的目标索引和状态完全隔离。
- 原子/索引计数边界在 32 位目标上无截断或溢出；运行时审计无 OS、锁、CAS 循环、动态分配、日志或隐藏 fallback。
- prepare 注入 commit、预触、锁页、metadata 预算和平台能力失败；失败不能留下可误用的半初始化 RT handle。
- quiescent teardown；首版改变owner需清空销毁pool后重新prepare，不导出活pool迁移；运行中的free不与descriptor/index销毁并发。
- 扫描吞吐、空扫描成本、归还后可复用的维护次数、伪共享和 p99.9 延迟分别测试；不得把空扫描较少的专门负载作为普遍优势。

所有性能结论需遵守 [基准与验收方法](benchmarking.md)。
