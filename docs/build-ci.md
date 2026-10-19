# 构建、工具链与持续验证规格

状态：实施规格，没有构建文件或已通过的CI。本文件规定I-03应创建什么、各阶段如何验收；下面命令是**未来目标约定，目前不可执行**。

## 1. 模块与依赖方向

| 未来模块 | 职责 | 允许依赖 |
| --- | --- | --- |
| public C header / ABI adapter | 校验版本、参数、输出及C符号 | core固定类型与各前端 |
| checked arithmetic / geometry | 长度、偏移、对齐及类表 | 无OS/allocator依赖 |
| metadata / index / broker | 区内pool、目录、统一地址账本 | checked arithmetic、backend接口 |
| object / heap | 普通本地与远程对象路径 | broker、metadata；不直接reserve第二片区域 |
| mapping | 句柄、步骤计划、部分结果 | broker、backend |
| realtime | prepare与固定slot run | prepare可用broker；run仅固定内存与原子 |
| persistence | 字节编码、日志、checkpoint、恢复 | broker、文件后端；不调用普通malloc分配scratch |
| platform windows / posix | VM、文件、同步、原位锁 | 目标系统接口；不能反向调用公共allocator |
| model / tests / benchmarks | 参考模型、调度器、适配器和测量 | 公共接口及受控测试钩子；外部堆与被测核心分账 |

生产编译单元不得包含测试的容器、日志或故障注入堆依赖。核心静态初始化仅常量数据和固定定位锚点；所有需要生命周期的native状态显式建立与释放。无全局默认arena，无进程malloc/new符号替换。

## 2. 工具链选择

CMake最低候选3.28，C17测试调用方、C++20核心；Ninja作为常见生成器，Windows也验证Visual Studio生成器。版本是首轮工具链候选，I-03必须保存真实编译器/SDK版本、链接参数与布局结果再确定支持表。

| 平台 | 候选工具链 | 发布必须覆盖的架构/条件 |
| --- | --- | --- |
| Windows | VS2022 MSVC v143、clang-cl；支持placeholder API的Windows SDK | x86-64、AArch64；真实Windows 10 1803+目标能力探测 |
| Linux | GCC13+、Clang17+ | x86-64、AArch64；内核5.4+基线运行 |
| macOS | Xcode/Apple Clang支持C++20版本，deployment target11 | Intel与Apple Silicon；真实4KiB/16KiB |
| Android | NDK r28+ Clang、API23最低目标 | armeabi-v7a、arm64-v8a生产；x86/x86-64兼容；16KiB装载与设备运行 |

选择新编译器不意味着目标OS必须升级。标准库/CRT的最低运行依赖必须在目标基线验证；不能通过开发机能链接便宣称兼容。Android 16KiB不仅是运行时P：ELF LOAD段对齐、NDK默认行为与应用打包方式也进入检查；以目标官方工具与装载结果确认，不能只看源文件里的常数。

核心编译禁异常/RTTI，隐藏非公开符号，C导出采用固定调用约定。Windows核心禁throw/unwind依赖必须同时检查编译设置与产物导入；POSIX检查未定义符号。无外堆不能只靠grep `malloc`：链接后审计operator new、CRT初始化、libatomic与平台包装分配，运行时计数被测模块调用。测试框架自身允许使用堆。

## 3. 构建选项与目标约定

| 未来CMake选项 | 默认 | 行为 |
| --- | --- | --- |
| SA_BUILD_SHARED | OFF | 静态库默认；共享库另验导出/装载/运行时依赖 |
| SA_BUILD_TESTS | OFF | 参考模型、故障后端、C调用和功能测试 |
| SA_BUILD_BENCHMARKS | OFF | 只构建测量框架，不自动运行高负载 |
| SA_BUILD_SPIKES | OFF | 独立平台可行性试验，不写正式兼容store |
| SA_ENABLE_ASSERTS | Debug开启 | 内部不变量诊断；不把assert作为公共参数处理 |
| SA_SANITIZER | none | 单独选择受支持的address/undefined或thread配置，不能混用互斥工具 |
| SA_TEST_FAULT_INJECTION | OFF | 仅测试目标可用；生产导出不包含故障钩子 |

运行时TRACK_REQUESTED、DIAGNOSTIC_CHECKS、容量和额度通过[C ABI](c-abi-spec.md)传入，不用为每个arena配置重编库。不能将系统页大小变成生产编译选项。普通、RT、持久是运行时显式能力组合，未实现阶段必须报告不可用。

未来基本操作顺序：

```text
cmake -S . -B build/debug -G Ninja -DCMAKE_BUILD_TYPE=Debug -DSA_BUILD_TESTS=ON
cmake --build build/debug
ctest --test-dir build/debug --output-on-failure -L core
ctest --test-dir build/debug --output-on-failure -L fault
```

Windows多配置生成器追加对应 `--config Debug` / `-C Debug`。Android在独立build目录提供NDK toolchain、目标ABI与API，先编译后由设备runner执行；交叉编译成功不构成运行通过。平台spike按 [实验规程](platform-experiments.md)逐项授权阶段运行。不能从这些文字命令推断目前存在相应目录、目标或二进制。

## 4. CI分层与阶段开关

| 阶段 | 触发与范围 | 失败处理 / 证据 |
| --- | --- | --- |
| 文档 | 每次文档变更 | 本地链接、围栏、公式、依赖与跨层术语；本次只执行此层 |
| compile/ABI | I-03后，每次相关改动 | C17/C++20、32/64位布局、导出、禁用异常/RTTI、native atomics |
| core/model | I-04后，每次核心改动 | 确定seed、区间参考模型、整数边界、16MiB资源耗尽 |
| fault/concurrency | I-07后，相关变更必跑 | 失败序号可重放、remote生命周期、owner转移、sanitizer |
| platform | 后端变更与候选发布 | 真实四平台VM/页大小；无runner明确missing，不勾选通过 |
| durable | I-10/I-11后，每个格式或I/O变更 | kill、torn/reorder、未知结果、双super/checkpoint切换；隔离实验文件 |
| performance | 正确性通过后的固定机器 | 基准manifest、噪声与回归阈值；共享繁忙runner只作烟测 |

合并验证按影响范围选择，候选发布覆盖全部适用矩阵；不为每个文档改动反复运行无关负载。真实断电验证单独计划，不由普通PR自动操纵设备电源。模型通过、模拟器通过、真机通过使用不同标签。

## 5. 测试钩子与结果归档

测试后端能在第n个record申请、quota预付、Pending发布、native操作、rollback及持久屏障边界注入三类结果：无效果失败、有部分效果失败、结果未知。调度器能暂停remote发布和owner回收等关键点。钩子只控制明确步骤，不能在生产RT路径添加日志或回调。

每个结果记录：任务/V用例、文档与代码版本、工具链、平台/P/G、模式、完整求值配置、seed/调度序列、退出状态、日志位置、跳过理由。ABI产物包括C/C++布局清单与导出列表；崩溃产物包括原始文件镜像和注入序号；benchmark按既定manifest归档。归档中的业务数据与用户文件不作为默认测试素材。

代码阶段首个提交应是获准范围内可验证的spike/模型，不先创建一堆空接口再标模块完成。各里程碑退出条件以[TODO](../TODO.md)及[验证矩阵](validation.md)为准。
