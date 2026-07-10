# Asterinas GDB Helpers 实施方案（RFC 定稿中文版）

> 本文依据 [RFC #3094](https://github.com/asterinas/asterinas/issues/3094) 的讨论结论、对应的 [PR #3412](https://github.com/asterinas/asterinas/pull/3412) 和上游就绪性审查整理而成。
>
> 核验日期：2026-07-10。PR #3412 当前仍为开放状态；本文描述已完成的设计、实现与验证，不将其表述为已合入 Asterinas 主线。

## 1. 背景与问题

Asterinas 通过 QEMU 的 GDB stub 支持远程调试，但直接用 GDB 观察 Rust 内核时，开发者仍要反复完成三类工作：

1. 在冗长的 Rust 符号与类型展开中筛选真正有用的信息；
2. 从 PID 表穿过 <code>Mutex</code>、<code>BTreeMap</code>、<code>Arc</code>、<code>Weak</code>、<code>Option</code> 等包装层，才能找到进程、线程或文件表；
3. 手动汇总进程、线程、文件描述符和运行时间等调试状态。

方案的直接来源是一次进程加载段错误排查。Asterinas 处理 ELF 的 <code>PT_INTERP</code> 段时，需要将动态链接器映射到用户地址空间。排查发现，构建时链接的 Rust 动态库与 rootfs 实际加载的版本不一致；符号名仍能匹配，但函数地址已经偏移，随后在 interpreter 与依赖库加载路径中触发 trap。

这次排查跨越动态链接、进程加载、trap handler 和 Rust 内核对象。GDB Helpers 保留 GDB 的原生能力，同时把上述重复的导航和观察动作沉淀为可复用的基础设施。

## 2. RFC 的定稿原则

RFC 讨论后，初始实现按以下原则收敛。

| 原则 | 定稿结论 |
| --- | --- |
| 优先级 | 第一阶段只保留投入回报最高、维护成本最低的能力，不以命令数量为目标。 |
| 分工 | Helper 解决对象导航；GDB 原生 <code>p</code> 负责自由查看字段；pretty-printer 负责降低 Rust 包装层噪声。 |
| 命令边界 | 只有遍历集合、建立关联或汇总多对象信息的功能才做成 <code>ast-*</code> 命令。单个表达式能完成的事不再另造命令。 |
| 运行环境 | 以 Asterinas 开发容器内的 <code>rust-gdb</code> 为前提，复用其标准库 printer，不维护一套重复的标准库 DWARF 兜底实现。 |
| 可维护性 | “不侵入”不是绝对目标。只要能避免脚本静默失效，就允许在 Rust 耦合点增加明确标记，并通过测试约束两侧同步。 |

## 3. 范围拆分

RFC 中的调试信号语义和 GDB 脚本按审查意见拆成两项独立上游工作：

| 工作 | 状态 | 说明 |
| --- | --- | --- |
| [PR #3120](https://github.com/asterinas/asterinas/pull/3120) | 已合并，2026-04-14 | 将 x86 的 <code>Debug</code> 与 <code>BreakPoint</code> 映射为 <code>SIGTRAP</code>，并补齐 LoongArch 的 fault-signal 转换与 x86 回归测试。它是独立的 Linux 兼容性修复，不与 Python 脚本混合评审。 |
| [PR #3412](https://github.com/asterinas/asterinas/pull/3412) | 开放 PR | GDB Helpers 主体：自动加载、wrapper printer、内核对象导航、聚合命令、文档、冒烟测试和 CI。 |

这样做的结果是：内核可观察性工具和内核行为语义分别接受审查，问题边界更清楚。

## 4. 第一阶段的三层能力

### 4.1 降噪层：wrapper-type pretty-printer

不为 <code>Process</code>、<code>PidEntry</code> 等每个业务类型单独写 printer。GDB 已能依据 DWARF 递归显示这些字段；真正的噪声来自包装类型。

第一阶段为以下高频类型注册 printer：

| 类型 | 输出效果 |
| --- | --- |
| <code>AtomicBool</code>、<code>AtomicU8/U16/U32/U64/Usize</code> | 折叠 <code>UnsafeCell</code> 等内部层，直接显示标量值。 |
| <code>ostd::sync::Mutex&lt;T&gt;</code> | 显示锁状态和内部值。 |
| <code>ostd::sync::RwMutex&lt;T&gt;</code> | 显示可读的内部值。 |
| <code>ostd::sync::SpinLock&lt;T&gt;</code> | 显示可读的内部值。 |

<code>Arc</code>、<code>Vec</code>、<code>BTreeMap</code>、<code>Option</code>、<code>String</code> 等标准库类型交由 <code>rust-gdb</code> 已有的 printer 处理。这样，开发者继续使用 GDB 自带的 <code>p</code>、<code>bt full</code>、<code>info locals</code>、<code>display</code> 和 IDE 变量面板，也能同时获得更清晰的输出。

### 4.2 导航层：GDB convenience function

复杂之处在于“如何到达对象”，而不是“如何打印对象”。因此 helper 返回 <code>gdb.Value</code>，让用户把返回值继续组合进原生 GDB 表达式。

| 函数 | 返回对象 | 隐藏的导航链路 |
| --- | --- | --- |
| <code>$ast_pid_table()</code> | PID 表 | 全局符号 → <code>Mutex</code> |
| <code>$ast_process(PID)</code> | 进程对象 | PID 表 → <code>BTreeMap</code> → <code>Arc</code> / <code>Weak</code> |
| <code>$ast_thread(TID)</code> | 线程对象 | PID 表 → 线程数据下转型 |
| <code>$ast_file_table(PID)</code> | 进程文件表 | 进程 → task 集合 → <code>PosixThread</code> → 文件表 |

例如，开发者可以执行 <code>p *$ast_process(1)</code> 查看整个进程，也可以继续访问新加字段，而无需等 helper 为该字段新增命令。

### 4.3 聚合层：只保留原生 <code>p</code> 无法替代的命令

最终命令集只有六项：

| 命令 | 作用 |
| --- | --- |
| <code>ast-version</code> | 显示 Asterinas 内核版本，确认调试会话和 helper 已连接。 |
| <code>ast-ps [PID]</code> | 遍历 PID 表，输出进程摘要。 |
| <code>ast-threads</code> | 遍历活动线程并显示所属进程。 |
| <code>ast-pstree</code> | 构造父子关系并以树形显示进程。 |
| <code>ast-fds PID</code> | 遍历指定进程的文件表，显示 fd、flags 和文件类型。 |
| <code>ast-uptime</code> | 读取 jiffies 并格式化内核运行时间。 |

原型中仅包装 QEMU <code>monitor</code> 的内存查看、地址转换、物理内存导出或写入功能不进入第一阶段；这些功能要么已有原生入口，要么会扩大维护面，或带来不必要的写内存风险。

## 5. 加载路径与代码结构

用户通过 Asterinas 原有工作流启动调试：

~~~bash
# 终端一：启动 QEMU，并等待 GDB 客户端
cargo osdk run --gdb-server wait-client

# 终端二：连接目标
cargo osdk debug
~~~

<code>cargo osdk debug</code> 使用 <code>rust-gdb</code>，并在 workspace 中发现 <code>scripts/gdb/asterinas-gdb.py</code> 时自动执行 <code>source</code>。缺少 <code>rust-gdb</code> 时应直接失败，而不是悄悄回退为不完整的普通 GDB 环境。

实现按职责拆分，避免命令层直接依赖 Rust 布局细节：

| 模块 | 职责 |
| --- | --- |
| <code>scripts/gdb/asterinas-gdb.py</code> | 入口；注册 Rust 标准库及 Asterinas 自定义 helper。 |
| <code>helper/gdb_bridge.py</code> | 封装 GDB Python API、命令和函数注册、参数解析、表格与错误输出。 |
| <code>helper/layout.py</code> | 读取 Rust wrapper 与集合布局。 |
| <code>helper/printers.py</code> | 注册 wrapper-type pretty-printer。 |
| <code>helper/kernel.py</code> | 导航 PID 表、进程、线程、文件表和 jiffies。 |
| <code>helper/commands.py</code> | 仅处理用户命令的参数和格式化输出。 |
| <code>helper/constants.py</code> | 集中保存符号名、下转型类型名、线程名长度、计时频率和文件类型标记等布局敏感信息。 |

## 6. 维护约束与验证

Python helper 会读取 Rust 对象的实际布局，因此维护机制是方案的一部分。

1. Rust 侧耦合点添加 <code>COUPLED</code> 注释，明确指出关联的 helper 模块；
2. 所有硬编码的符号名、类型名和常量集中在 <code>constants.py</code>，避免分散在命令实现中；
3. GDB 冒烟测试启动真实 QEMU、连接真实 <code>rust-gdb</code>，而非用 mock 代替运行时对象；
4. CI 工作流同时监视 <code>scripts/gdb</code>、相关 Rust 结构、OSDK debug 入口与 <code>rust-toolchain.toml</code>。工具链更新也会触发 GDB 测试；
5. GDB 工作流使用与项目其余 CI 一致的 Asterinas 容器版本。

冒烟测试在内核映射完成后停在 <code>__ostd_main</code> 和首次 syscall 等可观察点，检查 printer 注册、对象导航函数和全部 <code>ast-*</code> 命令。已记录的完整链路以 <code>SMOKE: all ok</code> 与 <code>[smoke] PASSED</code> 为通过标志；[PR #3412 的 gdb-smoke-test](https://github.com/asterinas/asterinas/actions/runs/28657288013/job/84989356187) 于 2026-07-03 成功完成。

上游就绪性审查还明确了两条正确性要求：

- <code>ast-fds</code> 不能假设 task 集合的第一个元素仍持有文件表；当主线程退出而同组线程继续运行时，应遍历 task，返回首个仍为 <code>Some</code> 的文件表；
- 文件类型识别应优先匹配具体 socket 类型，避免通用 <code>Socket</code> 提前遮蔽更精确的类型名。

前者已体现在当前实现的遍历逻辑中；后者作为持续的上游审查点保留，避免为展示而牺牲信息准确性。

## 7. 配套 Agent 与 Workflow

GDB Helpers 是本作品的基础设施层。配套的 [Asterinas Debugger Agent 项目](https://github.com/zevorn/asterinas-debugger) 及其中的 [GDB Helper Agent Workflow 定义](https://github.com/zevorn/asterinas-debugger/blob/main/skills/asterinas-debugger/references/workflows.md) 在此之上组织调试会话、断点、观察和探针步骤，使重复排障可以按证据驱动的 workflow 执行。

该 Agent 项目不改变 Asterinas 内核，也不替代开发者判断；它复用稳定的 GDB 原语，把“启动会话—停在证据点—读取对象—验证假设”的流程固定下来。上游 PR #3412 的评审范围仍聚焦 GDB Helpers 本身。

## 8. 相关链接

- [Asterinas 上游仓库](https://github.com/asterinas/asterinas)
- [RFC #3094：Add GDB helper scripts for non-invasive kernel debugging](https://github.com/asterinas/asterinas/issues/3094)
- [RFC 讨论中的定稿回复](https://github.com/asterinas/asterinas/issues/3094#issuecomment-4228621863)
- [PR #3412：Add GDB helpers for kernel debugging](https://github.com/asterinas/asterinas/pull/3412)
- [PR #3120：Map CPU exceptions to fault signals on x86 and LoongArch](https://github.com/asterinas/asterinas/pull/3120)
- [GDB Helpers 实现分支](https://github.com/zevorn/asterinas/tree/gdb-helpers-foundation)
- [Asterinas Debugger Agent Workflow](https://github.com/zevorn/asterinas-debugger/blob/main/skills/asterinas-debugger/references/workflows.md)
