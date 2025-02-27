# taic复盘笔记

## 拉取代码

```bash
git clone https://github.com/taic-repo/taic.git
git submodule update --init --recursive 
```

## 指定测试

将`taic/Makefile`中的A改为enq_deq_test`

```
TEST_DIR = taic-test
A ?= apps/enq_deq_test
ARCH ?= riscv64
```

## 构建运行

### 构建测试镜像

在`taic/taic-test`目录下

```bash
make disk_img
```

### 编译riscv64-linux-musl-gcc

taic需要musl的gcc编译器，所以需要重新编译一个交叉编译的工具链

#### 1.安装工具链

```bash
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip python3-tomli libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev
```

#### 2.获取源码

```bash
git clone https://github.com/riscv/riscv-gnu-toolchain
```

#### 3. 编译

`--prefix` 为编译后的目标路径，`--enable-multilib` 选项用于启用多库支持

make linux选择交叉编译

```bash
./configure --prefix=/opt/riscv --enable-multilib
make musl -j4
```

#### 4.重命名

这样编译出的编译器采用GNU 工具链的传统命名规范—— `riscv64-unknown-linux-musl-gcc`

使用简化命名`riscv64-linux-musl-gcc`

使用如下脚本统一重命名

```bash
#!/bin/bash

# 定义工具链路径
TOOLCHAIN_PATH="/path/to/your/toolchain/bin"  # 替换为你的工具链路径

# 进入工具链目录
cd "$TOOLCHAIN_PATH" || { echo "路径不存在"; exit 1; }

# 遍历所有工具链文件
for file in *-unknown-*; do
    if [[ -f "$file" ]]; then
        # 替换 "unknown" 为更简洁的描述
        new_name=$(echo "$file" | sed 's/-unknown-/-/')
        
        # 重命名文件
        mv "$file" "$new_name"
        echo "已重命名: $file -> $new_name"
    fi
done

echo "重命名完成！"
```

### 运行

在`taic`路径下

````bash
make upload_payload
make run_on_qemu
````

