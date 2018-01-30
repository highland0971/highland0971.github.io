---
layout: post
title: "Compile EOS.IO on clean install CentOS 7"
date: 2018-01-30 11:40:01 +0800
tags: ["EOS","Linux"]
---

This article describe how to complie EOS node on clean installed CentOS 7.

On the [official site](https://github.com/EOSIO/eos/wiki/Local-Environment#22-manual-build-script) only described how compile on Ubuntu or MacOS,missing other linux system. This article will give a clue on how to achieve successfull compile.

## Basic compile environment install

Because EOS.IO use C++14, the CentOS 7 default GNU compile toolchain will not be compatible with the requirements, newer C complier installation is required.

- Cmake 3.10.2
- GNU C/C++ compiler 6 serialies

```bash
sudo yum install -y centos-release-scl
sudo yum install -y devtoolset-6
sudo yum install -y git gmp-devel python-devel bzip2-devel openssl-devel libicu-devel autoconf automake libtool doxygen ocaml bzip2 wget

cd
curl -L https://cmake.org/files/v3.10/cmake-3.10.2-Linux-x86_64.tar.gz -o cmake-3.10.2-Linux-x86_64.tar.gz
tar -zxf cmake-3.10.2-Linux-x86_64.tar.gz -C ${HOME}/opt

export PATH=$PATH:/opt/rh/devtoolset-6/root/bin/:${HOME}/opt/cmake-3.10.2-Linux-x86_64/bin/
```

## Install EOS.IO dependencies
- Boost 1.64
- secp256k1-zkp
- Binaryen
- LLVM 4.0 with WebAssembly enabled

```bash
export BOOST_ROOT=${HOME}/opt/boost_1_64_0
curl -L https://sourceforge.net/projects/boost/files/boost/1.64.0/boost_1_64_0.tar.bz2 > boost_1.64.0.tar.bz2
tar xvf boost_1.64.0.tar.bz2
cd boost_1_64_0/
./bootstrap.sh "--prefix=$BOOST_ROOT"
./b2 install

cd
git clone https://github.com/cryptonomex/secp256k1-zkp.git
cd secp256k1-zkp
./autogen.sh
./configure
make
sudo make install

cd
git clone https://github.com/WebAssembly/binaryen
cd binaryen
git checkout tags/1.37.14
cmake . && make -j 4
sudo make install

cd
mkdir ${HOME}/wasm-compiler/build -p
cd ${HOME}/wasm-compiler
git clone --depth 1 --single-branch --branch release_40 https://github.com/llvm-mirror/llvm.git
cd llvm/tools
git clone --depth 1 --single-branch --branch release_40 https://github.com/llvm-mirror/clang.git
cd ${HOME}/wasm-compiler/build
cmake -G "Unix Makefiles" -DLLVM_ENABLE_RTTI=ON -DLLVM_TARGETS_TO_BUILD= -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly -DCMAKE_BUILD_TYPE=Release ../llvm
sudo make -j4 install
export WASM_LLVM_CONFIG=/usr/local/bin/llvm-config
```

## Compile EOS.IO

```bash
cd
git clone https://github.com/eosio/eos --recursive
mkdir -p ${HOME}/eos/build && cd ${HOME}/eos/build

cmake -DBOOST_INCLUDEDIR=${BOOST_ROOT}/include/boost \
-DBOOST_LIBRARYDIR=${BOOST_ROOT}/lib  ..
make -j4
```
