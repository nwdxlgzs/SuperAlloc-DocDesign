# 平台与虚拟内存契约

本文定义 SuperAlloc 的平台边界、后端能力和验收要求。它是设计约束，不表示这些后端已经实现或通过实机验证。公共 API 采用统一状态码与输出参数，具体签名以 [API 契约](api-contracts.md) 为准。

## 1. 支持范围与不变量

- SuperAlloc 提供独立的 malloc 类和 mmap 类函数，允许同一进程同时持有多个 arena；不替换进程 malloc、new/delete 或系统 mmap，不提供 interposition。
- 一个完整 VM arena 拥有一个独占、连续的虚拟地址保留范围。内核可以将它拆成多个 VMA、Windows placeholder、private allocation 或 file view；不保证只有一个内核映射对象。
- malloc 返回至少 16 B 对齐的地址；页映射以运行时查询的系统页大小 `P` 对齐。不得将 `P` 写死为 4096。
- 任何固定位置操作只能作用于本 arena 已持有且通过范围账本验证的区间，不能覆盖 malloc span、元数据、其他实例或进程中任意地址。
- 16 MiB 是最小 arena 容量；4 TiB 是 64 位实现的设计上限，不是所有设备必能成功保留、提交或驻留的容量。请求失败必须返回明确状态，不静默缩小 arena。
- 物理提交、驻留、页保护、后备对象和分配所有权分别记录。对地址执行 `NOACCESS` 不等于解除映射；解除某个 view 不保证后备对象的 commit 已被释放。

## 2. 首版维护基线

以下是项目选择的构建与 CI 支持基线。除明确引用的平台要求外，它们不是对 API 最早可用版本的声明。

| 平台 | 首版版本与架构 | 必测配置 |
|---|---|---|
| Windows | Windows 10 1803+ desktop；x86-64、AArch64 | placeholder 替换、原生系统页、不同 allocation granularity |
| Linux | kernel 5.4+；x86-64、AArch64 | 4 KiB、平台支持时的其他页大小；overcommit 和映射数量限制 |
| macOS | macOS 11+；x86-64、AArch64 | Intel 与 Apple Silicon，实际 4 KiB/16 KiB 页几何 |
| Android | API 23+；`armeabi-v7a`、`arm64-v8a`；NDK r28+ 构建 | 32 位、64 位、4 KiB、16 KiB 设备/系统镜像 |

Windows 的 `MapViewOfFile3` 官方最低客户端版本为 Windows 10 1803。其他平台基线是维护选择；支持某版本仍要求通过下文的能力探测和实机验收。Android x86/x86-64 纳入兼容测试矩阵，不能用模拟器上的结果替代 ARM 设备验收。

### 2.1 32 位 Android

- 配置长度、文件 offset 和持久 offset 使用明确宽度的 64 位整数；指针和运行时数组访问使用目标 ABI 的 `uintptr_t`、`size_t`。
- 单 arena 的设计上限是向下按 `P` 对齐后的 `PTRDIFF_MAX`，约 2 GiB；实际能保留的连续区间通常更小。4 TiB arena 不能在 32 位进程整体打开。
- 所有长度相加、乘法、向上取整和窄化转换必须在执行之前检查溢出。不能先转换到 32 位再校验。
- 快路径只使用已验证为目标平台 lock-free 的指针宽度或 32 位原子；不以 64 位、128 位 CAS 为通用前提。不允许 `libatomic` 的隐藏锁进入受限实时路径。
- 持久格式中的 64 位字段可能撕裂，必须由日志、代次与校验检测，不能依赖 ARMv7 上的自然 64 位原子写。

## 3. 三类 arena 来源

| 来源 | 首版保证 | 不自动获得的能力 |
|---|---|---|
| 分配器自行保留的 VM arena | 可在独占范围内管理匿名 extent、指定文件子区间和保护；能力依平台后端验证结果公开 | 不保证任意 backing 的页级回收、同一 OS 调用原子修改多个 view |
| 调用方提供的已映射 buffer | volatile malloc 与页对齐的逻辑子分配；所有元数据位于该 buffer 内 | 不自动 `mprotect`、decommit、解除或替换映射；不自动把它认作 durable store |
| 调用方提供完整 reservation/provider | 保留接口设计位置；首版不实现通用 adoption | 仅凭地址对齐或 `VirtualQuery` 结果不能推导出独占和替换权限 |

调用方 buffer 必须满足最小容量、基地址和长度的页对齐约束，并在 arena 生命周期内保持有效、可写和独占管理；buffer 的原始后备对象仍归调用方。该模式的页释放只把区间交还 arena，不能承诺向 OS 释放 commit。销毁 arena 不解除调用方的 buffer。首版受限实时 prepare 不接受借用 buffer，只接受分配器拥有 VM 操作权限的 volatile 匿名 arena。

未来 provider/adoption 必须提供可验证的范围所有权 token、允许的操作、页几何、失败回滚与释放责任。没有该契约时返回能力不足，不尝试猜测和转换调用方映射。

## 4. VM 后端能力矩阵

| 操作 | Linux / Android | macOS | Windows |
|---|---|---|---|
| 独占地址范围 | 匿名 `PROT_NONE` 保留 | 匿名 `PROT_NONE` 保留 | placeholder reservation |
| 匿名页启用 | 建立/恢复匿名 backing 和保护；按策略预触达 | 建立/恢复匿名 backing 和保护；按策略预触达 | private allocation 的 reserve/commit，或有明确生命周期的 pagefile section |
| 回收匿名物理资源 | discard/decommit 策略按后端区分；不得混淆保留与释放地址 | 同左，按支持的 discard 语义实现 | private allocation 可页级 decommit；section 有单独限制 |
| 文件映射 | 指定 FD、页对齐 offset、SHARED/PRIVATE | 指定 FD、页对齐 offset、SHARED/PRIVATE | section + `MapViewOfFile3` placeholder 替换 |
| 页范围保护 | 原生；仍需 ledger 验证 | 原生；仍需 ledger 验证 | 页保护按 backend view 边界分批执行 |
| 页范围解除/更换 backing | 原生范围能力，arena 内需恢复保留态 | 原生范围能力，arena 内需恢复保留态 | 普通 view 整体操作；可拆模式预先建立独立细粒度 views |
| 保证 RAM 永不换出 | 需要锁页成功及平台权限/额度 | 同左 | 同左 |

Linux/Android 与 macOS 没有与 Windows commit charge 完全相同的统一承诺。“启用访问成功”不代表之后不会缺页、发生内存压力或遇到 OOM。`reserve`、`commit/enable`、`prefault`、`lock` 必须是可区分的操作和统计口径。

POSIX 后端在 arena 内释放文件映射时，应按经验证的固定替换策略恢复 `PROT_NONE` 保留态，避免先释放地址再重新抢占产生地址窗口。不能把 `MAP_FIXED_NOREPLACE` 用于替换自己的 `PROT_NONE` 保留映射：该范围在内核看来已经被占用。任何固定替换均以 broker 所有权验证及仍有效的原生所有权证据为前提；broker 中存在 QUARANTINED 记录本身不提供固定覆盖权限。

### 4.1 Windows placeholder 的精确边界

`MapViewOfFile3(..., MEM_REPLACE_PLACEHOLDER, ...)` 要求 `BaseAddress` 和 `ViewSize` 精确匹配目标 placeholder。官方文档明确此模式不适用通常的 64 KiB `BaseAddress`/`Offset` 对齐限制，文件 offset 按底层系统页对齐。

`VirtualFree` 支持拆分、合并 placeholder，但不能由此推导出 active file view 可被拆分。`UnmapViewOfFile2` 没有长度参数，解除整个 view，配合 `MEM_PRESERVE_PLACEHOLDER` 恢复保留态。

`VirtualAlloc2` 文档对普通地址仍描述 allocation granularity 约束，其私有 allocation placeholder 替换没有与文件映射一样明确的页对齐豁免。首个实现阶段必须实测并冻结匿名后端的合法粒度；未经验证不能承诺任意 `P` 对齐的 private replacement。若目标能力不足，创建相应映射 profile 时明确失败，不能偷偷用影响相邻活跃页的重映射模拟。

### 4.2 普通与可拆映射

每次页映射在创建时选择一种 profile，建立后不可无扰动升级：

| Profile | 契约 | Windows 实现要求 |
|---|---|---|
| `ORDINARY` | 优先少量大 view；解除、更换 backing 以其实际 view 边界为单位 | 保留 view 边界表；不支持的部分请求在触及 OS 前拒绝 |
| `PAGE_SPLITTABLE` | 支持以 `P` 为单位解除中间区间和更换 backing，两侧活跃区间不被临时撤销 | 初次映射即把相应 placeholder 拆成逐页 view；不事后解除大 view 再补回两侧 |

Linux/macOS 可以使用原生范围操作满足可拆契约。跨平台调用者不能因为某个后端具有更宽能力，就把 `ORDINARY` 当作隐含可拆模式。

页映射 handle 管理其仍存活的片段。部分解除后，两侧存活片段保持同一 handle 与 generation，洞回到空闲账本；之后在洞里建立的新映射获得新 handle。任何后续操作都重新核对当前 owner，旧 handle 不能跨过已释放的洞操作新 owner。该验证先于 OS 调用。

可拆策略适用于需要更换 backing 的匿名和文件页映射。仅把 private allocation 的一部分 decommit，并不意味着能将这一洞直接换成另一个文件 view。

多 view 操作不具备单次 OS 操作的原子性。broker 在操作期间独占受影响区间，预备所有需要的元数据；成功后统一发布。创建失败须撤销已建立部分并保留范围所有权。对已有映射，只有原 backing、保护和内容契约都可证明恢复，才可以报告完整回滚；恢复地址不等于找回被丢弃的匿名页或 PRIVATE 修改。无法满足时返回 `PARTIAL_FAILURE`，逐片报告已完成、未改变和隔离结果；不确定范围标记 `QUARANTINED`，后续分配不能复用。详见 [架构](architecture.md) 与 [API 契约](api-contracts.md)。

### 4.3 view 预算

首版默认资源策略：

- 每次 `PAGE_SPLITTABLE` 映射最多 4,096 个细粒度 view。
- 每 arena 最多 16,384 个细粒度 view。
- 普通大 view 单独计数、单独受 arena 元数据/普通映射配额管理，不消耗上述细粒度预算。placeholder、普通 view 和细粒度 view 统计不得相互混淆。
- 配额可以显式配置；请求前提供 view 数量预估，超限在 OS 操作之前返回资源限制。不静默退回 `ORDINARY`。

以上是 SuperAlloc 的默认值，不是任何 OS 的公开硬上限。4 KiB 页下，16 MiB 可拆映射产生 4,096 个 view，1 GiB 产生 262,144 个，4 TiB 产生 1,073,741,824 个。不能承诺对整个最大 arena 默认逐页建 view。

多个 view 可以共享 section handle，不需要每页一个 handle；但映射元数据、内核记录、map/unmap/protect/flush 调用次数仍随 view 数量增长。成本应在报告中实测，不写未经验证的固定内核字节数。

### 4.4 Windows 匿名 section 的 commit 生命周期

- 普通 malloc spans 优先使用可独立回收的 private extents，按已验证的 backend 粒度管理。
- 可拆匿名映射若采用 pagefile-backed section，必须记录 section 与 views 的生命周期；释放一个 view 不等于释放它引用的底层 commitment。
- `SEC_COMMIT` 要求系统能够承担整个 mapping 的 commitment，不能用 4 TiB `SEC_COMMIT` section 表示一个仅保留地址的 4 TiB arena。
- `SEC_RESERVE` 允许后续提交，但官方文档明确已提交页不能由 `VirtualFree` 释放或 decommit。禁止将一个巨型 `SEC_RESERVE` section 宣称为可任意页回收 commitment 的 arena。
- 可拆匿名后端可使用受预算控制的有限 section 分组；该实现必须公开仍被其他 view 引用的 retained commitment。若要求释放后立即归还 commitment，则必须选取有独立后备生命周期的实现并计入 handle/section 成本；首版不能未经测量承诺这一额外能力。

### 4.5 元数据 island 与持久主文件 view

bootstrap 与 runtime metadata shard 使用独立的匿名 backing 单元，不与可被用户解除/更换的 payload view 混合。owned 后端按已验证的 reservation/private-allocation 粒度安排其边界；Windows 初始规划以 `G = lcm(P, allocation_granularity)` 对齐这些单元，不能为节省 padding 跨越其他活 view。实际 shard 尺寸与最低 bootstrap 预算在平台 spike 中核算，不在此声称所有平台已满足 64 KiB 控制区目标。

durable arena 仍在同一连续地址范围内，但主文件按独立 backend extents 按需建立 view，不能先把整个 arena 映为一个大 view，再声称任何逻辑 FREE 子页都可换成外部文件。主文件中的 allocation 释放后，若其所在大 view 仍有其他 live 对象，该空闲空间可以供同 backing 对象复用；只有 view 边界允许安全解除/替换时，才能交给其他 backing 的页映射。

因此地址账本中的 FREE 不等于所有 backing 请求都立即可用。映射选择必须同时满足持久分配账本、运行时所有权、backend unit 边界与目标 profile；必要时返回能力/连续范围不足，不搬动或暂时撤销相邻持久对象。runtime island 的持久保留范围与重建预算见 [持久化](persistence.md)。

## 5. 文件大小、共享与持久化

文件映射只获得对指定文件子区间的访问，不获得任意扩展、截断或跨进程写入协调权。默认验证 `offset + length` 在文件有效范围内；不要通过隐式扩容掩盖调用错误。

稀疏逻辑文件长度、已分配磁盘块和映射页的驻留是三个不同概念。macOS `ftruncate` 扩展的读零语义不等于物理磁盘空间已预留；`F_PREALLOCATE` 是额外能力。Windows sparse file、Linux 文件空间预留同样需独立探测。

`MAP_SHARED` 等价模式允许 payload 按应用协议共享，不等于多个进程可以共同调用同一 arena 的分配器。首版每个 durable arena 只有一个进程 owner；详见 [持久化](persistence.md)。`PRIVATE`/COW 修改不承诺写回原文件。

API 返回 flush 成功时必须指出达到的阶段，例如已向 OS 提交、后端同步操作完成、已达到配置的文件持久化屏障。后者仍依赖文件系统、驱动和设备遵守 flush 合同，不应宣传成所有硬件断电下的无条件保证。

## 6. 能力查询和失败契约

能力按 arena 来源、后端和具体操作返回，不使用一个含糊的“支持 mmap”布尔值。设计至少覆盖：

- `page_size`、`allocation_granularity`、有效地址宽度、arena 最大容量和实际保留容量。
- 文件 offset 粒度、普通 view 解除粒度、是否支持可拆 profile、最大 view 配额及当前占用。
- reserve、启用访问、discard、decommit、prefault、lock、protect、fixed backing replacement 分别可用与否。
- `persist` 可达到的阶段、同步文件屏障能力及经认证的文件系统范围。
- 是否拥有原始 reservation，是否仅借用固定 backing，是否允许执行 OS 映射修改。

能力阶段应明确区分 `UNAVAILABLE`、`IMPLEMENTED_UNVERIFIED`、`VERIFIED`；操作成功与否另有运行时状态。设计文档中的目标能力不能被 API 报成已验证能力。第一版不支持的请求应返回不支持/参数/配额错误，不能隐式复制到其他 arena、越过用户给定的容量或改用系统 heap。

### 6.1 原生失败、内容与清理证据

后端不能将所有失败压成一个“调用未成功”。每个已发步骤必须区分：确定无效果、确定已修改、已完整恢复、结果未知；上层据此发布完整片段报告并保留实际资源责任。统一决策与在途协议见 [故障恢复](failure-recovery.md)。

恢复同一 base/length 和原保护不代表完整回滚。匿名页被丢弃后新映射的零页不是旧内容；PRIVATE/COW view 重新映射原文件也不能还原已修改字节。多 view 中途失败只能在 backing 身份、保护、内容与相邻 live 范围全部符合原合同后声明恢复；否则确定片段按实际结果发布，未知片段隔离。所需备份、事务步骤和结果报告均在触及 OS 前取得预算，不能事后向外部 heap 求救。

原生解除/替换结果未知时，同一地址可能已被外部重用。查询发现“仍有映射”不能证明它仍属于本库；禁止盲目固定覆盖、解除或用整段 arena teardown 跨越该范围。能够界定受影响范围且其余控制/所有权可信时，局部 QUARANTINED 即可，其余有效范围继续使用。不可界定的控制或所有权破坏属于后端核心不变量失效，停止实例使用并保留诊断，不能借 WRITE_FAULTED 伪装成持久问题或承诺任意 free 仍然安全。

清理只针对仍有可靠身份的原生资源，已完成的 close/unmap 不重复执行，FD 数字或地址重现不构成安全重试依据。decommit、view 解除、最后 section 引用释放分别按后端已确认效果退账；未知和 retained commitment 继续计入 used。逻辑 free 的成功不因延后物理清理失败撤销。未知身份与可重试责任使用既有 cleanup 记录分开管理，控制岛最后处理。

## 7. 平台 spike 与验收门槛

在平台后端定型之前完成以下可复现 spike，并将 OS build、CPU、页大小、文件系统和结果写入验证记录：

1. Windows：在现代 placeholder 内测试 `P` 对齐文件 offset/base/size，包含非 64 KiB 子区间；验证 split、replace、unmap-preserve、coalesce。
2. Windows：确定 private replacement 合法粒度；分别测量 private decommit、pagefile section view 解除、最后一个 view/handle 释放后的 commitment 行为。
3. Windows：持续访问左右两侧时移除中页，再映射第二个文件；普通 view 应预检拒绝不支持的请求，可拆 view 不得使两侧瞬时失效。
4. Linux/macOS/Android：保留范围内的匿名/文件固定替换、保留态恢复、页保护和回收语义；验证操作失败不会把地址交给其他线程。
5. macOS/Android：4 KiB/16 KiB 实测；32 位 Android 验证 16 MiB 自举、容量上限、窄化检查及原子实现。
6. 大容量：64 位稀疏 4 TiB reservation、稀疏元数据按需增长；不能把成功保留当成 4 TiB RAM 或磁盘空间可用。资源不足是可验收的明确失败。
7. 文件：只读/共享/私有映射、文件边界、显式同步、磁盘满/文件截断错误、多个 arena 使用同一外部 payload 文件。
8. 故障：view/placeholder/元数据配额耗尽、OS 中途失败、rollback 失败进入隔离态；逐步验证保留态无地址窗口、匿名/COW 内容不被伪恢复、未知身份不会被盲清理、确认资源释放后才退账；失败期间其他 arena 正常运行。
9. 调用方 buffer：禁止 OS 改写 backing，所有元数据与对象位于 buffer 内；销毁不解除 buffer。
10. 受限实时：纯匿名预备池、预提交/预触达、登记线程；运行中没有 VM/文件/flush 调用。锁页失败不能静默宣传为无缺页保证，算法时延边界与 OS 调度/硬件时延分开。

## 8. 官方资料与核验状态

以下 Microsoft 与 Apple 页面已用于核对接口语义。Apple 为官方归档，现代系统仍需上节实测。

- [MapViewOfFile3：placeholder 替换、对齐和最低客户端版本](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile3)
- [VirtualAlloc2：reservation 与 placeholder](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc2)
- [VirtualFree：placeholder 拆分/合并与 private decommit](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualfree)
- [UnmapViewOfFile2：整个 view 恢复 placeholder](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-unmapviewoffile2)
- [CreateFileMapping：SEC_COMMIT/SEC_RESERVE 生命周期](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga)
- [Apple mmap](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/mmap.2.html)
- [Apple munmap](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/munmap.2.html)
- [Apple truncate](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/truncate.2.html)
- [Apple fcntl：预分配与 F_FULLFSYNC](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fcntl.2.html)

Linux/Android 的当前版本约束还需结合目标系统头文件、官方文档和实机 spike 核验。Android 16 KiB 页支持指南为 [Support 16 KB page sizes](https://developer.android.com/guide/practices/page-sizes)；此次设计阶段对该页面的网络读取超时，不能把它当作已经完成验证的证据。
