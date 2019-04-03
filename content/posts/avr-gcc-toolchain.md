---
title: "Avr Gcc Toolchain"
date: 2019-04-03T20:42:49+02:00
draft: true
---

## Intro

I have currently bought me my first Arduino UNO.. At least it says it's an UNO.. To my surprise all the examples uses C++ as the lingua franca of examples. This was disappointing as I wanted something a little more C-related. At the time, I thought getting the latest GCC for avr working I would have to compile it myself. 

### TL;DR

Here's the script I ended up with to build AVR GCC and friends. If you want to build for a 64 bit machine just leave out all the `-m32` lines in the script. (I have a busted old laptop I want to do this stuff on)

```bash
#!/usr/bin/env bash
# Instructions from https://www.nongnu.org/avr-libc/user-manual/install_tools.html
# From ftp://ftp.lip6.fr/pub/gcc/releases/gcc-8.3.0/gcc-8.3.0.tar.xz
PREFIX="${HOME}"/opt/cross/avr-32/
TARGET=avr
export PATH="${PREFIX}/bin:${PATH}"
export CFLAGS=-m32
export LDFLAGS=-m32
export CXXFLAGS=-m32

BINUTILS=binutils-2.32
BINUTILS_ARCHIVE="${BINUTILS}.tar.xz"
if [[ ! -f "${BINUTILS_ARCHIVE}" ]]
then
    wget https://ftp.gnu.org/gnu/binutils/${BINUTILS_ARCHIVE}
fi
if [[ ! -d  "${BINUTILS}" ]]
then
    tar -xvf "${BINUTILS_ARCHIVE}"
    cd "${BINUTILS}"
    mkdir build-dir && cd build-dir/
    ../configure --prefix="${PREFIX}"  --target="${TARGET}" --disable-nls
    make && make install
    if [[ $? -ne 0 ]]
    then
        echo "failed to build "${BINUTILS}""
        exit 1
    fi
    cd ../../
fi


GCC_VERSION=gcc-8.3.0
GCC_ARCHIVE="${GCC_VERSION}.tar.xz"
if [[ ! -f "${GCC_ARCHIVE}" ]]
then
    wget ftp://ftp.lip6.fr/pub/gcc/releases/${GCC_VERSION}/${GCC_ARCHIVE}
fi
if [[ !  -d  "${GCC_VERSION}" ]]
then
    tar -xvf  "${GCC_ARCHIVE}" 
    cd "${GCC_VERSION}"
    cp ../deps/* .
    ./contrib/download_prerequisites
    mkdir build-dir && cd build-dir
    ../configure  --prefix="${PREFIX}"  --target="${TARGET}" --enable-languages=c --disable-nls --disable-libssp --with-dwarf2
    make && make install
    if [[ $? -ne 0 ]]
    then
        echo "failed to build "${GCC_VERSION}""
        exit 1
    fi
    cd ../../
fi


AVR_LIB=avr-libc-2.0.0
AVR_LIB_ARCHIVE="${AVR_LIB}.tar.bz2"
if [[ ! -f "${AVR_LIB_ARCHIVE}" ]]
then
    wget http://download.savannah.gnu.org/releases/avr-libc/"${AVR_LIB_ARCHIVE}"
fi
if [[ ! -d "${AVR_LIB}" ]]
then
    tar -xvf "${AVR_LIB_ARCHIVE}"
    cd "${AVR_LIB}"
    mkdir build-dir && cd build-dir/
    ../configure --prefix="${PREFIX}"  --target="${TARGET}" --build=$(./config.guess) --host=avr
    make && make install
    if [[ $? -ne 0 ]]
    then
        echo "failed to build "${AVR_LIB}""
        exit 1
    fi
    cd ../../
fi

AVRDUDE=avrdude-6.3
AVRDUDE_ARCHIVE="${AVRDUDE}.tar.gz"
if [[ ! -f "${AVRDUDE_ARCHIVE}" ]] 
then
    wget http://download.savannah.gnu.org/releases/avrdude/"${AVRDUDE_ARCHIVE}"
fi
if [[ ! -d "${AVRDUDE}" ]]
then
    tar -xvf "${AVRDUDE}".tar.gz
    cd "${AVRDUDE}"
    mkdir build-dir && cd build-dir
    ../configure --prefix="${PREFIX}" 
    make && make install
    if [[ $? -ne 0 ]]
    then
        echo "failed to build "${AVRDUDE}""
        exit 1
    fi
    cd ../../
fi


GDB=gdb-8.2
GDB_ARCHIVE="${GDB}.tar.xz"
if [[ ! -f "${GDB_ARCHIVE}" ]]
then
    wget https://ftp.gnu.org/gnu/gdb/"${GDB_ARCHIVE}"
fi
if [[ ! -d "${GDB}" ]]
then
    tar -xvf "${GDB}".tar.xz
    cd "${GDB}"
    mkdir build-dir && cd build-dir
    ../configure --prefix="${PREFIX}"  --target="${TARGET}"
    make && make install
    if [[ $? -ne 0 ]]
    then
        echo "failed to build "${GDB}""
    fi
    cd ../../
fi


```

This builds everything from `avr-libc`, `gcc`, `gdb` and `avrdude` for a 32bit pc. It also follows most of the [avr-libc](https://www.nongnu.org/avr-libc/user-manual/install_tools.html) instructions. 

*Fun Fact* I built this on a 64bit machine, to run on a 32bit machine, to compile for an 8bit microcontroller. To use the nomenclature:
- The 64bit machine I built this one is called the "build machine"
- The 32bit machine I want to run this gcc on is called the "host machine"
- The 8bit micro I want to run the output of the gcc is called the "target machine"
If my understanding of [crosstool-ng](https://crosstool-ng.github.io/docs/toolchain-types/) and [wikipedia](https://en.wikipedia.org/wiki/Cross_compiler#Canadian_Cross) is correct, this counts as an **Canadian Cross Toolchain*. 

So now no more excuses. I can finally build avr using the latest (at time of writing) gcc and friends. 

*Fun Fact2* So it looks like an older avr-gcc is also available with the Arduino IDE...*sigh* At least I learnt something ;-)


