# Long Code Learning

**Read code. Trace execution. Understand the system. No need to write it from scratch.**

**面向源码阅读的 AI 学习 Skill：不以手写代码为目标，专注于看懂代码怎样一步步运行。**

## 为什么需要它？

在 AI 辅助开发越来越普遍的情况下，阅读现有代码、判断执行路径、理解数据如何流动、识别状态与资源何时变化，仍然是学习开源项目和排查问题的重要能力。

本项目不是代码生成器，也不是刷题式编程教程。它让 AI 扮演**代码执行过程讲解员**：拿到较长的源码后，先解释整个系统在哪里、各个模块怎样协作，再按照原始代码连续分段，**完整且不跳行**地展示它在一个具体输入下如何执行。

## 核心能力

- **架构先行**：说明文件/类/函数的职责、上下游调用、输入输出，以及“它不负责什么”。
- **运行过程优先**：按照真实调用顺序理解入口、调用栈、`if/else`、循环、返回值和副作用，区分代码书写顺序和执行顺序。
- **逐行零遗漏**：对长文件建立覆盖表，分轮次解释全部可见源码，避免突然省略中间函数或把概览当作精读。
- **具体数据推演**：用小规模可手算输入，列出每一步变量、张量、指针、内存、Block 或状态变化。
- **系统级数据流**：适用于 Python、C/C++、CUDA、Triton、vLLM、PyTorch、MLIR、调度器与缓存系统等。
- **交互式理解**：一次只检查一个“下一步会发生什么”的阅读问题；**不要求学习者手写代码**。
- **学习进度追踪**：记录已讲区段、未讲源码、外部依赖和待核对的疑点，下次从停止位置继续。

## 学习工作流

```text
输入完整源码 / 仓库路径
        │
        ▼
① 确认代码范围、版本、可见证据
        │
        ▼
② 理解系统定位、调用链、整体数据流
        │
        ▼
③ 建立文件/函数/行号覆盖索引
        │
        ▼
④ 逐段逐行分析：何时执行、谁执行、读写什么
        │
        ▼
⑤ 用具体输入推演分支、循环、状态与内存变化
        │
        ▼
⑥ 完整复原一次端到端运行过程
        │
        ▼
⑦ 阅读理解检查 + 保存学习进度
```

## 使用方式

将本仓库的 `SKILL.md` 以及 `references/`、`assets/` 一并放入支持 Agent Skills 的 AI 工具指定的 skills 目录；具体发现方式随客户端而异。也可以直接把 `SKILL.md` 的内容作为会话的教学指令提供给 AI。

示例提示词：

> 使用 Long Code Learning Skill 帮我学习下面的 vLLM `Sequence` 类。先说明它在推理系统中的位置、模块协作和数据流，再按原始代码顺序完整分段。每段逐行解释真实运行过程，用具体 Token 演示关键变量和 Block 如何变化。每次只问我一个运行预测问题，不要安排手写代码练习。讲过的代码不重复，未讲的代码不能跳过。

> 请按执行时序讲解这个 CUDA kernel：给出 grid/block/thread 和数据索引对应关系，推演每个线程读写的地址、同步点和最终输出。我只需要理解现有实现，不需要自己重写 CUDA。

## 仓库结构

```text
long-code-learning/
├── SKILL.md                             # Skill 核心指令
├── README.md                            # 项目说明与使用方法
├── references/
│   ├── TEACHING_DETAILS.md              # 分语言阅读与运行追踪规范
│   ├── SEQUENCE_WALKTHROUGH.md          # vLLM Sequence 学习计划
│   └── EXECUTION_TRACE_EXAMPLE.md       # 将代码转为事件时间线的完整示例
└── assets/
    └── PROGRESS_TEMPLATE.md             # 续讲与覆盖率记录模板
```

## 适用边界

本 Skill 用于**理解现有代码**，而非代替运行测试、源码证据或实际 profiling。手工模拟要标记为推演，外部未展示的函数要标记为待验证。需要写代码时可以另行提出需求，但不是本 Skill 的默认教学方式。

---

*Scope: code reading, execution tracing, control flow, data flow, state transitions, and system comprehension—not code-writing drills.*