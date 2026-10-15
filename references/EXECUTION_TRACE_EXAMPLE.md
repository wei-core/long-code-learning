# 执行追踪示例：把源码变成运行中的事件序列

> 示例使用对话中给出的简化 `Sequence` 类，不代表任何特定 vLLM 官方版本或完整调度器实现。所有数值均为**手工推演**，不是运行日志。

## 输入与执行目标

```python
seq = Sequence([10, 11, 12, 13, 14, 15], block_size=4)
seq.append_token(101)
result = seq.block(1)
```

**目标不是手写 `Sequence`，而是看到这三条语句时，清楚它们内部依次发生什么。**

## 初始调用时间线

| 时刻 | 触发位置 | 真正执行的动作 | 对象状态/结果 |
|---|---|---|---|
| T0 | `Sequence(...)` | 进入 `__init__`，读取列表和 `block_size` | `token_ids` 输入长度为 6 |
| T1 | `self.block_size = block_size` | 保存块容量 | `block_size=4` |
| T2 | `next(Sequence.counter)` | 获取下一个 ID | 假设本进程首次创建，`seq_id=0` |
| T3 | `SequenceStatus.WAITING` | 设置初始状态 | `status=WAITING` |
| T4 | `copy(token_ids)` | 将输入列表浅拷贝为内部列表 | `token_ids=[10,11,12,13,14,15]`，不与外部列表共用同一个 list |
| T5 | `self.token_ids[-1]` | 读取末尾元素 | `last_token=15` |
| T6 | `len(self.token_ids)` 两次 | 分别初始化总长度与 Prompt 长度 | `num_tokens=6`, `num_prompt_tokens=6` |
| T7 | 初始化缓存及映射 | 元数据初始值 | `num_cached_tokens=0`, `block_table=[]` |
| T8 | 采样参数赋值 | 把采样配置字段保存到实例 | 参数值以传入的 `SamplingParams` 为准 |

到这里**只创建了对象**；不等于 GPU 执行过 Prefill，也不等于 KV Cache 已分配。

## append_token(101) 内部逐步变化

| 时刻 | 执行语句 | 执行前 | 执行后 |
|---|---|---|---|
| T9 | `self.token_ids.append(token_id)` | `[10,11,12,13,14,15]` | `[10,11,12,13,14,15,101]` |
| T10 | `self.last_token = token_id` | `15` | `101` |
| T11 | `self.num_tokens += 1` | `6` | `7` |

没有出现在该函数中的属性**不会自动更新**：`num_prompt_tokens=6`, `num_cached_tokens=0`, `block_table=[]`。

## block(1) 的执行分支

1. 先调用 `num_blocks` 属性：`ceil(7 / 4) = 2`。
2. 检查 `assert 0 <= 1 < 2`：条件为真。
3. 再比较 `i == num_blocks - 1`：`1 == 1`，进入最后一个块的分支。
4. 求 `last_block_num_tokens = 7 - (2-1)*4 = 3`。
5. 执行 `self.token_ids[-3:]` 得到 `[14,15,101]`。
6. `result=[14,15,101]`；没有真正分配或者读取物理 GPU KV Block。

## 提取学习者应当能够回答的问题

- 哪一行让 Token 列表增长？哪一行让 `num_tokens` 增长？两者何时一致？
- 为什么 `block(1)` 返回三个 Token，而不是恰好四个？
- `append_token` 是否代表新 Token 的 K/V 已计算并写入显存？**不能如此推出。**
- 如果 `__setstate__` 在 Decode 阶段只恢复最后一个 Token，随后直接调用 `block(0)` 会发生什么？需要结合接收端真实持有的属性和逻辑值分析，不能把逻辑历史 Token 数误当成实际列表长度。

## 复用方法

对任何长函数都可以沿用“**选定具体输入 → 找到真实入口 → 列出执行顺序 → 跟踪变量、内存和状态 → 解释分支 → 标记外部未知依赖**”。尤其适合 Scheduler 的逐轮调度、CUDA kernel 的每线程访存和 MLIR Pass 的逐步 Rewrite。