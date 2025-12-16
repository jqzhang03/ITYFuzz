# ITYFuzz

## 环境配置

||操作系统|Rust|Solidity|Node|ITYFuzz|
|-|-|-|-|-|-|
|版本|Ubuntu22.04|1.92.0|0.8.20|22.21.1|fb4e87a36ae3b4596fe74178004d1b51e4cfcad5|

## 安装步骤

针对Ubuntu裸机，首先配置需要使用的Ubuntu系统命令
```bash
sudo apt upgrade

sudo apt install -y build-essential libssl-dev libz3-dev pkg-config cmake clang git curl
```

配置Rust环境
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 完成后另开一个终端激活环境变量
source "$HOME/.cargo/env"

# 利用以下命令检查Rust是否配置成功
rustc --version
```

配置Solidity环境
```bash
sudo apt install npm

sudo apt install nodejs

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

source ~/.bashrc

nvm install 22

sudo apt install -y python3-pip

python3 -m pip install solc-select

export PATH=$PATH:~/.local/bin

solc-select install 0.8.20

solc-select use 0.8.20
```

配置[ITYFuzz](https://github.com/fuzzland/ityfuzz)环境
```bash
curl -L https://ity.fuzz.land/ | bash

# 另起一个终端
ityfuzzup
```
![1](./public/1.png)
![2](./public/2.png)


#### 测试环境
```bash
mkdir repo

cd repo

touch Bug.sol
```
修改`Bug.sol`
```solidity
pragma solidity ^0.8.0;

contract Bug {
	function check(int a) public pure {
	    if (a == 1337) {
		    assert(false);
	    }
	}
}
```
测试ITYFuzz
```bash
# 会在目录下生成Bug.bin和Bug.abi文件
solc Bug.sol --bin --abi --optimize -o . --overwrite

ityfuzz evm -t ./Bug.bin
```

如果测试结果如下图所示，则代表ITYFuzz安装成功。在下图中ITYFuzz找到了一个BUG，并且覆盖率达到了92.11%。
+ `0x4e487b71`:指的是Solidity内置的`Panic(uint256)`错误的函数选择器。
+ `...0001`:Panic的错误代码，`0x01`对应的是`assert`失败。

![3](./public/3.png)

## 运行指南
ItyFuzz 支持 EVM 和 Move 智能合约的链下（本地）模糊和链上（叉）模糊。

### Onchain Fuzzing (EVM)
要运行链上模糊测试活动，指定目标合约和要分叉的链。
```bash
# -t [TARGET_ADDR]: specify the target contract
# --onchain-block-number [BLOCK]: fork the chain at block number [BLOCK]
# -c [CHAIN_TYPE]: specify the chain

ityfuzz evm\
    -t [TARGET_ADDR]\
    --onchain-block-number [BLOCK]\
    -c [CHAIN_TYPE]\
    --onchain-etherscan-api-key [Etherscan API Key] # (Optional) specify your etherscan api key
```
例如，要在以太坊上针对 WETH 运行链上模糊搜索活动，运行：
```bash
# -t [TARGET_ADDR]: specify the target contract
# --onchain-block-number [BLOCK]: fork the chain at block number [BLOCK]
# -c [CHAIN_TYPE]: specify the chain
# -f: (Optional) allow attack to get flashloan

ityfuzz evm\
    -t 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2\
    --onchain-block-number 0\
    -c ETH\
    --onchain-etherscan-api-key [Etherscan API Key]\
    -f
```
ItyFuzz 会从 Etherscan 拉取合同的 ABI 并进行模糊处理。如果 ItyFuzz 在内存中遇到未知槽，它会从链 RPC 中拉取该槽。如果 ItyFuzz 遇到调用外部未知合同，它会拉取该合同的字节码和 ABI。如果其 ABI 不可用，ItyFuzz 会反编译并获取 ABI。

### Offchain Fuzzing (EVM)
要运行本地模糊测试活动，请指定目标合约（只需字节码和 ABI）。
```bash
# -t [BUILD DIRECTORY GLOB]: specify the targets directory
# -f: (Optional) allow attack to get flashloan
# --concolic: (Optional) enable concolic execution
# --concolic-caller: (Optional) enable concolic execution to change caller to anyone

ityfuzz evm\
    -t "[BUILD DIRECTORY GLOB]"\
    -f\
    --concolic --concolic-caller
```
例如，在一个汇编的单一合同上运行一个简单的模糊测试活动：
```bash
ityfuzz evm -t './build/*'
```
ItyFuzz 会尝试将目录中的所有工件部署到没有其他智能合约的区块链上。
具体来说，项目目录应包含几个 [X].abi 和 [X].bin 文件。例如，要模糊一个名为 main.sol 的合同，你应该确保 main.abi 和 main.bin 存在于项目目录中。fuzzer 会自动检测目录中的合约及其之间的关联（参见 tests/evm/multi-contract），并对它们进行模糊处理。  
如果 ItyFuzz 未能推断合同间的相关性，可以添加 [X].address，其中 [X] 是合同名称，用于指定合同地址。   
要定义自定义不变量，可以查看 Custom Invariant 或 Echidna / Scribble Support。

注意事项：
请记住，ItyFuzz 是在干净的区块链上进行模糊检测，因此你应确保所有相关合约（例如 ERC20 代币、Uniswap 等）都已部署到区块链上，再进行模糊搜索。

### Offchain Fuzzing (MoveVM)
使用 sui move build 编译合约并运行 ItyFuzz：
```bash
# build example contract that contains a bug
cd ./tests/move/share_object
sui move build

# get back to ItyFuzz and run fuzzing on the built contract
cd ../../../
ityfuzz move -t "./tests/move/share_object/build"
```

#### Defining Invariants
你可以在合同中发出 AAAA__fuzzland_move_bug 事件，在发现漏洞时报告状况。
```bash
// define the event struct
use sui::event;

struct AAAA__fuzzland_move_bug has drop, copy, store {
    info: u64
}

... 
    // inside function
    event::emit(AAAA__fuzzland_move_bug { info: 1 });
...
```