---
layout: post
title: "Compile EOS.IO on clean install CentOS 7"
date: 2018-01-30 11:40:01 +0800
tags: ["EOS","Linux"]
---

This article describe how to complie EOS node on clean installed CentOS 7.

There is a lot of posts about how to compile EOS on MacOS and Ubuntu, and the ***[official site](https://github.com/EOSIO/eos/wiki/Local-Environment#22-manual-build-script)*** has given a complete compile procedure guidence & automatic scripts to help the developers.
However the choice of operting system is subjected to the actual cluster we use in production environment, and the official did not give a clue how to compile on different systems. According to my own compile practice, there are a lot of bugs and traps that need to be avoid when you compile the EOS. To help the developers who are anxious to experience the EOS on their prefered operating system, I write the following article to demonstrate how I compile on CentOS 7 and how to avoid the traps that hidden under the ***[offical guide](https://github.com/EOSIO/eos/wiki/Local-Environment#2-building-eos)*** .


# 1. Hardware and System requirements
Due to the requirements of building LLVM WebAssembly，the host must have 10GB free space and the RAM need to be greater than 16GB, multiple cpu cores is suggested.

The operating system we use is CentOS 7.4 , you may choose other Redhat releases.

# 2. Basic compile environment install

According to the description on EOS.IO github official site,the EOS project is composed on C++14 stander, the default compile come with CentOS 7 is GNU 4.8.5, thus we need to manual choose different repos to install higher verison of GNU compile toolchain. Besides, the CMake version in CentOS 7 official site is a bit old, we need to install the newest Cmake manually.

- Cmake 3.10.2
- GNU C/C++ compiler 6 serialies

```bash
export BUILD_TEMP=${HOME}/tmp
mkdir ${BUILD_TEMP}

#GNU C++ 6.3.1
sudo yum install -y centos-release-scl
sudo yum install -y devtoolset-6
scl enable devtoolset-6 bash

#Ninja build tools
sudo yum install -y epel-release && \
sudo yum install -y ninja-build

#CMake 3.10.2
export BUILD_TEMP=${HOME}/tmp
mkdir ${BUILD_TEMP}

cd  ${BUILD_TEMP}
mkdir ${HOME}/opt
curl -L https://cmake.org/files/v3.10/cmake-3.10.2-Linux-x86_64.tar.gz -o cmake-3.10.2-Linux-x86_64.tar.gz
tar -zxf cmake-3.10.2-Linux-x86_64.tar.gz -C ${HOME}/opt

export PATH=$PATH:${HOME}/opt/cmake-3.10.2-Linux-x86_64/bin/

#Other compile tool chain
sudo yum install -y git autoconf automake libtool doxygen ocaml  gmp-devel python-devel bzip2-devel openssl-devel libicu-devel bzip2

```

# 3. Install EOS.IO dependencies
- Boost 1.64
- secp256k1-zkp
- Binaryen
- LLVM 4.0 with WebAssembly (On actual compile environment, you also have to compile X86 target for a clean CentOS installation)

## 3.1 Compile Boost
```bash
export BOOST_ROOT=${HOME}/opt/boost_1_64_0
cd  ${BUILD_TEMP}
curl -L https://sourceforge.net/projects/boost/files/boost/1.64.0/boost_1_64_0.tar.bz2 > boost_1.64.0.tar.bz2 && \
tar xvf boost_1.64.0.tar.bz2 && \
cd boost_1_64_0/ && \
./bootstrap.sh "--prefix=$BOOST_ROOT" && \
./b2 install
```

## 3.2 Compile secp256k1-zkp
```bash
cd  ${BUILD_TEMP}
git clone https://github.com/cryptonomex/secp256k1-zkp.git && \
cd secp256k1-zkp && \
./autogen.sh && \
./configure && \
make
sudo make install
```

## 3.3 Compile Binaryen
```bash
cd  ${BUILD_TEMP}
git clone https://github.com/WebAssembly/binaryen && \
cd binaryen && \
git checkout tags/1.37.14 && \
cmake . && \
make -j 4
sudo make install
```

## 3.4 Compile MangoDB driver
```bash
cd  ${BUILD_TEMP}

wget https://github.com/mongodb/libbson/releases/download/1.9.2/libbson-1.9.2.tar.gz && \
tar -xzf libbson-1.9.2.tar.gz && \
cd libbson-1.9.2/ && \
./configure && \
make -j4
sudo make install

curl -L https://github.com/mongodb/mongo-c-driver/releases/download/1.8.0/mongo-c-driver-1.8.0.tar.gz -o mongo-c-driver-1.8.0.tar.gz && \
tar -xzf mongo-c-driver-1.8.0.tar.gz && \
cd mongo-c-driver-1.8.0 && \
./configure --disable-automatic-init-and-cleanup && \
make -j4
sudo make install

export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig

git clone --depth 1 -b releases/stable git://github.com/mongodb/mongo-cxx-driver &&\
cd mongo-cxx-driver &&\
cmake -H. -Bbuild -G Ninja -DCMAKE_BUILD_TYPE=Release  -DCMAKE_INSTALL_PREFIX=/usr/local && \
sudo cmake --build build --target install

sudo vi /etc/ld.so.conf
#Add following to files
#/usr/local/lib
sudo ldconfig
```

## 3.5 Compile LLVM WebAssembly+ Host Target
- TRAP 1
Due the EOS need LLVM to compile C++ to WebAssembly, the frontend of Clang and WebAssembly are both required, the official compile guidance only compile the WebAssembly target (I assume that is based on the clang is already installed on the ubuntu system,so the auther omitted it when compile LLVM for WebAssembly.), when we compile the LLVM wtih GNU compiler we need to both compile the WebAssembly and X86 target (which is achived by adding parameter `DLLVM_TARGETS_TO_BUILD='host'` to the cmake configuration), otherwise we would encounter `missing LLVMX86****.so` error when you compile EOS.
- TRAP 2
At current, the EOS only support LLVM of version 4.0,which is achived by clone branch 40 from [llvm mirror site](https://github.com/llvm-mirror/llvm.git), **DO NOT** pull the latest source from github directlly without branch selection.
- TRAP 3
You have to enable RTTI when you compile, otherwise you may encounter error like ***typeof ** undefine*** when you compile EOS. You can avoid this error by add parameter DLLVM_ENABLE_RTTI=ON to the cmake configuration.

```bash

mkdir ${BUILD_TEMP}/wasm-compiler/build -p
cd ${BUILD_TEMP}/wasm-compiler && \
git clone --depth 1 --single-branch --branch release_40 https://github.com/llvm-mirror/llvm.git && \
cd llvm/tools && \
git clone --depth 1 --single-branch --branch release_40 https://github.com/llvm-mirror/clang.git && \
cd ${BUILD_TEMP}/wasm-compiler/build && \
cmake -G "Unix Makefiles" -DLLVM_ENABLE_RTTI=ON \
-DLLVM_TARGETS_TO_BUILD='host' \
-DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly \
-DCMAKE_BUILD_TYPE=Release \
../llvm

sudo make -j4 install

# export WASM_LLVM_CONFIG=/usr/local/bin/llvm-config
```

## Compile EOS.IO

```bash
cd
git clone https://github.com/eosio/eos --recursive
cd eos

#If you need dawn 2.x version,uncomment following
#git checkout dawn-2.x

cmake -H. -B"build" -GNinja -DCMAKE_BUILD_TYPE=Release \
-DWASM_LLVM_CONFIG=/usr/local/bin/llvm-config \
-DCMAKE_INSTALL_PREFIX=/opt/eos \
-DBOOST_INCLUDEDIR=${BOOST_ROOT}/include/boost \
-DBUILD_MONGO_DB_PLUGIN=ON \
-DBOOST_LIBRARYDIR=${BOOST_ROOT}/lib
sudo cmake --build build --target install
```
Now, the binaries are installed in build/install/bin directory
```bash
ll build/install/bin/
# total 141276
# drwxrwxr-x. 2 highland highland       24 Feb 11 23:19 data-dir
# -rwxr-xr-x. 1 highland highland  3056136 Feb 11 23:07 embed_genesis
# -rwxr-xr-x. 1 highland highland 55330872 Feb 11 23:09 eosio-abigen
# -rwxr-xr-x. 1 highland highland  4309440 Feb 11 23:08 eosioc
# -rwxr-xr-x. 1 highland highland 35388792 Feb 11 23:08 eosio-codegen
# -r-xr-xr-x. 1 highland highland     4195 Feb 11 22:57 eosiocpp
# -rwxr-xr-x. 1 highland highland 39722544 Feb 11 23:08 eosiod
# -rwxr-xr-x. 1 highland highland  2293128 Feb 11 23:08 eosio-launcher
# -rwxr-xr-x. 1 highland highland  4545360 Feb 11 23:08 eosiowd
```


## Start your first node
Please refer to [official documents](https://github.com/EOSIO/eos/wiki/Local-Environment#4-creating-and-launching-a-single-node-testnet).
