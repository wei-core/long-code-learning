# Example: vLLM Sequence 逐行学习计划

**注意**：本文件以对话中给出的 `Sequence` 实现为示例，不表示 vLLM 官方所有版本一致。

## 1. 对象边界

- `Sequence`：保存单条生成请求的 token 历史与元数据，并暴露供其他模块使用的辅助接口。
- 它不直接执行注意力计算，不直接调用 GPU Kernel，不直接保存 KV Tensor。
- Scheduler、缓存管理、ModelRunner 如何使用这些字段，必须由调用方代码确认。

## 2. 在收到原始代码后建立真实行号索引

以方法顺序为准：

| 分段 | 内容 | 必须回答 |
|---|---|---|
| S01 | imports + `SequenceStatus` | `Enum`, `auto`, `count`, `copy` 是什么；三态如何定义 |
| S02 | `counter` + `__init__` | 参数含义、对象 ID、复制、token 计数、采样配置、缓存元数据 |
| S03 | `__len__` + `__getitem__` | Python 特殊方法何时被调用、与 `num_tokens` 和列表下标的关系 |
| S04 | `is_finished`、`num_completion_tokens`、`prompt_token_ids`、`completion_token_ids` | `@property`、切片边界、prompt/completion 分界 |
| S05 | `num_cached_blocks`、`num_blocks`、`last_block_num_tokens` | 向上取整与 Block 容量、0/整除/非整除边界 |
| S06 | `block(i)` | 断言、最后一个部分块与其他块的分支、切片索引 |
| S07 | `append_token` | 三项状态更新，哪些字段不会更新 |
| S08 | `__getstate__` | 序列化 5 元组、Prefill 与 Decode 传输内容不同的原因 |
| S09 | `__setstate__` | 反序列化分支、只恢复最后一个 Token 时能做什么不能做什么 |

## 3. 一组贯穿全程的具体数据

初始化：

```python
seq = Sequence(token_ids=[10,11,12,13,14,15], block_size=4)
```

已知可推导：

| 字段/属性 | 初值 |
|---|---|
| `num_tokens` | 6 |
| `num_prompt_tokens` | 6 |
| `num_completion_tokens` | 0 |
| `last_token` | 15 |
| `num_blocks` | 2 |
| `last_block_num_tokens` | 2 |
| `num_cached_tokens` | 0 |
| `num_cached_blocks` | 0 |
| `block_table` | `[]` |
| `block(0)` | `[10,11,12,13]` |
| `block(1)` | `[14,15]` |

追加：

```python
seq.append_token(101)
```

产生：`num_tokens=7`，`num_prompt_tokens=6`，`num_completion_tokens=1`，`last_token=101`，`num_blocks=2`，`last_block_num_tokens=3`。但 `num_cached_tokens` 和 `block_table` **不会因这个函数自动变更**。

## 4. 最值得单独讲的异常/边界

1. `__init__(sampling_params=SamplingParams())` 在函数定义时构造默认对象，后续实例复用；这里读取字段为主，但需知道共享默认对象的风险。
2. `num_cached_blocks=ceil(num_cached_tokens/block_size)` 在缓存 Token 不是 block_size 整倍数时向上取整，不能据此断言该物理 Block 已被写满。
3. `block(i)` 访问的是 `token_ids` 逻辑切片，不是真正 GPU 上 KV 数据。
4. `__getstate__` 在 completion 数量为 0 时传完整 `token_ids`，否则只传 `last_token`。
5. `__setstate__` 在 Decode 时将 `token_ids` 设为单元素列表，但 `num_tokens` 仍为逻辑总长度。这意味着反序列化后的接收端状态**不等同于拥有完整历史 token_ids 的发送端对象**；此时 `prompt_token_ids`、`block(i)` 等依赖完整历史的接口不应想当然地使用。应检查接收端真实调用路径。
6. `__setstate__` 仅恢复五项字段及 `last_token`，没有恢复 `block_size`、`seq_id`、`status`、采样相关字段；是否安全取决于对象创建/使用方式，不能断言直接反序列化出的对象可以调用所有方法。
7. 空序列情况下 `num_blocks=0`，`last_block_num_tokens=0`，`last_token=None`；`block(0)` 应触发断言。

## 5. 互动式续讲示范

如果上一轮已经完成 `__init__`，下一轮直接开始 S03 或 S04，而不是从 `Enum` 重新解释：

> 本轮：`__len__`、`__getitem__`、`is_finished` 和 `num_completion_tokens`。先给完整原代码，再逐行解释，最后用 `seq[0]`、`len(seq)` 和 `seq.num_completion_tokens` 做状态测试。最后只出一道独立预测题。

这只是示例计划；当用户提供更大仓库时，需要重新映射真实源码，不可套用示例行号。