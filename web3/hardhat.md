# hardhat
{docsify-updated}

## 项目结构与解释
```
hardhat.config.ts

contracts
├── Counter.sol
└── Counter.t.sol

test
└── Counter.ts

ignition
└── modules
    └── Counter.ts

scripts
└── send-op-tx.ts
```

+ `hardhat.config.ts` ：项目的主要配置文件。它定义了 Solidity 编译器版本、网络配置以及项目使用的插件和任务等设置。
+ `contracts` ： 包含项目的 Solidity 合约。你也可以在此使用 `.t.sol` 扩展名来包含 Solidity 测试文件。
+ `test` ：测试文件， 用于 TypeScript 集成测试。也可以在此处包含 Solidity 测试文件。
+ `ignition` ： 用于保存 Hardhat Ignition 部署模块，这些模块描述了合约的部署方式。
+ `scripts` : 用于存放自动执行部分工作流程的自定义脚本。脚本可完全访问 Hardhat 的运行时，并可使用插件、连接网络、部署合约等。


## 命令
```
pnpm hardhat node // 开启一个 Hardhat 节点

pnpm dlx hardhat --init //初始化项目
pnpm hardhat build //编译
pnpm hardhat test  //跑测试

pnpm hardhat run scripts/send-op-tx.ts //与链交互

pnpm hardhat ignition deploy --network ganache ignition/modules/Counter.ts //部署合约到本地网络
pnpm hardhat ignition visualize ignition/modules/Counter.ts
pnpm hardhat ignition wipe <deploymentId> <futureId> ---> pnpm hardhat ignition wipe chain-1337 CounterModule#Counter

pnpm hardhat console --network localhost //进入了一个 Node.js 环境 + Hardhat 的 ethers.js，可以直接调用合约或查询账户
```

## ganache