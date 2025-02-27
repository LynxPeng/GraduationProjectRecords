# rel4test 复现笔记

## 部署环境

|   环境    | 版本                                                |
| :-------: | :-------------------------------------------------- |
|   系统    | vmware虚拟机	ubuntu 24.04.1                      |
|  python   | mini-conda Python 3.12.8                            |
|   qemu    | QEMU emulator version 6.2.94                        |
| riscv-gcc | riscv64-unknown-linux-gnu-gcc (g04696df0963) 14.2.0 |
|   make    | GNU Make 4.3                                        |
|   cmake   | cmake version 3.28.3                                |
|   ninja   | version 1.11.1                                      |
|   repo    | repo version v2.50.1                                |

## 环境准备

> [!note]
>
> **sel4test文档新网址**https://docs.sel4.systems/projects/buildsystem/host-dependencies.html

### repo

repo使用清华源repo

```bash
mkdir ~/bin  
cd  ~/bin
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod a+x ~/bin/repo
nano ~/.bashrc
# 添加环境变量
export PATH=~/bin:$PATH #添加变量
source ~/.bashrc    #更新环境变量
repo --version #查看版本
```

> [!WARNING]
>
> repo拉取代码报错 
>
> ​	fatal: Cannot get https://gerrit.googlesource.com/git-repo/clone.bundle
>
> 解决办法：每次执行repo init时，增加选项--repo-url=https://gerrit-googlesource.lug.ustc.edu.cn/git-repo

### 软件包

```bash
sudo apt-get update
sudo apt-get install build-essential
sudo apt-get install cmake ccache ninja-build cmake-curses-gui
sudo apt-get install libxml2-utils ncurses-dev
sudo apt-get install curl git doxygen device-tree-compiler
sudo apt-get install u-boot-tools
sudo apt-get install python3-dev python3-pip python-is-python3
sudo apt-get install protobuf-compiler python3-protobuf
```

> [!WARNING]
>
>**mlton-compiler** 在新版Ubuntu系统中找不到，目前无需此包也可跑通rel4

### risc-v 工具链

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
make linux -j4
```

### qemu

本地编译qemu

```bash
sudo apt-get install zlib1g-dev libpixman-1-dev libfdt-dev
git clone https://github.com/rel4team/qemu.git
cd qemu
./configure --target-list=riscv64-softmmu --disable-werror --prefix=/home/xxx/rel4_qemu/
make -j4
make install
```

>  [!WARNING]
>
>  悬空指针警告导致编译报错 cc1: all warnings being treated as errors
>
>  ![image-20250226083903612](D:/Library/picture/Typora/image-20250226083903612.png)
>
>  警告被当成错误处理，需要在./configure中添加参数--disable-werror

### miniconda

conda仅用于部署Python环境

安装时选择初始化选项选择是

````bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
````

### rust

#### 1.工具链

```bash
sudo apt install curl build-essential gcc make
```

#### 2.换源

~/.bashrc

```bash
# 中科大
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
# 字节
export RUSTUP_DIST_SERVER=https://rsproxy.cn
export RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup
```

#### 3.安装

```bash
curl https://sh.rustup.rs -sSf | sh
source $HOME/.cargo/env
```

#### 4.cargo 换源

```yaml
[source.crates-io]
replace-with = 'rsproxy'
 
# 清华大学 5mb
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
 
# 中国科学技术大学 2mb
[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"
# 上海交通大学 2mb
[source.sjtu]
registry = "https://mirrors.sjtug.sjtu.edu.cn/git/crates.io-index"
 
# rustcc社区 2mb
[source.rustcc]
registry = "https://crates.rustcc.cn/crates.io-index"
# 字节跳动 10mb
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"

```

测试

```bash
cargo check
```

### Torjan代理 & proxychains配置

#### 1. 安装Trojan

下载Trojan客户端

```bash
sudo apt install wget -y
wget https://github.com/trojan-gfw/trojan/releases/latest/download/trojan-linux-amd64.tar.xz
tar -xvf trojan-linux-amd64.tar.xz
sudo mv trojan /usr/local/bin/
trojan --version
```

#### 2.配置Trojan

```bash
sudo mkdir -p /etc/trojan/
```

新建 `/etc/trojan/config.json`

```bash
{
    "run_type": "client",
    "local_addr": "127.0.0.1",
    "local_port": 1080,
    "remote_addr": "your_server_domain_or_ip",
    "remote_port": 443,	
    "password": [
        "your_password"
    ],
    "ssl": {
        "verify": false,
        "verify_hostname": false,
        "cert": "",
        "sni": "your_server_domain"
    }
}
```

#### 3.配置为系统服务

创建文件`/etc/systemd/system/trojan.service`

```bash
[Unit]
Description=Trojan Client
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/trojan -c /etc/trojan/config.json
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

启动服务并设置为开机自启

```bash
sudo systemctl daemon-reload
sudo systemctl enable trojan
sudo systemctl start trojan
sudo systemctl status trojan
```

#### 4.配置proxychains

安装

```bash
sudo apt-get install proxychains
```

编译配置文件`/etc/proxychains.conf`

```bash
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
# socks4        127.0.0.1 9050
socks5 127.0.0.1 1080
```

#### 5.使用

在需要代理的命令前加入proxychains即可使用代理

```bash
proxychains git clone https://github.com/rel4team/rust-root-task-demo.git
```

## 运行环境配置

#### 克隆仓库

```bash
mkdir rel4test && cd rel4test
repo init -u https://github.com/rel4team/sel4test-manifest.git
repo sync
```

#### 配置环境

```bash
cd ./rel4_kernel
make env
```

>[!WARNING]
>
>make env报错 error: rustup is not installed at '/root/.cargo'
>
>解决办法：卸载ubuntu原本rustup，重新安装rustup

>[!NOTE]
>
>目前可用代码仓库版本
>
>rel4 kernel branch: fpga test
>seL4test: main
>tools/opensbi branch: v1.3

## C侧代码

#### **构建编译：**

rel4test/rel4_kernel

```
./build.py -c 4 -u
```

#### 仿真运行：

rel4test/rel4_kernel/build

```
./simulate -b qemu-build路径/qemu-system-riscv64 -M virt --cpu-num 4
```

其中：

- -b：指定qemu路径，后续qemu路径需根据以往的qemu配置更改
- -M：指定虚拟机器类型
- --cpu-num：指定cpu核心数量

### 遇到的问题

> [!WARNING]
>
> rust版本问题报错
>
> Caused by:61.03package `home va.5.11` cannot be built because it requires rustc 1.81 or newer, while the currently active rustc version is 1.77.0-nightly61.03Either upgrade to rustc 1.81 or newer, or use61.03cargo update home@0.5.11 --precise ver61.03where`ver`is the latest version of `home`supporting rustc 1.77.0-nightly
>
> 1.  更新rust版本
> 2. rust-toolchain.toml会锁定当前rust版本
> 3. 重新make env

> [!WARNING]
>
>找不到指令
>
>![image-20250226094810089](D:/Library/picture/Typora/image-20250226094810089.png)
>
>**解决办法：**rust版本不能低于rustc 1.71.0-nightly(9ecda8de8 2023-04-30)
>
>后续版本移除了这些指令

>  [!WARNING]
>
>![image-20250226094938999](D:/Library/picture/Typora/image-20250226094938999.png)
>
>**解决办法：**sel4test checkout到2b63c9183a7aae707004afdbd3157b41aeb3ae7e

>   [!WARNING]
>
>riscv gcc找不到模块
>
>![image-20250226095149977](D:/Library/picture/Typora/image-20250226095149977.png)
>
>**解决办法：**卸载apt安装的/直接下载的编译器，riscv64-unknown-linux-gnu换成本地编译的。

>​    [!WARNING]
>
>运行测试卡在Initializing PLIC...
>
>![image-20250226095416948](D:/Library/picture/Typora/image-20250226095416948.png)
>
>**解决办法：**卸载ubuntu自带的qemu，改为本地编译的qemu

## R侧代码

```bash
# In rel4_kernel dirctory
$ cd rel4_kernel 
$ make env
$ ./build.py
# build smp version
$ ./build.py -c 4

# run on qemu
$ cd build
$ ./simulate -b <your qemu path> -M virt --cpu-num <cpu-num> #(1 or 4)
```

在rel4/project目录下克隆仓库

```bash
git clone --recursive https://github.com/rel4team/rust-root-task-demo.git projects/rust-root-task-demo
```

在rel4/rel4_kernel中安装头文件

```bash
$ make env
$ ./build.py -c 4 -u -i
```

添加环境变量 (~/.bashrc)

```shell
export SEL4_INSTALL_DIR=/root/rel4/rel4test/kernel/install
export SEL4_PREFIX=/root/rel4/rel4test/kernel/install
```

编译 (rel4/rel4_kernel)

```
./build.py -c 4 -u -r
```

运行

```
./simulate -b <your qemu path> -M virt --cpu-num 4
```

>  [!WARNING]
>
>build rust-root-task-demo失败
>
>![image-20250226095756372](D:/Library/picture/Typora/image-20250226095756372.png)
>
>**解决办法：**
>
>​	1. 在rust-root-task-demo中build，检查rust-root-task-demo代码是否有报错
>
>2. 检查环境变量和头文件安装



/root/taic/taic-qemu/build/qemu-system-riscv64  -machine virt -cpu rv64 -nographic -serial mon:stdio -m size=4095M -bios default -kernel images/example-image-riscv-spike  -smp 4

../taic-qemu/build/qemu-system-riscv64 -m 2G -smp 1 -machine virt -bios default -kernel apps/enq_deq_test/enq_deq_test_riscv64-qemu-virt.bin -nographic
