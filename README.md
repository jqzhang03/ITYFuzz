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

