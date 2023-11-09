## 第1周 练习作业

### 源码下载
```
git clone https://github.com/Rust-for-Linux/linux -b rust-dev --depth 1
git clone https://github.com/fujita/linux.git -b rust-e1000 --depth 1
```

### 环境搭建
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

### 编译时遇到的问题 bindgen版本过高,修改了部分增对bindgen参数的Makefile配置
<img width="1633" alt="image" src="https://github.com/xxkeming/rust/assets/11630632/3a04cfab-e44a-4f5a-9198-f983c7ba8ea4">

### 编译流程,配置开启rust,并把练习2的代码及配置加入
```
make ARCH=arm64 LLVM=1 O=build defconfig

make ARCH=arm64 LLVM=1 O=build menuconfig

cd build
make ARCH=arm64 LLVM=1 -j4
```

### 启动方式
```
qemu-system-aarch64 -machine 'virt' -cpu 'cortex-a57' -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -kernel /home/parallels/linux/build/arch/arm64/boot/Image.gz -initrd initrd -nographic -append "root=LABEL=rootfs console=ttyAMA0"

```

### 运行结果
<img width="1197" alt="image" src="https://github.com/xxkeming/rust/assets/11630632/73cae450-bf36-4d90-acdb-a0b61fd636d6">
