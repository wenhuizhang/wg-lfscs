# Quick Start for eBPF


Linux kernel supports 3 eBPF instruction sets, differs in impact on program size and performance. Set mcpu=probe to use the newest supported version.
- v1 only supports greater-than jumps.
- v2 only supports both greater-than jumps and lower-than jumps.
- v3, adds 32-bit variants of the existing conditional 64-bit jumps.


This document is for LLVM 10.0.1
```
$ llc -march=bpf -mcpu=help
Available CPUs for this target:

  generic - Select the generic processor.
  probe   - Select the probe processor.
  v1      - Select the v1 processor.
  v2      - Select the v2 processor.
  v3      - Select the v3 processor.

Available features for this target:

  alu32    - Enable ALU32 instructions.
  dummy    - unused feature.
  dwarfris - Disable MCAsmInfo DwarfUsesRelocationsAcrossSections.
  
Use +feature to enable a feature, or -feature to disable it.
For example, llc -mcpu=mycpu -mattr=+feature1,-feature2

```

## 1. Install Dependencies

### 0. 1. Update Linux Kernel

```
sudo apt-get install make cmake libncurses5-dev flex bison libssl-dev dkms libelf-dev git

git clone https://github.com/torvalds/linux.git
cd linux/
git tag
git checkout v5.10
make config
make -j4
sudo make modules_install
sudo make install
sudo update-grub2
sudo reboot
```

### 1. install cmake
```
sudo apt-get install libssl-dev
git clone https://github.com/Kitware/CMake.git
cd CMake/
git checkout release
./bootstrap && make && sudo make install 
```
### 2. install python3
```
wget https://www.python.org/ftp/python/3.10.2/Python-3.10.2.tgz
tar -xvf Python-3.10.2.tgz
cd Python-3.10.2/
./configure
make
make test
sudo make install
python3 --version
which python3
```

## 2. Install LLVM

```
sudo apt-get install cmake ninja-build gcc
sudo apt install clang-11 --install-suggests


git clone https://github.com/llvm/llvm-project.git
cd llvm-project
git checkout llvmorg-10.0.1
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS=clang "Unix Makefiles" ../llvm
cmake -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm
make -j4
sudo cmake --build . --target install
ls ./llvm-project/build/bin/
```
The binaries are located in `./llvm-project/build/bin/`


## 3. Compile BPF Program 

To use LLVM LLC to compile your BPF program.
```
clang -O2 -Wall -target example -emit-llvm -c example.c -o example.bc
lli example.bc
llc example.bc -march=bpf -mcpu=probe -filetype=obj -o example.o
readelf -x .text example.o
```
-mcpu parameter defaults to generic, an alias for v1, the oldest instruction set. probe will select the newest instruction set your kernel supports.

## 4. Load BPF Program

To load userpace programs to kernel space, you can use bpftool based on libbpf.

The source code for bpftool can be found in the Linux kernel repository, under tools/bpf/bpftool

1. Install bpftool binary
```
sudo apt install linux-tools-common linux-tools-generic
sudo apt-get install libz-dev
```
2. Install bpftool from source

```
sudo apt-get install libz-dev
git clone https://github.com/torvalds/linux.git
cd linux
git checkout v5.15
cd tools/bpf/bpftool
make
make install
make doc doc-install
```

## 5. Load Program
```
bpftool -V
bpftool version -p

bpftool prog show
bpftool prog list

bpftool prog load example.o /sys/fs/bpf/example

bpftool prog show --json id 3
bpftool prog dump xlated id 3
bpftool prog dump jited pinned /sys/fs/bpf/example

bpftool prog show --bpffs
bpftool -f map

bpftool map show

bpftool prog pin id 27 /sys/fs/bpf/example_prog
rm /sys/fs/bpf/example_prog

bpftool prog tracelog
cat /sys/kernel/debug/tracing/trace_pipe
```

Once loaded, programs of the relevant types can be attached to sockets with:
```
# bpftool prog attach <program> <attach type> <target map>
```


For attaching programs to cgroup, the command differs:
```
# bpftool cgroup attach <cgroup> <attach type> <program> [flags]
```

And like ip link, bpftool can attach programs to the XDP hook (and later detach them):
```
# bpftool net attach xdp id 42 dev eth0
# bpftool net detach xdp dev eth0
```
The xdpgeneric/xdpdrv/xdpoffload variants for generic XDP (a.k.a SKB XDP), driver XDP (a.k.a native XDP), or XDP hardware offload, are also supported.


## 6. Reusing Maps
Load a program, but reuse for example two existing maps (instead of automatically creating new ones):
```
# bpftool prog load example.o /sys/fs/bpf/example_prog \
        map idx 0 id 27 \
        map name stats pinned /sys/fs/bpf/stats_map
```
where idx 0 is the index of the map in the ELF program file.


## 7. Loading Several Programs
For object files with more than one program, bpftool can load all of them at once:
```
# bpftool prog loadall example_flow.o /sys/fs/bpf/flow type flow_dissector
```
This is especially useful when working with tail calls. Maps can be pinned by adding pinmaps `<directory path in bpffs>`.
  
  
## 8. Test Runs
  
`BPF_PROG_TEST_RUN` is a command for the bpf() system call. 
  
It is used to manually trigger a “test” run for a program loaded in the kernel, with specific input data (for example: packet data) and context (for example: struct __sk_buff). 
  
It returns the output data and context, the return value of the program, and the duration of the execution. 
  
Although this feature is not available to all program types, bpftool can use it to test-run programs:
```
# bpftool prog run PROG data_in <file> data_out <file>
```
  
## 9. Profiling Programs

Recent bpftool versions can attach programs (of types “fentry” or “fexit”) to the entry or exit of other eBPF programs and use perf events to collect statistics on them.

```
# bpftool prog profile <prog> <metrics>
```
  
This requires that the kernel running on the system has been compiled with BTF information, and bpftool with the use of a “skeleton”.
Here is another example, featuring two metrics that were more recently added: ITLB and DTLB misses for a running eBPF program.

```
# bpftool prog profile <prog> itlb_misses dtlb_misses
```
  
  
## 10. Program Stat

Linux 5.1 introduced statistics for attached eBPF programs: it can collect the total run time and the run count for each program. 

When available, this information is displayed by bpftool when listing the programs:
```
# bpftool prog show
```
Gathering statistics impacts performance of  program execution by ~10 to 30 nanoseconds per run, so it is disabled by default. 

Activate it with:
```
# sysctl -w kernel.bpf_stats_enabled=1
```

## 11. Managing with libbpf

libbpf: https://github.com/libbpf/libbpf

```
git clone https://github.com/libbpf/libbpf.git
cd libbpf
cd src
mkdir build root
BUILD_STATIC_ONLY=y OBJDIR=build DESTDIR=root make install
PKG_CONFIG_PATH=/build/root/lib64/pkgconfig DESTDIR=/build/root make install
```

## 12. Edit BPF Bytecode

Instead of compiling an eBPF program from C to an ELF object file, we could compile it to an assembly language, edit it accordingly, then assemble this version as the final object file. 

1. Source Code

```
$ cat  example.c
int func()
{
        return 0;
}
```

2. Compiling from C to eBPF Assembly
```
$ clang -target example -S -o example.s example.c
$ cat example.s
        .text
        .globl        func                    # -- Begin function func
        .p2align        3
func:                                   # @func
# %bb.0:
        r1 = 0
        *(u32 *)(r10 - 4) = r1
        r0 = r1
        exit
        
```
              
3. Modify
```
$ sed -i '$a \\tr0 = 3' example.s
$ cat example.s
        .text
        .globl        func                    # -- Begin function func
        .p2align        3
func:                                   # @func
# %bb.0:
        r1 = 0
        *(u32 *)(r10 - 4) = r1
        r0 = r1
        exit
                                        # -- End function

        r0 = 3
# Assembling to an ELF Object File
 llvm-mc -triple example -filetype=obj -o example.o example.s
# dump the bytecode
$ readelf -x .text example.o

Hex dump of section '.text':
  0x00000000 b7010000 00000000 631afcff 00000000 ........c.......
  0x00000010 bf100000 00000000 95000000 00000000 ................
  0x00000020 b7000000 03000000 b7000000 03000000 ................
$ llvm-objdump -d example.o
$ llvm-objdump -S example.o         # add C code, if -g was passed to clang

bpf.o:        file format ELF64-BPF

Disassembly of section .text:
func:
       0:        b7 01 00 00 00 00 00 00         r1 = 0
       1:        63 1a fc ff 00 00 00 00         *(u32 *)(r10 - 4) = r1
       2:        bf 10 00 00 00 00 00 00         r0 = r1
       3:        95 00 00 00 00 00 00 00         exit
       4:        b7 00 00 00 03 00 00 00         r0 = 3
```


## 13. Add Debug Symbols  
```
$ clang -target example -g -S -o example.s example.c
$ llvm-mc -triple example -filetype=obj -o example.o example.s
$ llvm-objdump -S example.o
```

## 14. Install BCC

```
sudo apt-get install arping bison clang-format cmake dh-python dpkg-dev pkg-kde-tools ethtool flex inetutils-ping iperf   libbpf-dev libclang-dev libedit-dev libelf-dev libfl-dev libzip-dev linux-libc-dev llvm-dev libluajit-5.1-dev   luajit python3-netaddr python3-pyroute2
sudo apt install -y bison build-essential cmake flex git libedit-dev   libllvm11 llvm-11-dev libclang-11-dev python zlib1g-dev libelf-dev libfl-dev
sudo apt install clang-11 --install-suggests
sudo apt-get -y install luajit luajit-5.1-dev

git clone https://github.com/iovisor/bcc.git
mkdir bcc/build; cd bcc/build
cmake ..
make
sudo make instal
```
## 15. Reference

1. An assembler for eBPF programs written in Intel-like assembly syntax. https://github.com/Xilinx-CNS/ebpf_asm

3. BPF commands https://qmonnet.github.io/whirl-offload/2021/09/23/bpftool-features-thread/
