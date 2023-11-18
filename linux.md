
## 源码下载
```
git clone https://github.com/Rust-for-Linux/linux -b rust-dev --depth 1
git clone https://github.com/fujita/linux.git -b rust-e1000 --depth 1
```

## 环境搭建
```
## Install dependency packages
sudo apt-get -y install \
  binutils build-essential libtool texinfo \
  gzip zip unzip patchutils curl git \
  make cmake ninja-build automake bison flex gperf \
  grep sed gawk bc \
  zlib1g-dev libexpat1-dev libmpc-dev \
  libglib2.0-dev libfdt-dev libpixman-1-dev libelf-dev libssl-dev

## Install the LLVM
sudo apt-get install clang-format clang-tidy clang-tools clang clangd libc++-dev libc++1 libc++abi-dev libc++abi1 libclang-dev libclang1 liblldb-dev libllvm-ocaml-dev libomp-dev libomp5 lld lldb llvm-dev llvm-runtime llvm python3-clang

## Add Rust environment
cd linux
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
make LLVM=1 rustavailable
```

### bindgen 版本问题需修改rust/Makefile
```
--size_t-is-usize -> del
--blacklist-type -> --blocklist-type
--whitelist-var -> --allowlist-var
--whitelist-function -> --allowlist-function
```

### 编译,配置开启rust
```
make ARCH=arm64 LLVM=1 O=build defconfig
make ARCH=arm64 LLVM=1 O=build menuconfig
cd build && time make ARCH=arm64 LLVM=1 -j4
```

### 生成 rust-analyzer, rust-project.json
```
linux/build目录下执行
make ARCH=arm64 LLVM=1 rust-analyzer

外部项目执行,会添加当前目录所以*.rs
make ARCH=arm64 -C ~/linux/build/ M=$PWD rust-analyzer
```

### 配置vocode
```
ctrl + shift + p 选择 open workspace settings json
settings.json
{
    "rust-analyzer.linkedProjects": ["rust-project.json"]
}
```

## 自定义基本模块, 通过insmod test.ko 加载
### test.rs
```
use kernel::prelude::*;
module! {
  type: RustTest,
  name: "rust_test",
  author: "keming",
  description: "test module in rust",
  license: "GPL",
}
struct RustTest {}
impl kernel::Module for RustTest {
  fn init(_name: &'static CStr, _module: &'static ThisModule) -> Result<Self> {
      pr_info!("test from Rust module");
      Ok(RustTest {})
  }
}
```
### Makefile
```
obj-m := test.o
PWD := $(shell pwd)
ARCH ?= arm64
KDIR ?= /lib/modules/$(shell uname -r)/build
default:
	$(MAKE) ARCH=$(ARCH) LLVM=1 -C $(KDIR) M=$(PWD)/ modules
clean:
	$(MAKE) ARCH=$(ARCH) LLVM=1 -C $(KDIR) M=$(PWD)/ clean
```

## qemu相关配置
### 主机网络配置
```
sudo ip link add br0 type bridge
sudo ip addr add 192.168.100.50/24 brd 192.168.100.255 dev br0
sudo ip tuntap add mode tap user $(whoami)
ip tuntap show
sudo ip link set tap0 master br0
sudo ip link set dev br0 up
sudo ip link set dev tap0 up
```

### 虚拟机网络配置
```
ifconfig lo 127.0.0.1 netmask 255.0.0.0 up
ifconfig eth0 192.168.100.224 netmask 255.255.255.0 broadcast 192.168.100.255 up
```

### 启动方式参考,及busybox生成
```
qemu-system-aarch64 -machine virt -cpu cortex-a53 -nographic -smp 1 -m 2048 -kernel build/arch/arm64/boot/Image.gz -initrd qemu-initramfs.img

qemu-system-aarch64 \
  -kernel build/arch/arm64/boot/Image.gz \
  -initrd qemu-initramfs.img \
  -M virt \
  -cpu cortex-a72 \
  -smp 2 \
  -nographic \
  -vga none \
  -no-reboot \

qemu-system-aarch64 -M virt -cpu cortex-a57 -smp 1 -m 256M -kernel build/arch/arm64/boot/Image.gz -initrd ../busybox-1.36.1/initramfs.cpio.gz -nographic -vga none -no-reboot -append "init=/init console=ttyAMA0" -netdev tap,ifname=tap0,id=tap0,script=no,downscript=no -device e1000,netdev=tap0
```
