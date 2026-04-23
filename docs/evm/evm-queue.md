# evmQueue 机制说明

> 相关文档：[功能概览](evm-overview.md) | [交易生命周期](evm-tx-lifecycle.md)

## 1. 背景

在 Sei 的共识层 mempool 中，EVM 交易不能只按全局优先级排序。

原因是同一 EVM 地址的交易存在严格 nonce 依赖：

- nonce 更小的交易未执行前，nonce 更大的交易不能先执行
- 但不同地址之间又希望按 fee priority 竞争出块顺序

为同时满足这两点，Sei 在 `TxPriorityQueue` 中引入了 `evmQueue`。

## 2. 数据结构

代码位置：`sei-tendermint/internal/mempool/priority_queue.go`

```go
type TxPriorityQueue struct {
    txs []*WrappedTx // 全局 priority heap

    // EVM 专用队列：按地址分桶，每桶按 nonce 升序
    evmQueue map[string][]*WrappedTx
}
```

其中 `WrappedTx` 中的 EVM 元数据来自 CheckTx 返回值：

- `evmAddress`
- `evmNonce`
- `isEVM`

定义位置：`sei-tendermint/internal/mempool/tx.go`

## 3. evmQueue 的核心目的

### 3.1 保证同地址 nonce 顺序

`evmQueue[address]` 内部永远按 nonce 升序，出块只能从队首开始推进。

### 3.2 保证跨地址优先级竞争

全局 `txs` heap 仍按优先级排序，但每个地址只把“当前可执行”的队首 tx 放进 heap。

这样不同地址可以竞争，单地址又不会 nonce 乱序。

### 3.3 支持同 nonce 替换

同地址同 nonce 新交易如果 priority 更高，可替换旧交易（类以太坊 gas bump）。

## 4. 三条不变量

源码注释中给出了 3 条不变量：

1. 同地址队列中不能有重复 nonce
2. 同地址队列中不能有 nonce gap
3. 每个地址队列头部必须在 heap 中

这 3 条决定了 `evmQueue` 不是普通缓存，而是“可执行性索引”。

## 5. 插入逻辑

入口：`PushTx()` -> `pushTxUnsafe()`

### 5.1 非 EVM 交易

直接进入全局 heap。

### 5.2 EVM 交易

- 若地址首次出现：
  - 初始化 `evmQueue[address] = [tx]`
  - 同时 push 到 heap
- 若地址已存在：
  - 按 nonce 插入该地址队列
  - 只有当新 tx 比当前队首 nonce 更小，才替换 heap 代表项
  - 否则仅留在 `evmQueue`，等待前置 nonce 被消费

### 5.3 同 nonce 冲突

`tryReplacementUnsafe()` 会检查同地址同 nonce：

- 新 tx priority 更高：替换旧 tx
- 否则丢弃新 tx

## 6. 弹出与推进逻辑

入口：`PopTx()` -> `popTxUnsafe()`

- 非 EVM：从 heap 弹出即完成
- EVM：
  1. 从 heap 弹出地址队首
  2. 同时从 `evmQueue[address]` 删除该队首
  3. 若该地址还有后续 nonce，把新的队首 push 回 heap

这一步实现了“同地址 nonce 链式推进”。

## 7. 与 Reap 出块的关系

`ReapMaxBytesMaxGas` 最终通过 `priorityIndex.ForEachTx()` 读取交易。

由于 `ForEachTx()` 内部也是“临时 pop 再 reenqueue”，EVM 交易在遍历期间仍遵守：

- 单地址 nonce 顺序
- 跨地址优先级排序

最终收集阶段会把 EVM tx 和 non-EVM tx 分开收集，并返回时把 EVM tx 放前面。

## 8. 与 pendingTxs 的区别

`evmQueue` 只解决“主队列内已可竞争交易”的 nonce 顺序问题。

`pendingTxs` 解决的是另一类问题：交易 nonce 太大（future nonce），暂时不可执行。

- future nonce tx 会进入 `pendingTxs`
- 当前置 nonce 就位后，经 `PendingTxChecker` 评估 Accepted，再进入主 mempool
- 进入主 mempool 后才受 `evmQueue` 管理

可理解为两级机制：

1. `pendingTxs`：是否具备进入竞争队列资格
2. `evmQueue`：进入竞争队列后如何保证可执行顺序

## 9. 设计收益

- 防止同地址 nonce 乱序执行
- 不牺牲跨地址 fee priority 竞争
- 支持同 nonce 提价替换
- 与现有 heap 结构兼容，复杂度可控

## 10. 一句话总结

`evmQueue` 的本质是“按地址维护 nonce 可执行链”，并把每条链的头部暴露给全局优先级 heap，使 Sei 能同时满足 EVM nonce 语义和 mempool 优先级调度。