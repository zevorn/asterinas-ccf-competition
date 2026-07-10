# 第八届 CCF 开源创新大赛 · 星绽开源社区贡献赛题

## 作品信息

| 项目 | 内容 |
| --- | --- |
| 战队 | 搭把手战队 |
| 队长 | 泽文（GitHub: [zevorn](https://github.com/zevorn)） |
| 队员 | 邵志航（GitHub: [BattiestStone4](https://github.com/BattiestStone4)） |
| 参赛项目 | [Asterinas（星绽）](https://github.com/asterinas/asterinas) —— Rust 编写的 Linux 兼容内核 |
| 主作品 | [#3412 Add GDB helpers for kernel debugging](https://github.com/asterinas/asterinas/pull/3412) |
| 设计讨论 | [#3094 Add GDB helper scripts for non-invasive kernel debugging](https://github.com/asterinas/asterinas/issues/3094) |
| 演示 PPT | [asterinas-gdb-helpers-ccf-demo.pptx](outputs/asterinas-gdb-helpers-ccf-demo.pptx)（含视频演示流程页） |
| 资料核验日期 | 2026-07-10 |

> 状态说明：#3412 当前为上游开放 PR，尚未合并。本作品展示已实现、已测试的方案，不将其表述为“已合入 Asterinas 主线”。

## 摘要

本作品聚焦 Asterinas 的 GDB helpers 内核调试方案。它将一次真实的进程加载断错误排查，沉淀为一套可自动加载、可脚本化、可回归测试的调试工作流。

方案不向内核加入专用调试 hook、日志或系统调用，而是复用已有的 DWARF 调试信息、GDB Python API、QEMU gdbstub 和 <code>rust-gdb</code>。开发者运行 <code>cargo osdk debug</code> 后，即可直接使用面向 Asterinas 内核对象的命令与函数，观察进程、线程、文件描述符和运行时间，并通过真实 QEMU + GDB smoke test 防止脚本随内核布局演进而静默失效。

## 1. 为什么需要这套 GDB helpers

### 真实问题的起点

方案不是从抽象功能清单出发，而来自一次真实排障：在进程加载阶段，Asterinas 会处理 ELF 中的 <code>PT_INTERP</code>，将动态链接器映射进用户进程地址空间。排查时发现，编译阶段链接到的 Rust 动态库版本与 OS rootfs 实际加载的版本不一致。

表面看，符号名仍然能匹配；实际地址背后的函数已经发生偏移。在加载 interpreter 和后续依赖库时，内核最终触发 trap，用户进程收到 Segmentation Fault。继续追踪时，排障需要跨越动态链接、进程加载、trap handler、内核对象和 Rust 类型系统。

### 真正的成本：重复且分散的调试动作

GDB 可以连接 QEMU gdbstub，但 Rust 内核的默认调试体验仍有明显摩擦：

1. Rust 的 crate、模块、impl / trait 上下文与泛型单态化会形成冗长、相似的符号名；
2. <code>AtomicBool</code>、<code>Mutex&lt;T&gt;</code>、<code>RwMutex&lt;T&gt;</code>、<code>SpinLock&lt;T&gt;</code> 等 wrapper 会把 <code>UnsafeCell</code>、锁内部状态等实现细节展开；
3. 找一个进程或线程需要从 PID 表开始，经过锁、<code>BTreeMap</code>、<code>Arc</code>、<code>Weak</code> 和 <code>Option</code> 等多层结构；
4. 这些动作单独都不难，但组合起来会持续打断调试者的注意力和假设验证节奏。

因此，GDB helpers 的目标不是替代 GDB，而是把最耗注意力、最重复的动作变成稳定的调试基础设施。

## 2. 方案目标

| 目标 | 方案回应 |
| --- | --- |
| 非侵入 | 只读取已有 DWARF 信息和实时内核对象，不增加运行时调试接口 |
| 开箱可用 | <code>cargo osdk debug</code> 自动使用 <code>rust-gdb</code> 并 source helper |
| Rust 类型感知 | 对原子类型、OSTD 锁等提供 pretty-printer，减少实现噪声 |
| 内核对象可达 | 提供 PID 表、进程、线程、文件表的 convenience function |
| 高频状态可读 | 以 <code>ast-*</code> 命令聚合进程、线程、fd、进程树和运行时间 |
| 长期可靠 | 用 QEMU + GDB smoke test 与 CI 验证基础能力 |

## 3. 实现思路与策略

### 3.1 先做高 ROI 的三层能力

RFC 讨论后，方案没有追求“命令越多越好”，而是优先实现最通用、投入回报最高的三层能力。

| 层次 | 实现 | 价值 |
| --- | --- | --- |
| 减噪 | wrapper-type pretty-printer | 让原子类型、锁等直接以业务可读的形式显示 |
| 导航 | GDB convenience function | 解决“如何到达对象”，保留原生 GDB 表达式的灵活性 |
| 聚合 | <code>ast-*</code> 命令 | 只处理集合遍历、汇总、表格和树形展示 |

这形成清晰分工：**导航交给 helper，展示交给 GDB，pretty-printer 负责降低噪声。**

### 3.2 自动加载：把正确入口变成默认行为

在 [OSDK 的 debug 命令实现](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/osdk/src/commands/debug.rs) 中，调试器被切换为 <code>rust-gdb</code>。代码通过 Cargo metadata 找到 workspace root；发现 <code>scripts/gdb/asterinas-gdb.py</code> 后，向 GDB 注入 <code>source</code> 指令。

用户不需要记忆脚本路径或自行维护 <code>.gdbinit</code>。启动调试器时，远端连接和 helper 加载一起完成。

### 3.3 分层实现，隔离 GDB API 与内核布局

| 模块 | 责任 |
| --- | --- |
| [asterinas-gdb.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/asterinas-gdb.py) | 入口；注册 Rust 标准库和 Asterinas 自定义 helper |
| [gdb_bridge.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/helper/gdb_bridge.py) | 封装 GDB Python API、命令/函数注册、参数解析、表格与错误输出 |
| [layout.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/helper/layout.py) | 读取 <code>Arc</code>、<code>Weak</code>、<code>Option</code>、<code>Vec</code>、<code>BTreeMap</code>、原子类型和锁布局 |
| [printers.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/helper/printers.py) | 注册 wrapper-type pretty-printer |
| [kernel.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/helper/kernel.py) | 读取 PID 表，关联 task、thread、PosixThread、文件表和 jiffies |
| [commands.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/helper/commands.py) | 只负责用户命令的参数解析与格式化输出 |
| [constants.py](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/helper/constants.py) | 集中管理符号名、类型名和布局敏感常量 |

与 helper 耦合的 Rust 结构位置以 <code>COUPLED</code> 标记显式提示。这样内核布局发生变化时，维护者能知道需要同步检查 Python helper，而不是让脚本在数月后悄悄输出错误结果。

## 4. 面向开发者的调试接口

| 接口 | 用途 |
| --- | --- |
| <code>ast-version</code> | 显示 Asterinas 内核版本，确认 helpers 已加载 |
| <code>ast-ps [PID]</code> | 按 PID、PPID、状态、线程数、名称显示进程 |
| <code>ast-threads</code> | 显示线程 ID、所属进程和名称 |
| <code>ast-pstree</code> | 以树形关系展示进程父子结构 |
| <code>ast-fds &lt;PID&gt;</code> | 显示 fd、flags、文件类型和标志说明 |
| <code>ast-uptime</code> | 读取 jiffies 并格式化内核运行时间 |
| <code>$ast_pid_table()</code> | 定位 PID 表 |
| <code>$ast_process(PID)</code> | 定位进程对象 |
| <code>$ast_thread(TID)</code> | 定位线程对象 |
| <code>$ast_file_table(PID)</code> | 定位进程文件表 |

convenience function 返回 <code>gdb.Value</code>。这意味着 helper 只解决复杂对象导航，开发者仍可用原生 GDB 表达式继续查看任意新增字段，而不必等待固定命令更新。

## 5. 验证策略与已有证据

### 真实 QEMU + GDB smoke test

[run_smoke.sh](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/test/run_smoke.sh) 执行的是完整调试链路，而不是 mock 环境：

1. 通过 <code>cargo osdk run --gdb-server wait-client</code> 启动 QEMU；
2. 等待 OSDK 创建 GDB socket；
3. 以 <code>rust-gdb --batch</code> 加载带调试符号的内核 ELF 与 [smoke.gdb](https://github.com/zevorn/asterinas/blob/gdb-helpers-foundation/scripts/gdb/test/smoke.gdb)；
4. 在 <code>__ostd_main</code> 和首次 syscall handler 设置断点，确保 PID 1 已建立；
5. 自动断言 printer 注册、关键符号解析、<code>$ast_*</code> 函数、<code>ast-*</code> 命令与对象导航行为；
6. 仅当输出包含 <code>SMOKE: all ok</code> 时判定通过。

PR #3412 的独立 [gdb-smoke-test CI](https://github.com/asterinas/asterinas/actions/runs/28657288013/job/84989356187) 于 2026-07-03 成功完成。这是方案的关键证据：helper 可以随代码在自动化环境中启动真实内核、连接真实 GDB 并完成断言。

## 6. 最终演示成果

### 6.1 录制前准备

~~~bash
git clone https://github.com/zevorn/asterinas.git
cd asterinas
git checkout gdb-helpers-foundation

# 终端 1：启动内核并等待 GDB
cargo osdk run --gdb-server wait-client
~~~

另开终端：

~~~bash
cd asterinas
cargo osdk debug
~~~

加载成功后，GDB 会提示 <code>Asterinas GDB helpers loaded</code>。随后演示：

~~~gdb
ast-version
ast-ps
ast-threads
ast-pstree
ast-fds 1
ast-uptime
p (*$ast_thread(1)).is_exited
~~~

录屏应突出四个结果：

- <code>ast-ps</code> 直接输出进程表，不再手工遍历 PID 表；
- <code>ast-pstree</code> 直接呈现进程树；
- <code>ast-fds 1</code> 直接呈现 fd、flags 和文件类型；
- 打印线程字段时，pretty-printer 不再用 <code>UnsafeCell</code> 等内部实现细节淹没业务值。

实际 PID、名称、fd 数量和 uptime 以录制时内核状态为准。

### 6.2 作品交付物

| 交付物 | 内容 |
| --- | --- |
| 方案文档 | 本 README |
| 精简演示 PPT | [asterinas-gdb-helpers-ccf-demo.pptx](outputs/asterinas-gdb-helpers-ccf-demo.pptx)（含视频演示流程页） |
| 源代码 | [PR #3412](https://github.com/asterinas/asterinas/pull/3412) 和 [gdb-helpers-foundation 分支](https://github.com/zevorn/asterinas/tree/gdb-helpers-foundation) |
| 自动化证据 | [gdb-smoke-test 成功记录](https://github.com/asterinas/asterinas/actions/runs/28657288013/job/84989356187) |
| 5 分钟视频 | 按以下讲稿与录制步骤补充 |

## 7. 五分钟讲稿

> 建议正常语速讲解，并在“现场演示”段同步录屏。命令执行和切换终端的时间已包含在时长内。

### 0:00–0:35｜方案介绍

各位评委好，我们是搭把手战队。本次作品是 Asterinas 的 GDB helpers 内核调试方案。Asterinas 是一个使用 Rust 编写、追求 Linux 兼容性的内核。它可以通过 QEMU 的 GDB server 进行底层调试，但开发者过去仍要手工理解 Rust 的 Arc、锁、集合和内核对象布局。我们的目标不是替代 GDB，而是在不改变内核运行时行为的前提下，让开发者能更直接、更可靠地理解正在运行的 Asterinas。

### 0:35–1:20｜真实问题与需求来源

这套工具不是从抽象需求开始，而来自一次真实排障。我们在进程加载阶段检查 ELF 的 PT_INTERP 和动态链接器，发现构建时链接到的 Rust 动态库版本与 OS rootfs 中实际加载的版本不一致。符号名看上去能匹配，真实地址背后的函数已经偏移，最后触发 trap，用户进程收到 Segmentation Fault。进一步追踪时，需要穿过动态链接、进程加载、trap handler、内核对象和 Rust 类型系统。真正耗时间的不是某一条命令，而是不断重复的对象导航、类型解包和日志整理。

### 1:20–2:15｜实现策略

因此，我们将能力按投入回报分成三层。第一层先处理 Rust wrapper 的显示：Atomic、Mutex、RwMutex 和 SpinLock 直接显示为开发者能读懂的值，降低 UnsafeCell 等内部细节带来的噪声。第二层解决对象导航，提供 ast_process、ast_thread、ast_pid_table 和 ast_file_table 等 convenience function，把 GDB 直接带到目标对象。第三层只保留真正需要聚合的 ast 命令，例如进程表、线程表、进程树、文件描述符和运行时间。这样，导航交给 helper，展示仍交给 GDB，pretty-printer 专注降低噪声。

### 2:15–3:10｜架构与可维护性

在架构上，cargo osdk debug 会自动使用 rust-gdb，并加载 asterinas-gdb.py。入口脚本注册 Rust 标准库和 Asterinas 自定义的 printer、function 与 command。底层的 gdb_bridge 封装 GDB Python API；layout 统一读取 Arc、锁和集合；kernel 知道如何从 PID 表走到进程、线程和文件表；commands 只做参数解析与表格输出。符号名、类型名和其他布局敏感常量集中在 constants 中，与 Rust 结构耦合的位置明确标记，这样后续内核演进时，helper 的维护边界是清楚的。

### 3:10–4:05｜验证策略

调试脚本最怕的不是崩溃，而是内核布局变化后继续运行却给出错误结果。因此，我们新增了真实 QEMU 加真实 GDB 的 smoke test。测试启动 Asterinas，等待 GDB socket，在内核入口和首次 syscall handler 设置断点，确保 PID 1 已建立。之后自动检查 pretty-printer、关键全局符号、对象定位函数和 ast 命令。只有输出 SMOKE: all ok 才通过。当前 PR 的独立 gdb-smoke-test GitHub Actions 已成功完成，说明这条完整链路可以在自动化环境重复验证。

### 4:05–4:45｜现场演示

现在演示实际使用。左侧终端已经执行 cargo osdk run 并打开 GDB server。右侧运行 cargo osdk debug 后，可以看到 helpers 自动加载。首先运行 ast-version 确认目标；接着运行 ast-ps，直接得到 PID、父 PID、状态、线程数和进程名。运行 ast-threads 和 ast-pstree 可以看线程及进程层级。对 PID 1 运行 ast-fds 1，可以看到文件描述符、flags 和类型。最后运行 ast-uptime，并用 ast_thread 打印线程字段，可以看到 wrapper 已被 pretty-printer 简化显示。

### 4:45–5:00｜总结

这套 helpers 把“会用 GDB”推进到“能看懂 Asterinas 的运行状态”。它保持零侵入，能自动加载，且有真实内核与 GDB 的 CI 验证。当前实现以 #3412 PR 和 #3094 RFC 公开，后续将继续扩展到调度、内存、锁竞争和网络诊断。谢谢各位评委。

## 8. 团队既有 Asterinas 贡献台账

本节汇总两位成员在 <code>asterinas/asterinas</code> 的公开记录，作为团队持续参与社区的背景支撑；本作品的主体仍是上文的 GDB helpers。

| 贡献者 | 成果 | 状态与价值 |
| --- | --- | --- |
| 邵志航 | [#3086](https://github.com/asterinas/asterinas/pull/3086) 修复 <code>pread</code>/<code>pwrite</code> 定位 I/O 错误码 | 已合并，2026-04-09；修复管道、<code>O_PATH</code> 与 nsfs 的 Linux 兼容语义，并补充回归测试 |
| 邵志航 | [#3117](https://github.com/asterinas/asterinas/pull/3117) 抑制 <code>O_PATH</code> fd 的 fsnotify 事件 | 已合并，2026-06-23；补齐 <code>FMODE_NONOTIFY</code> 语义与 inotify 回归测试 |
| 泽文 | [#3120](https://github.com/asterinas/asterinas/pull/3120) 映射 CPU 异常到 fault signals | 已合并，2026-04-14；修复 Trap Flag / <code>int3</code> 触发的内核 panic，并增加 <code>SIGTRAP</code> 回归测试 |
| 泽文 | [#3062](https://github.com/asterinas/asterinas/pull/3062) README 作为 crate-level Rust doc | 已合并，2026-03-22；消除 <code>ostd</code> 与 <code>int-to-c-enum</code> 的文档重复维护 |
| 泽文 | [#3412](https://github.com/asterinas/asterinas/pull/3412) GDB helpers | 开放 PR；6 个提交、25 个文件、+2167 / -9；为本作品主成果 |
| 泽文 | [#3094](https://github.com/asterinas/asterinas/issues/3094) GDB helpers RFC | 开放 Issue；记录非侵入式方案和社区讨论 |

截至核验日，<code>zevorn</code> 在该上游仓库有 2 个已合并 PR、1 个开放 PR、1 个自发 Issue / RFC，以及 4 个进入主线的提交；<code>BattiestStone4</code> 有 2 个已合并 PR、5 个进入主线的提交。除 #3094 的跟进评论外，未发现 <code>zevorn</code> 在该仓库提出的其他公开 Issue 或 PR review，因此不将无关互动重复计为独立成果。

## 9. 提交清单与开源托管

赛题支持“文件 + 链接（文件不超过 150 MB）”或“仅提交压缩包（不超过 1 GB）”。必含成果包括 5 分钟内作品展示视频、方案文档 / PPT 和源代码；所有成果必须托管至 GitLink 平台开源。

| 必交材料 | 当前内容 | 提交前动作 |
| --- | --- | --- |
| 方案文档 | 本 README | 已完成 |
| PPT | [asterinas-gdb-helpers-ccf-demo.pptx](outputs/asterinas-gdb-helpers-ccf-demo.pptx)（含视频演示流程页） | 已完成 |
| 5 分钟视频 | 第 7 节讲稿与第 6 节录制流程 | 录制并导出视频 |
| 源代码 | PR #3412 和实现分支 | 同步到 GitLink 公开项目 |
| 开源托管 | 本 GitHub 仓库用于协作镜像 | **必须再同步到 GitLink**；GitHub 单独托管不满足赛题资格要求 |

推荐采用“文件 + 链接”模式：将方案文档 / PPT / 5 分钟视频打包为不超过 150 MB 的附件，并在提交页填写 GitLink 公开仓库链接和上游 PR 链接。

## 10. 参考链接

- [技术叙事参考：星绽 OS 的 GDB Helper：从定位断错误到调试用的 Debug Workflow](https://mp.weixin.qq.com/s/mntHv8Ax0SXcTksX1xiKxA)
- [Asterinas 上游仓库](https://github.com/asterinas/asterinas)
- [GDB helpers PR #3412](https://github.com/asterinas/asterinas/pull/3412)
- [GDB helpers RFC #3094](https://github.com/asterinas/asterinas/issues/3094)
- [GDB helpers 实现分支](https://github.com/zevorn/asterinas/tree/gdb-helpers-foundation)
- [gdb-smoke-test 成功记录](https://github.com/asterinas/asterinas/actions/runs/28657288013/job/84989356187)
