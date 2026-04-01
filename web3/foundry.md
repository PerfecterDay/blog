# foundry
{docsify-updated}

## 安装
```
curl -L https://foundry.paradigm.xyz | bash
foundryup
foundryup -i nightly //Install the latest nightly release
```

## 工具
| 工具   | 功能说明 |
|--------|----------|
| forge  | 构建、测试、调试、部署和验证智能合约 |
| anvil  | 运行本地以太坊开发节点，支持fork功能 |
| cast   | 与合约交互、发送交易、获取链上数据 |
| chisel | 快速 Solidity REPL，用于快速原型开发和调试 |


### forge
用于开发、构建合约、运行测试、部署合约等功能
```
# Create a new project called Counter
forge init Counter
cd Counter

forge build
forge test
forge test --fork-url https://reth-ethereum.ithaca.xyz/rpc


# Use forge scripts to deploy contracts
# Set your private key
export PRIVATE_KEY="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"

# Deploy to local anvil instance
forge script script/Counter.s.sol --rpc-url http://127.0.0.1:8545 --broadcast --private-key $PRIVATE_KEY
```

### anvil
Anvil 是一款快速的本地以太坊开发节点，非常适合在受控环境中测试智能合约及其他区块链工作流程。
```
anvil //启动一个开发节点
anvil --fork-url https://reth-ethereum.ithaca.xyz/rpc
```

### Cast
Cast 是命令行与以太坊应用交互的瑞士军刀。可以调用智能合约、发送交易，或检索任何类型的链上数据。
```
# Check ETH balance
cast balance vitalik.eth --ether --rpc-url https://reth-ethereum.ithaca.xyz/rpc
 
# Call a contract function to read data
cast call 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 \
"balanceOf(address)" 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 \
--rpc-url https://reth-ethereum.ithaca.xyz/rpc


# Set your private key
export PRIVATE_KEY="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"

# Send ETH to an address
cast send 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 --value 10000000 --private-key $PRIVATE_KEY


# Call JSON-RPC methods directly
cast rpc eth_getHeaderByNumber $(cast 2h 22539851) --rpc-url https://reth-ethereum.ithaca.xyz/rpc
 
# Get latest block number
cast block-number --rpc-url https://reth-ethereum.ithaca.xyz/rpc
```

### Chisel
Chisel 是一款快速、实用且功能强大的 Solidity REPL，专为快速原型开发和调试而设计。它非常适合测试 Solidity 代码片段，并交互式地探索合约行为。
```
// Create and query variables
➜ uint256 a = 123;
➜ a
Type: uint256
├ Hex: 0x7b
├ Hex (full word): 0x000000000000000000000000000000000000000000000000000000000000007b
└ Decimal: 123
 
// Test contract functions
➜ function add(uint256 x, uint256 y) public pure returns (uint256) { return x + y; }
➜ add(5, 10)
Type: uint256
└ Decimal: 15
```

## 实战
```

```
✅  [Success] Hash: 0x06d724130e66e52924f8e57e8de032bf7054d3ff3915c8bb84fa261b52fb01cb
Contract Address: 0x28935D3BE1d3593d055C28F1e72Dc3d98bd5DBcB
Block: 9198817
Paid: 0.000000157597535439 ETH (156813 gas * 0.001005003 gwei)


调用 `increment` ：
```
cast send 0x28935D3BE1d3593d055C28F1e72Dc3d98bd5DBcB "increment()" \
--private-key xxxxxxxxx \
--rpc-url https://sepolia.infura.io/v3/4b7ae60b751f4b48b238713e94358abd

blockHash            0x7decb431969dc7ac8c93c6e0e1710688554b23147d2eeae0f12f32207a14b398
blockNumber          9205620
contractAddress      
cumulativeGasUsed    13257860
effectiveGasPrice    1052918
from                 0x83D7c17ba0AFC43F05d6CCeeDb0396D0F8bE2a79
gasUsed              43482
logs                 []
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root                 
status               1 (success)
transactionHash      0x8774c9eba1672d1077662f43e9e6b180519cf9c8b1204dcc4ee63cedeffc004b
transactionIndex     91
type                 2
blobGasPrice         
blobGasUsed          
to                   0x28935D3BE1d3593d055C28F1e72Dc3d98bd5DBcB
```

然后再读取值：
```
cast call 0x28935D3BE1d3593d055C28F1e72Dc3d98bd5DBcB "number()(uint256)" \
--rpc-url https://sepolia.infura.io/v3/4b7ae60b751f4b48b238713e94358abd
```