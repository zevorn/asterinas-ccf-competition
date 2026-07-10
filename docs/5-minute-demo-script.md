# Asterinas GDB Helpers 五分钟展示讲稿

> 对应 [演示 PPT](../outputs/asterinas-gdb-helpers-ccf-demo.pptx)。
>
> 时间分配：PPT 约 4 分钟，录屏演示 1 分钟。录制素材见 [演示录屏](../videos/asterinas-gdb-helpers-demo-recording.mp4)。

## 第 1 页｜0:00–0:25｜开场

大家好，我是搭把手战队的队长泽文，队员是邵志航。我们来自进程使命，参加第八届 CCF 开源创新大赛“星绽开源社区贡献”赛题。

本次作品是基于 Asterinas 和 GDB Helpers 的便携式内核调试方案。我们还配套了 Agent 和 Workflow，用于把重复的调试步骤组织成自动化流程。

## 第 2 页｜0:25–1:05｜问题从一次段错误开始

方案来自一次真实的进程加载段错误排查。Asterinas 处理 ELF 的 <code>PT_INTERP</code> 时需要映射动态链接器。我们发现，编译时链接的 Rust 动态库与 rootfs 实际加载的版本不一致：符号名还能对上，但函数地址已经偏移，最终在加载 interpreter 和依赖库时触发了 trap。

排查要跨动态链接、进程加载、trap handler 和内核对象。普通 GDB 调试 Rust 时，符号和包装类型展开很长；从 PID 表找到一个进程或线程，也要穿过多层数据结构。这些重复动作会打断验证假设的节奏。

## 第 3 页｜1:05–1:40｜三层实现策略

上游 RFC 讨论后，我们没有追求命令数量，而是先做三层高 ROI 能力。

第一层是 wrapper printer，压缩原子类型和锁的内部噪声。第二层是 convenience function，把从 PID 表定位进程、线程和文件表的复杂链路封装成 GDB 变量。第三层才是 <code>ast-*</code> 命令，只保留进程表、线程、文件描述符和进程树这类需要遍历或汇总的能力。

这样，helper 解决导航，GDB 原生 <code>p</code> 仍然可以查看任何字段。

## 第 4 页｜1:40–2:30｜实现与自动加载

开发者执行 <code>cargo osdk debug</code> 后，OSDK 会使用 <code>rust-gdb</code> 并自动加载 <code>asterinas-gdb.py</code>，不需要手工维护脚本路径或 <code>.gdbinit</code>。

底层由 <code>gdb_bridge.py</code> 封装 GDB Python API；<code>layout.py</code> 和 <code>printers.py</code> 处理 Rust 类型；<code>kernel.py</code> 和 <code>commands.py</code> 管理内核对象与命令；<code>constants.py</code> 集中布局敏感信息。Agent Workflow 则在这些稳定接口上组织自动化调试步骤。

## 第 5 页｜2:30–3:15｜长期维护

GDB 脚本会读取 Rust 结构布局，所以只写出来还不够。我们在 Rust 与 Python 的耦合点添加 <code>COUPLED</code> 注释，并把符号名、类型名和常量集中管理。

同时，冒烟测试会启动真实 QEMU，连接真实 GDB，检查 printer、对象导航和 <code>ast-*</code> 命令。只有输出 <code>SMOKE: all ok</code> 才通过；PR #3412 的独立 gdb-smoke-test 已成功跑通。

## 第 6 页｜3:15–4:00｜阶段成果

这套 Helpers 不改变原有调试方式，而是缩短了从“我怀疑这里有问题”到“我拿到内核状态”的路径。

为了让审查边界清晰，和 GDB 相关的 <code>SIGTRAP</code> 兼容性修复已拆成独立的 PR #3120 并合入主线；GDB Helpers 主体以 PR #3412 持续接受上游审查。下面用一分钟展示已经跑在真实内核上的调试接口。

## 第 7 页｜4:00–5:00｜录屏演示

录屏先展示已经等待连接的 QEMU 和已连接的 GDB。

1. 在内核早期断点处，用实际变量展示 <code>Atomic&lt;bool&gt;</code>、<code>Mutex&lt;T&gt;</code> 等 wrapper 的简化输出；
2. 继续运行到 shell 就绪后的观察点，执行 <code>ast-version</code>，确认 helper 已加载；
3. 依次展示 <code>ast-ps</code>、<code>ast-pstree</code>、<code>ast-threads</code>，快速读取进程和线程状态；
4. 执行 <code>ast-fds 1</code>，展示 PID 1 的文件描述符、flags 和文件类型；
5. 以 <code>ast-uptime</code> 收尾，说明常用内核状态已经可以通过稳定命令直接获得。

旁白可以简短收束为：这些命令没有替代原生 GDB；它们把最重复、最容易分散注意力的部分收敛成了可复用的调试基础设施。
