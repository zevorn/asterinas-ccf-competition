# 第八届 CCF 开源创新大赛｜Asterinas（星绽）开源社区贡献

本仓库是搭把手战队的参赛材料仓库。主作品是一套面向 Asterinas Rust 内核的 GDB Helpers 调试方案；同时整理了两位成员在 Asterinas 上游的公开贡献。

## 作品信息

| 项目 | 内容 |
| --- | --- |
| 战队 | 搭把手战队（来自进程使命） |
| 队长 | 泽文（GitHub: [zevorn](https://github.com/zevorn)） |
| 队员 | 邵志航（GitHub: [BattiestStone4](https://github.com/BattiestStone4)） |
| 参赛项目 | [Asterinas](https://github.com/asterinas/asterinas) —— Rust 编写的 Linux 兼容内核 |
| 作品主题 | Asterinas GDB Helpers：可自动加载、可脚本化、可回归测试的内核调试方案 |
| 材料核验日期 | 2026-07-10 |

## 本作品核心上游提交

RFC 讨论后，调试信号语义与 GDB Helpers 被拆成两条独立、可分别审查的 PR。它们是本作品最直接的上游提交。

| PR | 状态 | 内容与作用 |
| --- | --- | --- |
| [#3120](https://github.com/asterinas/asterinas/pull/3120) | 已合并，2026-04-14 | 将 x86 的 <code>Debug</code>、<code>BreakPoint</code> 异常映射为 <code>SIGTRAP</code>，补齐 LoongArch 的 fault-signal 转换，并添加 <code>int3</code> / 单步回归测试。该兼容性修复从 RFC 中独立出来，避免与脚本实现混合评审。 |
| [#3412](https://github.com/asterinas/asterinas/pull/3412) | 开放 PR | GDB Helpers 主体：自动加载、Rust wrapper printer、内核对象导航、<code>ast-*</code> 聚合命令、冒烟测试与 CI。共 6 个提交、25 个文件、+2167 / −9。 |

> 状态说明：#3412 目前尚未合并。本仓库只陈述已经实现和验证的能力，不将开放 PR 写成 Asterinas 主线功能。

同时，队员邵志航的两项已合并 Linux 兼容性修复也纳入本次贡献材料，详见后文“其他上游贡献”。

## 方案概述

这套方案来自一次真实的进程加载段错误排查。Asterinas 处理 ELF 的 <code>PT_INTERP</code> 段时，需要把动态链接器映射进用户地址空间。排查发现，构建时链接的 Rust 动态库与 rootfs 实际加载的版本不一致：符号名仍可匹配，函数地址已经偏移，随后在 interpreter 和依赖库加载路径中触发 trap。

问题不只在于能否连上 GDB，而在于调试 Rust 内核时的重复成本：

- Rust 的符号、泛型和 wrapper 展开冗长，真正的业务状态被 <code>UnsafeCell</code>、锁实现等细节淹没；
- 定位进程、线程或文件表，需要手工穿过 PID 表、<code>Mutex</code>、<code>BTreeMap</code>、<code>Arc</code>、<code>Weak</code> 与 <code>Option</code>；
- 获取进程表、线程、fd、进程树与运行时间时，开发者反复做相同的遍历和汇总。

RFC 讨论后，方案收敛为三层高投入回报能力：

| 层次 | 实现 | 解决的问题 |
| --- | --- | --- |
| 降噪 | wrapper-type pretty-printer | 把原子类型、<code>Mutex</code>、<code>RwMutex</code>、<code>SpinLock</code> 等内部展开压缩成可读值。 |
| 导航 | GDB convenience function | 把复杂的对象定位链路封装为 <code>$ast_process(PID)</code>、<code>$ast_thread(TID)</code>、<code>$ast_file_table(PID)</code> 等变量。 |
| 聚合 | <code>ast-*</code> 命令 | 仅处理单个 <code>p</code> 表达式无法完成的集合遍历、表格汇总与树形展示。 |

因此，helper 解决“如何到达对象”，原生 GDB 继续负责“如何观察任意字段”。开发者不需要为每个新增内核字段等待新的专用命令。

### 自动加载与实现结构

<code>cargo osdk debug</code> 使用 <code>rust-gdb</code>，并自动加载 <code>scripts/gdb/asterinas-gdb.py</code>。用户不必手工维护脚本路径或 <code>.gdbinit</code>。

实现按以下模块分层：

| 模块 | 职责 |
| --- | --- |
| <code>asterinas-gdb.py</code> | 入口，注册 Rust 标准库与 Asterinas helper。 |
| <code>gdb_bridge.py</code> | 封装 GDB Python API、命令/函数注册、参数、表格和错误输出。 |
| <code>layout.py</code>、<code>printers.py</code> | 解析 Rust wrapper 和集合布局，注册 pretty-printer。 |
| <code>kernel.py</code> | 导航 PID 表、进程、线程、文件表和 jiffies。 |
| <code>commands.py</code> | 处理用户命令参数与格式化输出。 |
| <code>constants.py</code> | 集中管理符号名、类型名和布局敏感常量。 |

Rust 与 Python 的耦合点以 <code>COUPLED</code> 注释标记。配合真实 QEMU + GDB 冒烟测试，可在内核布局或 Rust 工具链变化后及时发现脚本失效，而不是让它悄悄输出错误信息。

## 调试接口与演示结果

| 接口 | 用途 |
| --- | --- |
| <code>ast-version</code> | 显示 Asterinas 内核版本，确认 helper 已连接。 |
| <code>ast-ps [PID]</code> | 输出 PID、PPID、状态、线程数和名称。 |
| <code>ast-threads</code> | 输出线程 ID、所属进程和名称。 |
| <code>ast-pstree</code> | 显示进程父子关系。 |
| <code>ast-fds PID</code> | 显示文件描述符、flags 和文件类型。 |
| <code>ast-uptime</code> | 显示内核运行时间。 |
| <code>$ast_pid_table()</code>、<code>$ast_process(PID)</code>、<code>$ast_thread(TID)</code>、<code>$ast_file_table(PID)</code> | 返回可继续与原生 <code>p</code> 组合的 GDB 对象。 |

冒烟测试启动真实 QEMU、连接真实 <code>rust-gdb</code>，验证 printer、对象导航与全部 <code>ast-*</code> 命令。通过标志为 <code>SMOKE: all ok</code> 和 <code>[smoke] PASSED</code>；[PR #3412 的 gdb-smoke-test](https://github.com/asterinas/asterinas/actions/runs/28657288013/job/84989356187) 于 2026-07-03 成功完成。

配套的 [Asterinas Debugger Agent](https://github.com/zevorn/asterinas-debugger) 和 [GDB Helper Agent Workflow](https://github.com/zevorn/asterinas-debugger/blob/main/skills/asterinas-debugger/references/workflows.md) 在这些稳定的 GDB 原语之上组织会话、断点、观察和探针步骤，用于复用证据驱动的调试 workflow；它不改变 Asterinas 内核，也不替代开发者判断。

## 交付物

| 交付物 | 位置或链接 | 说明 |
| --- | --- | --- |
| 正式方案文档 | [RFC 定稿中文版](docs/gdb-helpers-rfc-final-zh.md) | 依据 RFC #3094 最终讨论结论整理的正式中文方案。 |
| 5 分钟讲稿 | [5-minute-demo-script.md](docs/5-minute-demo-script.md) | 4 分钟 PPT + 1 分钟演示的逐页讲稿与录屏节奏。 |
| 演示 PPT | [asterinas-gdb-helpers-ccf-demo.pptx](outputs/asterinas-gdb-helpers-ccf-demo.pptx) | 含“来自进程使命”标注和最后的演示页。 |
| 演示录屏 | [asterinas-gdb-helpers-demo-recording.mp4](videos/asterinas-gdb-helpers-demo-recording.mp4) | 原始录屏的压缩副本；提交前可按比赛时长继续裁剪或加速。 |
| 源代码 | [PR #3412](https://github.com/asterinas/asterinas/pull/3412)；[gdb-helpers-foundation 分支](https://github.com/zevorn/asterinas/tree/gdb-helpers-foundation) | 上游实现及完整提交历史。 |
| Agent Workflow | [Asterinas Debugger Workflows](https://github.com/zevorn/asterinas-debugger/blob/main/skills/asterinas-debugger/references/workflows.md) | 将 GDB 会话、观察、断点和探针组合为可重复的调试闭环。 |
| 自动化证据 | [gdb-smoke-test 成功记录](https://github.com/asterinas/asterinas/actions/runs/28657288013/job/84989356187) | 真实 QEMU + GDB 测试记录。 |

## 其他上游贡献

### 邵志航：已合并的 Linux 兼容性修复

#### [PR #3086：修复 <code>pread</code> / <code>pwrite</code> 定位 I/O 错误处理](https://github.com/asterinas/asterinas/pull/3086)

已于 2026-04-09 合并，3 个提交、11 个文件、+138 / −26。该工作修复 Asterinas 与 Linux 在定位 I/O 错误码上的偏差：

- 管道的 <code>pread</code> / <code>pwrite</code> 原先先做权限检查而返回 <code>EBADF</code>；Linux 先检查可定位 I/O 能力，因此应优先返回 <code>ESPIPE</code>；
- <code>O_PATH</code> fd 在零长度或空 iov 的 <code>preadv</code> / <code>pwritev</code> 中会绕过 <code>read_at</code> / <code>write_at</code>，错误返回成功；修复后返回 <code>EBADF</code>；
- nsfs 能支持定位 I/O、但不支持 <code>lseek</code>，旧抽象无法表达这一差异；新增 <code>check_positional_io()</code> 默认方法，仅由 nsfs 覆写。

PR 同时新增管道 <code>ESPIPE</code>、<code>O_PATH</code> <code>EBADF</code>、nsfs <code>EINVAL</code> 和 <code>/dev/full</code> <code>ENOSPC</code> 的回归测试。

#### [PR #3117：抑制 <code>O_PATH</code> fd 的 fsnotify 事件](https://github.com/asterinas/asterinas/pull/3117)

已于 2026-06-23 合并，2 个提交、3 个文件、+104 / −13。Linux 会在 <code>O_PATH</code> fd 上设置 <code>FMODE_NONOTIFY</code>，从而不产生 fsnotify 事件；Asterinas 原先会把它当作普通 fd，错误触发 <code>IN_OPEN</code> / <code>IN_CLOSE_NOWRITE</code>。

修复复用 <code>FileLike::status_flags()</code> 判断 <code>O_PATH</code>，在 <code>on_open</code> / <code>on_close</code> 中通过共享的 <code>notifiable_path</code> helper 抑制通知。<code>on_access</code> / <code>on_modify</code> 无需额外处理，因为 <code>O_PATH</code> 的空 <code>Rights</code> 会先使读写以 <code>EBADF</code> 失败。新增 <code>inotify_o_path</code> 回归测试，验证 <code>O_PATH</code> 打开没有事件、普通打开仍有事件。

### 两位成员的公开 PR 与 Issue 台账

以下按 GitHub author 查询 [asterinas/asterinas](https://github.com/asterinas/asterinas) 的公开记录；“关联 Issue”不等同于该成员创建 Issue，表中只统计作者本人提交的 PR 与 Issue。

| 贡献者 | PR | Issue | 说明 |
| --- | --- | --- | --- |
| 邵志航（BattiestStone4） | [#3086](https://github.com/asterinas/asterinas/pull/3086)、[#3117](https://github.com/asterinas/asterinas/pull/3117)，均已合并 | 未检索到作者本人创建的独立 Issue | 两项工作沿着 <code>O_PATH</code> 与文件描述符 Linux 兼容性主线展开。 |
| 泽文（zevorn） | [#3062](https://github.com/asterinas/asterinas/pull/3062)、[#3120](https://github.com/asterinas/asterinas/pull/3120) 已合并；[#3412](https://github.com/asterinas/asterinas/pull/3412) 开放 | [#3094](https://github.com/asterinas/asterinas/issues/3094) 开放 RFC | #3062 将 <code>ostd</code> 与 <code>int-to-c-enum</code> 的 README 嵌入 crate-level Rust 文档，消除重复维护；#3120 是本作品拆分出的兼容性修复。 |

## 提交与 GitLink 同步说明

比赛要求包含作品展示视频、方案文档或 PPT 和源代码，并要求成果在 GitLink 平台开源。本仓库先作为 GitHub 协作与材料归档仓库；GitLink 同步由战队手动完成。

同步时请确认：

1. 将本仓库的文档、PPT 与视频一并推送到 GitLink 公开仓库；
2. 将 GDB Helpers 源代码所在的分支或等价代码镜像也设为公开可访问；
3. 替换或更新演示视频后，保证最终提交附件符合比赛的时长和体积限制；
4. 在比赛提交页填写 GitLink 仓库链接、上游 PR 链接及材料链接。

## 参考链接

- [Asterinas 上游仓库](https://github.com/asterinas/asterinas)
- [GDB Helpers RFC #3094](https://github.com/asterinas/asterinas/issues/3094)
- [GDB Helpers PR #3412](https://github.com/asterinas/asterinas/pull/3412)
- [SIGTRAP 兼容性修复 PR #3120](https://github.com/asterinas/asterinas/pull/3120)
- [定位 I/O 错误修复 PR #3086](https://github.com/asterinas/asterinas/pull/3086)
- [<code>O_PATH</code> fsnotify 修复 PR #3117](https://github.com/asterinas/asterinas/pull/3117)
- [技术叙事参考：星绽 OS 的 GDB Helper](https://mp.weixin.qq.com/s/mntHv8Ax0SXcTksX1xiKxA)
