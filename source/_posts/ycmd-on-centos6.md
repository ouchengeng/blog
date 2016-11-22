title: 在CentOS 6下构建ycmd环境
date: 2016-11-14
tags: 
- ycmd
- centos6
- spacemacs
thumbnail: /images/ycmd.png
categories: infra
---

本文旨在合理安装ycmd于CentOS 6，非修改libc。

<!-- more -->

首先做用户隔离，将`/home/amos`替换为你的用户根。

```
export MANPATH=/home/amos/share/man:$MANPATH
export INFOPATH=/home/amos/share/info:$INFOPATH
export PATH=/home/amos/bin:$PATH
export LIBRARY_PATH=/home/amos/lib:$LIBRARY_PATH
export LD_LIBRARY_PATH=/home/amos/lib:$LD_LIBRARY_PATH
```

下载python2.7.12，编译安装。
```
./configure --prefix=/home/amos --enable-shared
make -j`nproc` && make install
```

下载cmake3.7.0，编译安装。
```
./configure --prefix=/home/amos
make -j`nproc` && make install
```

下载llvm3.9.0以及cfe-3.9.0，编译安装。
```
tar -xf llvm-3.9.0.src.tar.xz
cd llvm-3.9.0.src
tar -xf ../cfe-3.9.0.src.tar.xz -C tools
mv tools/cfe-3.9.0.src tools/clang
mkdir -p build
cd build

yum install ncurses-devel
cmake -DCMAKE_INSTALL_PREFIX=/home/amos           \
      -DCMAKE_BUILD_TYPE=Release            \
      -DLLVM_BUILD_LLVM_DYLIB=ON            \
      -DLLVM_TARGETS_TO_BUILD="host;AMDGPU" \
      -Wno-dev ..
make -j `nproc` && make install
```

安装devtoolset-4，这个比较大。
```
yum install centos-release-scl-rh
sudo yum install devtoolset-4
```

安装ycmd。
```
scl enable devtoolset-4 bash
git clone https://github.com/Valloric/ycmd
cd ycmd
git submodule update --init --recursive
./build.py --clang-completer --system-libclang
```

**确保将emacs daemon启动在devtoolset-4的shell里。**

注：
spacemacs ycmd 的 global_conf.py 自动抽取`compile_commands.json`，
cmake工程使用`-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`生成`compile_commands.json`，
make工程使用`https://github.com/rizsotto/Bear`。
