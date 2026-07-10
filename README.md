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

因此，GDB helpers 把最耗注意力、最重复的动作沉淀成稳定的调试基础设施。

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

## 7. 5 分钟讲稿（4 分钟 PPT + 1 分钟演示）

> 第 1–6 页只讲 4 分钟；第 7 页切到录屏，用 1 分钟完成命令演示。下面的时间是剪辑与讲解的总时长，不录 QEMU 启动等待。

### 第 1 页｜0:00–0:30｜开场

大家好，我是搭把手战队队长泽文，队员是邵志航。我们参加第八届 CCF 开源创新大赛“星绽开源社区贡献”赛题。参赛作品是 Asterinas GDB helpers：一条可自动加载、可脚本化的内核调试流程，也能接入 Agent 和 Workflow 做固定检查。

### 第 2 页｜0:30–1:15｜问题从一次段错误开始

方案来自一次进程加载段错误。Asterinas 处理 ELF 的 `PT_INTERP` 时，需要映射动态链接器。我们发现构建时的 Rust 动态库和 rootfs 中实际加载的版本不一致：符号名还能对上，函数地址已经偏移，最终触发了 trap。

随后排查要跨动态链接、进程加载、trap handler 和 Rust 结构，反复解符号、穿锁和集合、从 PID 表找对象。于是我们把这次排查沉淀成了 GDB helper。

### 第 3 页｜1:15–1:55｜先做三层最常用的能力

和上游讨论后，我们先做三层：`wrapper_type` 和 pretty-printer 压低 Rust wrapper 噪声；convenience function 从 PID 表导航到进程、线程和文件表；`ast-*` 命令聚合进程、线程、FD、进程树和运行时间。

先把重复动作固定下来，开发者仍能用原生 GDB 看任何字段。

### 第 4 页｜1:55–2:50｜实现与自动加载

执行 `cargo osdk debug` 后，OSDK 会使用 `rust-gdb` 并加载 `asterinas-gdb.py`，用户不用自己找脚本或维护 `.gdbinit`。

底层 `gdb_bridge.py` 收口 GDB Python API；`layout.py` 和 `printers.py` 处理 Rust 类型；`kernel.py` 与 `commands.py` 管理内核对象和命令，`constants.py` 集中布局敏感常量。Agent 和 Workflow 复用这个入口，自动跑固定检查。

### 第 5 页｜2:50–3:30｜让脚本能长期维护

Python helper 会读到和 Rust 结构强耦合的数据。我们在耦合点加 `COUPLED` 标记，并用真实 QEMU + GDB smoke test 检查 printer、对象导航和 `ast-*`。

只有输出 `SMOKE: all ok` 才通过；PR #3412 的 CI 已成功跑通。

### 第 6 页｜3:30–4:00｜阶段成果

这套 helpers 不改原生调试行为，却把 Rust 类型、内核对象和高频状态放到更短的路径上。底层对象仍可用原生 GDB 继续展开。

下面用一分钟看真实内核上的命令输出。

### 第 7 页｜4:00–5:00｜作品演示

录屏从已经启动的 QEMU 和已打开的 GDB 开始，保留命令与关键输出，避免录入启动等待。

| 时间 | 操作 | 旁白 |
| --- | --- | --- |
| 4:00–4:08 | 执行 `cargo osdk debug` | 这里可以看到 helpers 随 debug 命令自动加载，不需要额外配置。 |
| 4:08–4:16 | 执行 `ast-version` | 先确认当前内核与 helper 已经连接成功。 |
| 4:16–4:29 | 执行 `ast-ps`、`ast-pstree` | 进程表和父子关系不再需要从 PID 表手工遍历。 |
| 4:29–4:40 | 执行 `ast-threads` | 这里直接查看线程与所属进程。 |
| 4:40–4:52 | 执行 `ast-fds 1` | PID 1 的文件描述符、flags 和文件类型会被格式化输出。 |
| 4:52–5:00 | 执行 `ast-uptime` | 最后读取内核运行时间。到这里，常见的内核状态已经能通过一组稳定命令快速拿到。 |

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
