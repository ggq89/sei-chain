# Sei Chain EVM 功能概览

Sei 的 EVM 是**深度集成**方案——在 Cosmos SDK 存储层上运行 go-ethereum 的 EVM 解释器，实现 CosmWasm 与 EVM 的完全互操作。

## 架构简图

```mermaid
graph TD
    User["用户 / DApp"]

    User -->|JSON-RPC| EVMRPC["evmrpc/<br/>HTTP + WebSocket"]
    User -->|Cosmos TX| Ante["ante/ 路由"]
    User -->|CosmWasm| WB["wasmbinding/<br/>CosmWasm ↔ EVM 绑定"]

    EVMRPC --> Keeper["x/evm/keeper<br/>EVM 执行引擎"]
    Ante --> Keeper
    WB -->|MsgInternalEVMCall| Keeper

    Keeper --> StateDB["x/evm/state/<br/>StateDB 桥接"]
    Keeper --> Precompiles["precompiles/<br/>Cosmos → EVM 接口"]
    Keeper --> Artifacts["x/evm/artifacts/<br/>Pointer 合约"]
```

---

## 1. `x/evm/` — EVM 核心模块

Cosmos SDK 的 EVM 模块，将完整的以太坊虚拟机集成到 Sei 链中。

### 核心子包

| 子包 | 说明 |
|------|------|
| `keeper/` | EVM 执行引擎、地址管理、余额、nonce、receipt、pointer 合约、fee 管理（40+ 文件） |
| `state/` | 在 Cosmos store 上实现 go-ethereum 的 `StateDB` 接口（usei/wei 双精度余额、snapshot/revert、journal、transient storage） |
| `types/` | Protobuf 消息（`MsgEVMTransaction`、`MsgInternalEVMCall` 等）、参数、错误码、key 定义、链配置 |
| `types/ethtx/` | 以太坊交易类型（Legacy、EIP-2930 AccessList、EIP-1559 DynamicFee、EIP-7702 SetCode） |
| `ante/` | EVM 专用 ante handler 链（路由、签名验证、nonce 校验、fee 验证、gas 计量、Cosmos 字段拒绝等） |
| `artifacts/` | 内嵌 pointer 合约字节码（CW20/CW721/CW1155/ERC20/ERC721/ERC1155/native/wsei） |
| `migrations/` | 存储迁移（base fee、EIP-1559 参数、bloom、pointer 升级等，15+ 迁移） |
| `config/` | EVM 链配置（ChainConfig、硬分叉参数） |
| `derived/` | 派生数据结构 |
| `querier/` | gRPC 查询服务 |
| `replay/` | 以太坊区块重放配置 |
| `blocktest/` | 以太坊 blocktest 测试框架配置 |
| `client/` | CLI 与 WASM 绑定 |

### 核心特性

- **双地址模型**：Sei (bech32) ↔ EVM (hex) 双向映射，支持显式关联和默认 cast 地址
- **支持的交易类型**：Legacy、EIP-2930、EIP-1559、EIP-7702 (SetCode)、Associate（Sei 专有）
- **EIP-1559 动态 base fee**：逐块根据 gas 使用量调整
- **Pointer 合约**：CW20↔ERC20、CW721↔ERC721、CW1155↔ERC1155 双向互操作
- **Deferred processing**：bloom filter、fee surplus、错误 receipt 在 EndBlock 汇总
- **Synthetic receipts**：跨 CW/EVM 调用产生的合成 receipt 和日志

---

## 2. `evmrpc/` — EVM JSON-RPC 服务

实现以太坊兼容的 JSON-RPC API 服务器（HTTP + WebSocket）。

### 关键文件

| 文件 | 说明 |
|------|------|
| `server.go` | HTTP/WS 服务器初始化，注册所有 RPC namespace |
| `block.go` | `eth_getBlockBy*` 系列 |
| `tx.go` | `eth_getTransaction*`、`eth_getTransactionReceipt` |
| `send.go` | `eth_sendRawTransaction` |
| `filter.go` | `eth_getLogs`、`eth_newFilter` 等日志过滤 |
| `state.go` | `eth_getBalance`、`eth_getCode`、`eth_getStorageAt` |
| `simulate.go` | `eth_call`、`eth_estimateGas` |
| `tracers.go` | `debug_traceTransaction`、`debug_traceBlockByNumber` 等 |
| `subscribe.go` | WebSocket 订阅（`eth_subscribe`） |
| `bloom.go` | Bloom filter 相关 |
| `association.go` | Sei/EVM 地址关联查询 |
| `info.go` | `eth_chainId`、`eth_gasPrice` 等 |
| `txpool.go` | `txpool_*` 端点 |
| `net.go` | `net_*` 端点 |
| `web3.go` | `web3_*` 端点 |
| `sei_legacy.go` | `sei_*`/`sei2_*` 旧版端点（含 Cosmos 交易可见性） |
| `worker_pool.go` | RPC 请求的 worker pool 并发控制 |
| `watermark_manager.go` | 区块高度可用性管理 |
| `config/` | RPC 服务器配置（端口、超时、worker 池大小） |
| `stats/` | RPC 调用追踪统计 |

### 关键特性

- **即时最终性**：无 `pending` 概念，`pending` 等同于 `latest`
- **`sei_*` 端点**：可见 EVM + Cosmos synthetic receipt 交易
- **`sei2_*` 端点**：区块中包含 bank transfer
- **Legacy API 网关**：通过 `app.toml` 白名单控制，已标记为废弃
- **Debug tracing**：忠实重放历史执行
- **不支持**：uncle/trie/PoW/blob 等以太坊特性

---

## 3. `precompiles/` — EVM 预编译合约

将 Cosmos SDK 原生功能暴露给 EVM 调用者。每个预编译都有 Solidity 接口、ABI 定义、Go 实现和版本控制。

### 注册的预编译

| 预编译 | 路径 | 说明 |
|--------|------|------|
| **Bank** | `precompiles/bank/` | EVM 中访问 Cosmos bank 模块（转账、余额查询） |
| **Staking** | `precompiles/staking/` | EVM 中质押/解质押/重委托操作 |
| **Gov** | `precompiles/gov/` | EVM 中提案投票 |
| **Distribution** | `precompiles/distribution/` | EVM 中领取质押奖励 |
| **Oracle** | `precompiles/oracle/` | EVM 中查询预言机数据 |
| **IBC** | `precompiles/ibc/` | EVM 中发起 IBC 跨链转账 |
| **Wasmd** | `precompiles/wasmd/` | EVM 中调用 CosmWasm 合约 |
| **Addr** | `precompiles/addr/` | Sei ↔ EVM 地址转换 |
| **Pointer** | `precompiles/pointer/` | 注册/管理 pointer 合约 |
| **PointerView** | `precompiles/pointerview/` | 查询 pointer 合约信息 |
| **JSON** | `precompiles/json/` | EVM 中的 JSON 解析工具 |
| **P256** | `precompiles/p256/` | P-256 (secp256r1) 签名验证 |
| **Solo** | `precompiles/solo/` | Solo 预编译 |

### 架构模式

- 统一基类 `precompiles/common/precompiles.go` 实现 `vm.PrecompiledContract`
- `precompiles/setup.go` 负责所有预编译的初始化和注册
- `precompiles/utils/expected_keepers.go` 定义 keeper 接口
- 每个预编译通过 `GetVersioned()` 支持按区块高度分版本

---

## 4. `app/` 中的 EVM 集成

| 文件 | 说明 |
|------|------|
| `app/precompiles.go` | `PrecompileKeepers` 结构体——将所有 Cosmos keeper 包装为预编译所需的接口 |
| `app/eth_replay.go` | 以太坊区块重放引擎——从以太坊主网拉取区块并在 Sei 上重放执行，逐交易验证余额和状态 |
| `app/abci.go` | ABCI 层 EVM 交易处理——提取 EVM nonce/sender/txhash，构建 `EvmTxInfo` |
| `app/ante.go` | Ante handler 路由——区分 EVM 和 Cosmos 交易走不同处理链 |
| `app/receipt.go` | Receipt 处理和存储 |

---

## 5. 其他 EVM 相关

| 路径 | 说明 |
|------|------|
| `contracts/` | Hardhat 项目——EVM 合约源码、测试、部署脚本 |
| `integration_test/evm_module/` | EVM 模块集成测试 |
| `wasmbinding/` | CosmWasm ↔ EVM 绑定（查询和消息） |

---

## 6. EVM 交易排序与执行流程

EVM 交易的完整生命周期：

1. 用户通过 JSON-RPC 发送 `eth_sendRawTransaction`
2. 原始以太坊交易被包装为 Cosmos 的 `MsgEVMTransaction` 消息
3. 经过 **Tendermint/CometBFT 共识层**排序，进入区块
4. 在 `DeliverTx` 阶段路由到 `x/evm/keeper` 的 `EVMTransaction()` 执行

```mermaid
sequenceDiagram
    participant User as 用户/DApp
    participant RPC as evmrpc (JSON-RPC)
    participant TM as Tendermint 共识层
    participant Ante as ante/ 路由
    participant EVM as x/evm/keeper

    User->>RPC: eth_sendRawTransaction
    RPC->>TM: 包装为 MsgEVMTransaction
    TM->>TM: 共识排序，打包进区块
    TM->>Ante: DeliverTx
    Ante->>EVM: EVMTransaction()
    EVM->>EVM: go-ethereum EVM 解释器执行
    EVM-->>TM: 返回执行结果
```

---

## 7. EVM 数据存储架构

### 7.1 EVM Transaction — Tendermint Block Store

EVM 交易作为 `MsgEVMTransaction`（Cosmos TX）存储在 **Tendermint Block Store** 中，**不独立存储**。查询交易时通过 receipt 中的 txHash 定位到 Tendermint 区块中的原始交易。

### 7.2 EVM Chain State — Cosmos KV Store

所有 EVM 链状态存储在 EVM 模块的 Cosmos KV Store（store key = `"evm"`）中。前缀定义在 `x/evm/types/keys.go`：

| 前缀 | 数据 | Key 格式 | Value |
|------|------|---------|-------|
| `0x01` | EVM→Sei 地址映射 | `evmAddr (20B)` | Sei bech32 地址 bytes |
| `0x02` | Sei→EVM 地址映射 | `seiAddr` | EVM 地址 (20B) |
| `0x03` | 合约存储槽 | `address (20B) + slotHash (32B)` | value (32B) |
| `0x07` | 合约字节码 | `address (20B)` | 原始字节码 |
| `0x08` | 代码哈希 | `address (20B)` | keccak256 hash (32B) |
| `0x09` | 代码大小 | `address (20B)` | uint64 (8B) |
| `0x0a` | Nonce | `address (20B)` | uint64 (8B) |
| `0x0d` | BlockBloom | 覆盖式写入 | 当前块合并 bloom |
| `0x15` | Pointer 注册 | ERC20/721/1155 | pointer 合约地址 |
| `0x1b` | BaseFeePerGas | — | 当前基础费 |
| `0x1c` | NextBaseFeePerGas | — | 下一块基础费 |

**余额例外**：余额**不在 EVM store 中**，而是通过 **Bank 模块**存储，采用双精度设计：

- `usei`（6 位小数）：Cosmos 原生，存在 Bank 模块
- `wei`（额外 12 位小数）：`1 usei = 10^12 swei`，通过 BankKeeper 扩展方法管理
- 对外展示 18 位小数（swei），兼容以太坊应用

**桥接层**：`x/evm/state/DBImpl` 实现了 go-ethereum 的 `vm.StateDB` 接口，将 EVM 读写映射到上述 Cosmos KV Store。

### 7.3 EVM Block — 不独立存储，实时派生

EVM 区块**没有独立存储**。JSON-RPC 返回的以太坊格式区块由 `evmrpc/block.go` 从 Tendermint 区块实时转换：

| EVM 字段 | 来源 |
|---------|------|
| `number` | `block.Block.Height` |
| `hash` | Tendermint `BlockID.Hash` |
| `parentHash` | `Block.LastBlockID.Hash` |
| `miner` | `Block.ProposerAddress` |
| `timestamp` | `Block.Time` |
| `baseFeePerGas` | EVM store `0x1c` |
| `gasUsed` | 累计各 receipt 的 GasUsed |
| `logsBloom` | EVM store `0x0d` (块级合并 bloom) |
| `difficulty/nonce/uncles` | 固定为空值（不适用） |

唯一持久化的区块级数据是 **BlockBloom**（前缀 `0x0d`），存在 EVM KV Store 中。

### 7.4 EVM TX Receipt — 独立 Receipt Store

Receipt 存储最复杂，分三层架构：

```
┌────────────────────────────────────────────┐
│  Layer 1: Transient Store（块内临时）         │
│  交易执行后 WriteReceipt → SetTransientReceipt │
│  生命周期：单个区块内                           │
├────────────────────────────────────────────┤
│  Layer 2: 内存缓存（cachedReceiptStore）      │
│  最近数百块的 receipt                         │
├────────────────────────────────────────────┤
│  Layer 3: 持久化 Receipt Store               │
│  后端：PebbleDB（默认）或 Parquet             │
│  目录：{home}/data/receipt.db               │
│  Key：ReceiptKeyPrefix(0x0b) + txHash       │
└────────────────────────────────────────────┘
```

**Receipt 生命周期**：

1. **交易执行时** → `SetTransientReceipt()` 写入 transient store
2. **EndBlock** → `FlushTransientReceipts()` 遍历 transient store，批量写入持久化 Receipt Store
3. **查询时** → 内存缓存 → PebbleDB/Parquet → fallback 到 legacy Cosmos KV Store

### 7.5 存储总结

| 数据 | 存储位置 | 独立 DB？ |
|------|---------|----------|
| **EVM Transaction** | Tendermint Block Store（作为 Cosmos TX） | 否 |
| **EVM Chain State** | Cosmos KV Store（`"evm"` module），余额在 Bank 模块 | 否 |
| **EVM Block** | 不存储，从 Tendermint 区块实时派生 | — |
| **EVM TX Receipt** | 独立 Receipt Store（PebbleDB/Parquet） | **是**，`data/receipt.db` |

---

## 8. evmrpc 查询架构

### 8.1 evmrpc 不是独立进程

evmrpc HTTP/WS 服务器运行在 **seid 进程内部**，通过 goroutine 启动，直接共享进程内的 keeper、Tendermint 客户端、Cosmos 状态等对象，**不经过任何网络调用**。

启动链路：`seid` 启动 → `App.RegisterTendermintService()` → `evmrpc.NewEVMHTTPServer()` → 等待第一个区块 EndBlock 信号后开始监听 8545 端口。

核心依赖注入：

| 参数 | 类型 | 作用 |
|------|------|------|
| `tmClient` | `rpcclient.Client` | Tendermint RPC 客户端（进程内调用） |
| `keeper` | `*keeper.Keeper` | EVM keeper，直接内存访问 |
| `ctxProvider` | `func(int64) sdk.Context` | 按高度获取 SDK 上下文 |
| `app` | `*baseapp.BaseApp` | 用于 DeliverTx 重放 |
| `stateStore` | `types.StateStore` | SeiDB 状态存储 |

### 8.2 ctxProvider 机制

`ctxProvider` 是统一的状态访问抽象，传入区块高度，返回指向对应 IAVL 快照的 `sdk.Context`：

- **`height = -1`（latest）**：返回 CheckTx 上下文（最新已提交状态）
- **具体高度**：调用 `app.CreateQueryContext(height, false)`，创建指向该历史高度的只读快照上下文

### 8.3 查询 EVM Transaction（`eth_getTransactionByHash`）

```mermaid
flowchart LR
    A["eth_getTransactionByHash(hash)"] --> B{mempool 中?}
    B -->|是| C["tmClient.UnconfirmedTxs()\n遍历匹配 hash"]
    B -->|否| D["keeper.GetReceipt(hash)\n得到 blockNumber + txIndex"]
    D --> E["tmClient.Block(blockNumber)\n从 TM 区块定位原始交易解码"]
```

**数据源**：Receipt Store（定位交易位置）+ Tendermint Block Store（获取原始交易数据），都是进程内直接访问。

### 8.4 查询 EVM TX Receipt（`eth_getTransactionReceipt`）

```mermaid
flowchart LR
    A["eth_getTransactionReceipt(hash)"] --> B["keeper.GetReceipt(hash)"]
    B --> C["内存缓存"]
    C -->|miss| D["PebbleDB / Parquet"]
    D -->|miss| E["legacy Cosmos KV Store"]
    B --> F["tmClient.Block(blockNumber)\n补全 ante 失败交易的 from/to 等字段"]
```

**数据源**：Receipt Store（主要）+ Tendermint Block Store（补全信息）。

### 8.5 查询 EVM Chain State（`eth_getBalance` / `eth_getCode` / `eth_getStorageAt` / `eth_call`）

```mermaid
flowchart TD
    A1["eth_getBalance"] --> CTX["ctxProvider(height)\nSDK Context → IAVL 快照"]
    A2["eth_getCode"] --> CTX
    A3["eth_getStorageAt"] --> CTX
    A4["eth_call"] --> CTX

    CTX --> S["state.NewDBImpl(ctx, keeper)"]
    S --> KV["Cosmos KV Store (SeiDB)"]

    A1 -.->|余额| BANK["BankKeeper\nusei + wei"]
    A4 -.->|模拟执行| EVM["go-ethereum EVM 解释器"]
```

**数据源**：全部通过 Cosmos KV Store（SeiDB/IAVL），`ctxProvider` 提供对应高度的快照上下文。`eth_call` 额外调用 go-ethereum 的 EVM 解释器做模拟执行。

### 8.6 查询 EVM Block（`eth_getBlockByNumber` / `eth_getBlockByHash`）

```mermaid
flowchart LR
    A["eth_getBlockByNumber(height)"] --> B["tmClient.Block(height)\nTendermint Block Store"]
    B --> C["EncodeTmBlock() 实时转换"]
    C --> D["遍历区块内 TX"]
    D --> E["keeper.GetReceipt()\n获取 gasUsed / bloom"]
    C --> F["keeper (EVM store)\n获取 baseFeePerGas"]
```

**字段映射**：`number` ← Block.Height, `hash` ← BlockID.Hash, `miner` ← ProposerAddress, `baseFeePerGas` ← EVM store `0x1c`, `gasUsed` ← 累计 receipt.GasUsed, `logsBloom` ← OR 合并各 receipt bloom。

### 8.7 高度一致性保障：WatermarkManager

由于区块数据、链状态、receipt 分别存在不同的存储后端，写入进度可能不一致。`WatermarkManager` 取三者的**最小高度**作为安全的"最新可查高度"，避免查询到尚未完全同步的数据。

### 8.8 架构总图

```
┌─────────────────── seid 进程 ───────────────────────┐
│                                                      │
│  evmrpc Server (goroutine, :8545/:8546)              │
│  ├─ StateAPI ──────→ ctxProvider → keeper → KV Store │
│  ├─ TransactionAPI → keeper.GetReceipt → Receipt DB  │
│  │                   tmClient.Block → Block Store     │
│  ├─ BlockAPI ──────→ tmClient.Block + GetReceipt     │
│  ├─ SimulationAPI ─→ state.DBImpl → geth EVM 执行    │
│  ├─ FilterAPI ─────→ keeper logs + tmClient blocks   │
│  └─ DebugAPI ──────→ app.DeliverTx 重放执行          │
│       │                    │                │        │
│       ▼                    ▼                ▼        │
│  ┌─────────┐    ┌──────────────┐   ┌──────────────┐ │
│  │ Cosmos  │    │  Tendermint  │   │ Receipt Store│ │
│  │ KV Store│    │  Block Store │   │ (PebbleDB)   │ │
│  │ (SeiDB) │    │              │   │ receipt.db   │ │
│  └─────────┘    └──────────────┘   └──────────────┘ │
│                                                      │
│  全部进程内直接访问，无网络开销                          │
└──────────────────────────────────────────────────────┘
```

---

## 9. Mempool 中的 EVM 交易处理

### 9.1 双层数据结构

Mempool 为 EVM 交易维护了专门的 `evmQueue`，定义在 `sei-tendermint/internal/mempool/priority_queue.go`：

```go
type TxPriorityQueue struct {
    txs      []*WrappedTx            // priority heap（最大堆，按优先级排序）
    evmQueue map[string][]*WrappedTx // 按地址分组，每组按 nonce 升序排列
}
```

- **`txs`（heap）**：跨地址按 `priority` 降序排列，每个 EVM 地址**只有最低 nonce 的交易**在 heap 中
- **`evmQueue`**：`map[evmAddress] -> []*WrappedTx`，同地址交易按 nonce 升序排列

三条不变量：
1. 同一地址不能有重复 nonce
2. 同一地址不能有 nonce 间隔（gap）
3. 每个地址队列的头部（最低 nonce）必须在 heap 中

### 9.2 EVM 元数据注入

EVM ante handler（`x/evm/ante/sig.go`）验证签名后，将 EVM 元数据注入 context，最终通过 `CheckTx` 响应传递给 mempool：

```go
// CheckTx 响应中的 EVM 字段
EVMNonce         uint64
EVMSenderAddress string
IsEVM            bool
Priority         int64   // = EffectiveGasPrice / PriorityNormalizer
```

### 9.3 Nonce 路由逻辑

CheckTx 阶段根据 nonce 分三种情况：

| 情况 | 处理 |
|------|------|
| `txNonce < nextNonce` | **拒绝**（已过期 nonce） |
| `txNonce == nextNonce` | **正常**进入 mempool 主队列 |
| `txNonce > nextNonce` | 标记为 **PendingTransaction**，进入 pending 存储等待前置 nonce 就位 |

### 9.4 插入 mempool 主队列

正常交易进入 `priorityIndex.PushTx()` 时的处理逻辑：

| 场景 | 处理 |
|------|------|
| 该地址首笔交易 | 同时进入 heap + evmQueue |
| nonce < 队首 | 替换 heap 中的代表交易 |
| nonce > 队首 | 仅入 evmQueue（不进 heap） |
| 同 nonce 已存在 | priority 更高则替换（类以太坊 gas price bumping） |

### 9.5 Pending 交易异步激活

nonce 有 gap 的交易进入 `pendingTxs` 存储，附带 `PendingTxChecker` 回调。回调逻辑评估：

1. `nextNonceToBeMined` — 链上确认的下一个 nonce
2. `nextPendingNonce` — 考虑 mempool 中所有已知交易后的下一个可用 nonce

当 `txNonce < nextPendingNonce`（前置 nonce 全部就位）→ 交易被 **Accepted**，从 pending 移入主 mempool。

### 9.6 出块选择（`ReapMaxBytesMaxGas`）

```mermaid
flowchart TD
    A["从 heap 弹出最高优先级交易"] --> B{是 EVM 交易?}
    B -->|是| C["该地址下一个 nonce\n自动补入 heap"]
    B -->|否| D["直接收集"]
    C --> E["收集到 evmTxs"]
    D --> F["收集到 nonEvmTxs"]
    E --> G["返回 evmTxs + nonEvmTxs\nEVM 交易排在前面"]
    F --> G
```

### 9.7 出块后更新（`Update`）

1. 移除已出块交易（EVM 通过 `EvmTxInfo.Nonce + Address` 匹配）
2. `handlePendingTransactions()` → 评估每个 PendingTxChecker → Accepted 的移入主队列
3. 过期清理（TTL 时间 + 区块高度）

### 9.8 关键特性总结

| 特性 | 说明 |
|------|------|
| **双层结构** | heap（跨地址优先级）+ evmQueue（同地址 nonce 序） |
| **Nonce 顺序保证** | 每个地址只有最低 nonce 进 heap，弹出后下一个自动补入 |
| **同 nonce 替换** | 更高 gas price 可替换已有同 nonce 交易 |
| **Future nonce 处理** | nonce 有 gap 的交易进 pendingTxs，前置 nonce 就位后自动激活 |
| **出块优先** | EVM 交易排在 Cosmos 交易之前 |
| **优先级计算** | `priority = EffectiveGasPrice / PriorityNormalizer` |

---

## 10. EVM Block 与 Sei Chain Block 的关系

### 10.1 核心结论：完全一致，没有分开

EVM **不是**独立的链或独立的区块生产者，它只是 Cosmos SDK 模块中嵌入的执行引擎。EVM block 和 Sei chain block 是**同一个东西**。

| 维度 | 说明 |
|------|------|
| **区块高度** | EVM block number **等于** Tendermint block height，一一对应 |
| **区块哈希** | EVM block hash **等于** Tendermint BlockID.Hash |
| **出块节奏** | 完全跟随 Tendermint 共识出块，没有独立的 EVM 出块周期 |
| **区块存储** | EVM 没有独立的区块存储，查询时从 Tendermint 区块实时转换 |
| **交易排序** | EVM TX 和 Cosmos TX 在同一个 Tendermint 区块中排序（EVM TX 排在前面） |

代码依据（`evmrpc/block.go`）：

```go
number := big.NewInt(block.Block.Height)                        // EVM block number = TM block height
blockhash := common.HexToHash(block.BlockID.Hash.String())      // EVM block hash = TM block hash
miner := common.HexToAddress(block.Block.ProposerAddress.String()) // miner = TM proposer
```

### 10.2 执行模型

Sei 的 EVM 不是像 Optimism 或 Arbitrum 那样的独立执行层/Rollup，而是嵌入在 Cosmos 应用层中：

```mermaid
graph TD
    TM["Tendermint 共识"] --> Block["一个区块 (height=N)"]
    Block --> EVMTxs["EVM TXs（排在前面）"]
    Block --> CosTxs["Cosmos TXs（排在后面）"]
    EVMTxs --> EVMKeeper["x/evm/keeper\n(go-ethereum EVM)"]
    CosTxs --> CosMods["其他 Cosmos 模块\n(bank/staking/...)"]
    EVMKeeper --> Store["同一个 Cosmos KV Store (SeiDB)"]
    CosMods --> Store
```

- **一个共识**：Tendermint/CometBFT
- **一个区块序列**：Tendermint blocks，EVM 和 Cosmos 交易混在同一个区块中
- **一个出块节奏**：Tendermint 的出块间隔（Sei 主网约 400ms）
- **一个状态树**：所有模块共享 SeiDB，EVM 状态只是其中 store key = `"evm"` 的一个子树
