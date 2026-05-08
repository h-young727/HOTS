# HOTS - Hunter's Operating System

Welcome to my x86-32 operating system kernel written in C++, built as a self-learning project. Using the [OSDev Bare Bones](https://wiki.osdev.org/Bare_Bones) tutorial as a starting point, I'm aiming to understand and implement the fundamentals of OS kernel development, from bootloaders to interrupts and more. My OS targets i686, a subset of the x86-32 ISA, and for now boots via GRUB on QEMU.

---

## Environment Setup

The following steps document the full environment setup required to build and run this project, both for my own reference and so that I can repeat the steps on any machine.

```bash
sudo apt install -y build-essential bison flex make diffutils libgmp-dev libmpfr-dev libmpc-dev libisl-dev texinfo nasm qemu-system-x86 grub-pc-bin grub-common xorriso mtools
```
Installs all system dependencies required to build the cross-compiler and run the OS.

- `build-essential` -- meta-package providing `gcc`, `g++`, `make`, and other core build tools needed to compile software from source
- `bison` -- parser generator required by GCC's build system
- `flex` -- lexical analyzer required by GCC's build system
- `make` -- build automation tool that runs Makefile recipes
- `diffutils` -- provides `diff` and `cmp`, used by GCC's bootstrapping process
- `libgmp-dev` -- GNU Multiple Precision library, required by GCC for arbitrary precision arithmetic
- `libmpfr-dev` -- Multiple Precision Floating-Point library, required by GCC for floating point constant folding
- `libmpc-dev` -- GNU MPC complex number library, required by GCC
- `libisl-dev` -- Integer Set Library, used by GCC for loop optimization
- `texinfo` -- documentation system required by GCC's build process
- `nasm` -- Netwide Assembler, assembles `boot.asm` into an ELF object file
- `qemu-system-x86` -- emulates a complete x86 machine in software for development and testing
- `grub-pc-bin` -- GRUB bootloader binaries packaged into the bootable ISO
- `grub-common` -- provides `grub-mkrescue`, which assembles the kernel and GRUB into a bootable ISO
- `xorriso` -- ISO creation tool used internally by `grub-mkrescue`
- `mtools` -- MS-DOS filesystem utilities required by `grub-mkrescue`

---

```bash
export PREFIX="$HOME/opt/cross"
```
Sets the installation directory for the cross-compiler. All cross-compiler binaries will be installed to `~/opt/cross/bin/`.

```bash
export TARGET=i686-elf
```
Sets the target triple. `i686` is 32-bit x86. `elf` means produce ELF binaries with no OS-specific assumptions. The resulting tools will be prefixed `i686-elf-` (e.g. `i686-elf-g++`).

```bash
export PATH="$PREFIX/bin:$PATH"
```
Prepends the cross-compiler bin directory to PATH so tools like `i686-elf-g++` are callable by name from anywhere in the current session.

```bash
mkdir -p ~/src && cd ~/src
```
Creates and enters a temporary build workspace for downloading and compiling the cross-compiler source. Can be deleted after installation.

```bash
wget https://ftp.gnu.org/gnu/binutils/binutils-2.46.0.tar.gz
tar -xzf binutils-2.46.0.tar.gz
mkdir build-binutils && cd build-binutils
```
Downloads and extracts binutils 2.46.0, then creates a separate build directory. Binutils provides the cross-linker (`i686-elf-ld`) and related tools targeting bare metal i686.

```bash
../binutils-2.46.0/configure --target=$TARGET --prefix="$PREFIX" --with-sysroot --disable-nls --disable-werror
```
Configures the binutils build for cross-compilation.
- `--target=$TARGET` -- produces tools that output `i686-elf` code, not host machine code
- `--prefix="$PREFIX"` -- installs into `~/opt/cross` instead of system directories
- `--with-sysroot` -- tells the linker not to search host system library paths
- `--disable-nls` -- disables translation support, speeds up the build
- `--disable-werror` -- prevents compiler warnings from failing the build

```bash
make && make install
```
Compiles binutils from source and installs it into `~/opt/cross/bin/`.

```bash
cd ~/src
wget https://ftp.gnu.org/gnu/gcc/gcc-15.2.0/gcc-15.2.0.tar.gz
tar -xzf gcc-15.2.0.tar.gz
mkdir build-gcc && cd build-gcc
```
Downloads and extracts GCC 15.2.0, then creates a separate build directory. GCC provides `i686-elf-gcc` and `i686-elf-g++`, the compilers used to build all kernel C++ code into bare metal i686 binaries.

```bash
../gcc-15.2.0/configure --target=$TARGET --prefix="$PREFIX" --disable-nls --enable-languages=c,c++ --without-headers --disable-hosted-libstdcxx
```
Configures the GCC build for bare metal cross-compilation.
- `--target=$TARGET` -- produces a compiler that outputs `i686-elf` machine code
- `--prefix="$PREFIX"` -- installs into `~/opt/cross`
- `--disable-nls` -- disables translation support
- `--enable-languages=c,c++` -- builds both `i686-elf-gcc` and `i686-elf-g++`
- `--without-headers` -- tells GCC not to rely on any system headers since the OS has none
- `--disable-hosted-libstdcxx` -- disables the full C++ standard library, which requires an OS underneath it

```bash
make -j$(nproc) all-gcc
```
Compiles the GCC compiler itself, parallelized across all available CPU cores.

```bash
make -j$(nproc) all-target-libgcc
```
Builds a minimal GCC runtime library providing low-level operations like integer division that the compiler may emit calls to.

```bash
make -j$(nproc) all-target-libstdc++-v3
```
Builds a freestanding (no-OS) version of the C++ standard library.

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
./configure
make -j$(nproc)
sudo make install
```
Downloads, builds, and installs NASM 3.01 from source. Built from source to get the latest version, as the apt package is outdated.

```bash
echo 'export PATH="$HOME/opt/cross/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
Makes the cross-compiler permanently available in all future terminal sessions. Without this, the PATH exports above only last for the current session.

```bash
rm -rf ~/src
```
Deletes the temporary build workspace. The cross-compiler is fully installed into `~/opt/cross` and the source directories are no longer needed.

---

## Verify Installation

```bash
i686-elf-gcc --version     # 15.2.0
i686-elf-g++ --version     # 15.2.0
nasm --version              # 3.01
qemu-system-x86_64 --version
grub-mkrescue --version
```

---

## References

- [OSDev Wiki -- Bare Bones](https://wiki.osdev.org/Bare_Bones)
- [OSDev Wiki -- GCC Cross Compiler](https://wiki.osdev.org/GCC_Cross-Compiler)
- [NASM Documentation](https://www.nasm.us/docs.php)