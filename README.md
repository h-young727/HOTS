# HOTS - Hunter's Operating System

Welcome to my x86-32 operating system kernel written in C++! This is purely a self-learning project, the motivation for which being nothing more than simply wanting to better understand how computers work. I began this journey with [OSDev Bare Bones](https://wiki.osdev.org/Bare_Bones), which seemed to provide an excellent starting-point tutorial. By the end this project, I want to thoroughly understand and know how to implement the fundamental requirements of an operating system kernel.

## Disclaimer

This README probably goes into far too unnecessary detail for most readers, but I wanted it to function as both an introduction to my project, as well as a unified reference for me in case I forget any of the numerous details of operating system development. As such, I will be assuming nothing about the reader's knowledge, and explain every single acronym.

## Prerequisite Knowledge
Simply put, an operating system (OS) is a type of software that acts as the interface between a computer's hardware and the software available for users. Since an OS is just software, it of course needs hardware to run on, and there are three core software components that work in tandem to start an OS. These components are the bootloader, kernel entry point, and kernel. To keep things simple, the bootloader and kernel entry point exist to start the kernel, and the kernel can be thought of as the main piece of the OS, or the OS itself.

For this project, I will be using the GRand Unified Bootloader (GRUB), which is a well-established pre-packaged bootloader provided by GNU's Not Unix (GNU). GNU is a free and open-source software project that produces many useful tools for OS development, among many other things. The kernel entry point is written in assembly using the Netwide Assembler (NASM), a popular assembler for Intel x86 architectures due to its compatibility with 16-, 32-, and 64-bit systems and its use of Intel-style syntax. NASM takes assembly source code as input and produces machine code as output.

x86 is the name of the instruction set architecture (ISA) primarily in use today by Intel and AMD central processing units (CPUs), and an ISA is the set of instructions a CPU is designed to understand and execute. This is important to know as the ISA of the CPU you want your OS to run on will affect how you design the OS. You also need to know the size of the CPU's registers. Registers are the most fundamental unit of memory on a CPU, and the "32" in "x86-32" refers to the number of bits a CPU register can hold. On a 32-bit system, a register can hold a 32-bit value, and when that value is interpreted as a memory address, it can point to 2^32 unique locations in memory. It is worth noting that each of those memory locations contains one byte of data.

My OS targets i686, which is a specific subset of x86-32. For testing the OS during development, I will use Quick Emulator (QEMU), which is software that emulates x86 hardware, allowing the OS to be developed and tested without running on real hardware.

Now it might have occurred to you that while writing the collection of C++ source code that is the kernel, you can't really compile that code on the OS hosting the development. Well, you could, but the resulting binary would contain assumptions about the host OS underneath it, which won't necessarily hold for your kernel. For this reason, we use a cross-compiler, which is a compiler that runs on your host machine but produces binaries targeting a completely different system. In this case, the cross-compiler targets i686-elf, meaning it produces 32-bit x86 machine code in the Executable and Linkable Format (ELF), and with zero assumptions about the OS. 

After this, the kernel entry point and kernel are each compiled into ELF object files, which are then linked together into a single ELF binary. This binary is then packaged alongside GRUB into a bootable disk image in the International Organization for Standardization (ISO) format. A disk image is a file that represents the complete state of a physical storage device like a hard drive (HDD) or solid state drive (SSD). When your computer or emulator boots, the Basic Input/Output System (BIOS), which is motherboard firmware (specialized software pre-installed on hardware), reads the first 512 bytes of the disk, known as the Master Boot Record (MBR), which will contain the initial GRUB binary. GRUB then finds the kernel ELF binary within the same disk image, loads it into memory, and hands control to the kernel entry point, which can finally start the OS.

## Environment Setup

Below are all of the steps required to set up the development environment and build and run the OS.

```bash
sudo apt install -y build-essential bison flex diffutils libgmp-dev libmpfr-dev libmpc-dev libisl-dev texinfo nasm qemu-system-x86 grub-pc-bin grub-common xorriso mtools
```
Installs all system dependencies required to build the cross-compiler and run the OS.

- `build-essential` -- provides core build tools required to automate build processes and compile software, including Make and the GNU Compiler Collection
- `bison`, `flex`, `diffutils` -- parser generator, lexical analyzer, and file comparison tools required to build GCC
- `libgmp-dev`, `libmpfr-dev`, `libmpc-dev`, `libisl-dev` -- numerical libraries required by GCC
- `texinfo` -- documentation format required by GCC
- `nasm` -- assembles kernel entry point bootstrap assembly code into an ELF file
- `qemu-system-x86` -- emulates an x86 machine in software for development testing
- `grub-pc-bin`, `grub-common` -- GRUB bootloader binaries and tooling, including `grub-mkrescue` which packages the kernel ELF binary alongside GRUB into a bootable ISO disk image
- `xorriso`, `mtools` -- provides tools required by `grub-mkrescue` to create the ISO

Before building the cross-compiler, we need to set three environment variables that the build process will reference throughout.

```bash
export PREFIX="$HOME/opt/cross"
```
Sets the installation directory for the cross-compiler.
```bash
export TARGET=i686-elf
```
Sets the target architecture and binary format for the cross-compiler.

```bash
export PATH="$PREFIX/bin:$PATH"
```
Adds the cross-compiler to PATH so its tools are callable by name from anywhere in the current terminal session.

```bash
mkdir -p ~/src && cd ~/src
```
Creates and enters a temporary directory for downloading and compiling the cross-compiler source (can be deleted after installation).

```bash
wget https://ftp.gnu.org/gnu/binutils/binutils-2.46.0.tar.gz
tar -xzf binutils-2.46.0.tar.gz
mkdir build-binutils && cd build-binutils
```
Downloads and extracts the concurrent latest binutils release and creates a dedicated build directory. Binutils provides the tools to link ELF binaries and more.

```bash
../binutils-2.46.0/configure --target=$TARGET --prefix="$PREFIX" --with-sysroot --disable-nls --disable-werror
```
Configures the binutils build for cross-compilation.
- `--target=$TARGET` -- produces tools that output `i686-elf` machine code
- `--prefix="$PREFIX"` -- installs into `~/opt/cross`
- `--with-sysroot` -- enables sysroot support, giving the linker a defined root to search for libraries rather than falling back to host system paths
- `--disable-nls` -- tells binutils not to include native language support, thereby speeding up the build
- `--disable-werror` -- prevents compiler warnings from crashing the build

```bash
make && make install
```
Compiles binutils and installs it into `~/opt/cross/bin/`.

```bash
cd ~/src
wget https://ftp.gnu.org/gnu/gcc/gcc-15.2.0/gcc-15.2.0.tar.gz
tar -xzf gcc-15.2.0.tar.gz
mkdir build-gcc && cd build-gcc
```
Downloads and extracts the concurrent latest GCC release and creates a dedicated build directory. GCC provides the compilers used to build all kernel C++ code into i686 binaries.

```bash
../gcc-15.2.0/configure --target=$TARGET --prefix="$PREFIX" --disable-nls --enable-languages=c,c++ --without-headers --disable-hosted-libstdcxx
```
Configures the GCC build for bare metal cross-compilation.
- `--target=$TARGET` -- produces a compiler that outputs `i686-elf` machine code
- `--prefix="$PREFIX"` -- installs into `~/opt/cross`
- `--disable-nls` -- tells GCC not to include native language support, thereby speeding up the build
- `--enable-languages=c,c++` -- builds both `i686-elf-gcc` and `i686-elf-g++`, the i686 compilers for C and C++, respectively
- `--without-headers` -- tells GCC not to rely on any language library headers since the OS has none
- `--disable-hosted-libstdcxx` -- disables the full C++ standard library, as it makes assumptions about the underlying OS

```bash
make -j$(nproc) all-gcc
```
Compiles the GCC compiler for the target system, parallelized across all available CPU cores for a faster build.

```bash
make -j$(nproc) all-target-libgcc
```
Builds the GCC runtime library for the target system, parallelized across all available CPU cores for a faster build.

```bash
make -j$(nproc) all-target-libstdc++-v3
```
Builds a version of the C++ standard library for the target system, parallelized across all available CPU cores for a faster build.

```bash
make install-gcc
make install-target-libgcc
make install-target-libstdc++-v3
```
Installs each component into `~/opt/cross/bin/`.

```bash
cd ~/src
wget https://www.nasm.us/pub/nasm/releasebuilds/3.01/nasm-3.01.tar.gz
tar -xzf nasm-3.01.tar.gz
cd nasm-3.01
```
Downloads and extracts the concurrent latest NASM release.

```bash
./configure
```
Configures the NASM build. No special flags are needed since NASM is a host tool.

```bash
make -j$(nproc) && sudo make install
```
Compiles NASM and installs it into `/usr/local/bin/`.

```bash
echo 'export PATH="$HOME/opt/cross/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
Makes the cross-compiler permanently available in all future terminal sessions.

```bash
rm -rf ~/src
```
Deletes the temporary build directory. The cross-compiler should now be fully installed into `~/opt/cross`, so the source directories are no longer needed.

Verify that the following have installed correctly and are up to date:

```bash
i686-elf-gcc --version
i686-elf-g++ --version
nasm --version
qemu-system-i386 --version
grub-mkrescue --version
```

## The Part After The Environmnt Setup