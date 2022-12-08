# Neutron clang

![Kernel Builds](https://github.com/Neutron-Toolchains/linux-kernel-build-tester/actions/workflows/workflow.yml/badge.svg)

This is a [LLVM](https://llvm.org/) and [Clang](https://clang.llvm.org/) compiler toolchain that is built for kernel development. Builds are always made from the latest LLVM sources rather than stable releases, so complete stability cannot be guaranteed.

This toolchain targets the AArch32 & AArch64 architectures. It is built with LTO, POLLY, PGO, BOLT and O3 optimizations to reduce compile times as much as possible. [Polly](https://polly.llvm.org/), LLVM's polyhedral loop optimizer, is also included for users who want to experiment with additional optimization. Note that this toolchain is **not** suitable for anything other than bare-metal development; it has not been built with support for any libc or userspace development in mind.

[binutils](https://www.gnu.org/software/binutils/) is also included for convenience. Neutron Clang uses binutils source from the [HEAD of the latest release branch](https://sourceware.org/git/?p=binutils-gdb.git;a=shortlog;h=refs/heads/binutils-2_39-branch), which allows us to have more upto date binutils compared to latest stable release as well as avoiding instability or breakage compared to bleeding edge binutils. This means that **users do not need to download separate GCC toolchains** to build the Linux kernel.

Every Saturday at 8 AM IST, Automated builds are performed using fresh sources from the LLVM Git [monorepo](https://github.com/llvm/llvm-project). The finished build tars are released in our [catalogue](https://github.com/Neutron-Toolchains/clang-build-catalogue) repository. The catalogue repository will not be updated if any of the builds fail. You can find the build scripts for this [here](https://github.com/Neutron-Toolchains/clang-build). To know how to install and manage Neutron clang builds, look at [How to install](#how-to-install)

Build notifications and other information can be obtained from the [Telegram channel](https://t.me/neutron_updates).

## How to install

**[AntMan](https://github.com/Neutron-Toolchains/antman.git)** `(A Nonsensical Toolchain Manager)` is a manager written in bash, It is used by Neutron Clang to download/sync, upgrade and manage toolchain builds. AntMan script can be found [here](https://github.com/Neutron-Toolchains/antman/blob/main/antman).

Here's an exmaple on how to sync latest build using AntMan:
```bash
mkdir -p "$HOME/toolchains/neutron-clang"
cd "$HOME/toolchains/neutron-clang"
curl -LO "https://raw.githubusercontent.com/Neutron-Toolchains/antman/main/antman"
./antman -S
```

AntMan can also be ran without actually downloading the script:
```bash
mkdir -p "$HOME/toolchains/neutron-clang"
cd "$HOME/toolchains/neutron-clang"
bash <(curl -s "https://raw.githubusercontent.com/Neutron-Toolchains/antman/main/antman") -S
```

Some more AntMan commands:
- To sync latest toolchain build. `./antman -S` or `./antman -S=latest`
- To sync a specific toolchain release. `./antman -S=<release tag>`
- To check for updates. `./antman -U`
- To check for updates and sync update. `./antman -Uy`
- To sync a specific update. `./antman -Uy=<release tag>`
- To delete synced build. `./antman -D`
- To show information on synced build. `./antman -I`

Run `./antman --help` for more information about AntMan.

## Building Linux

Make sure you have this toolchain in your `PATH`:

```bash
export PATH="$HOME/toolchains/neutron-clang/bin:$PATH"
```

For an AArch64 cross-compilation setup, you must set the following variables. Some of them can be environment variables, but some must be passed directly to `make` as a command-line argument. It is recommended to pass **all** of them as `make` arguments to avoid confusing errors:

- `CC=clang` (must be passed directly to `make`)
- `CROSS_COMPILE=aarch64-linux-gnu-`
- If your kernel has a 32-bit vDSO: `CROSS_COMPILE_ARM32=arm-linux-gnueabi-`

Note: Android kernels 4.19 and newer use the upstream variable `CROSS_COMPILE_COMPAT`. When building these kernels, replace `CROSS_COMPILE_ARM32` in your commands and scripts with `CROSS_COMPILE_COMPAT`.

Optionally, you can also choose to use as many LLVM tools as possible to reduce reliance on binutils. All of these must be passed directly to `make`:

- `AR=llvm-ar`
- `NM=llvm-nm`
- `OBJCOPY=llvm-objcopy`
- `OBJDUMP=llvm-objdump`
- `STRIP=llvm-strip`

Note: Android 4.19 and newer kernels support additional flags for clang
- `LLVM=1`     - Enables use of llvm binutils
- `LLVM_IAS=1` - Enables use of clang's integrated assembler

Note, however, that additional kernel patches may be required for these LLVM tools to work. It is also possible to replace the binutils linkers (`lf.bfd` and `ld.gold`) with `lld` and use Clang's integrated assembler for inline assembly in C code, but that will require many more kernel patches and it is currently impossible to use the integrated assembler for *all* assembly code in the kernel.

Android kernels older than 4.14 will require patches for compiling with any Clang toolchain to work; those patches are out of the scope of this project. See [android-kernel-clang](https://github.com/nathanchance/android-kernel-clang) for more information.

### Differences from other toolchains

Neutron Clang has been designed to be easy-to-use compared to other toolchains, such as [AOSP Clang](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/). The differences are as follows:

- `CLANG_TRIPLE` does not need to be set because we don't use AOSP binutils
- `LD_LIBRARY_PATH` does not need to be set because we set library load paths in the toolchain
- No separate GCC/binutils toolchains are necessary; all tools are bundled

## Common problems

### `as: unrecognized option '-EL'`

Usually, this means that `CROSS_COMPILE` is not set correctly. Check that variable as well as your `PATH` to make sure that the required tools are available for Clang to invoke. You can test it manually like tihs:

```bash
$ ${CROSS_COMPILE}ld -v
GNU ld (GNU Binutils) 2.39
```

If you see `Command not found` or any other error, one of the two variables mentioned above is most likely set incorrectly.

If you continue to encounter this error after verifying `CROSS_COMPILE` and `PATH`, you are probably running into a change in Clang 12's handling of cross-compiling. The fix is to either merge linux-stable (which already has the fix included) cherry-pick ["Makefile: Fix GCC_TOOLCHAIN_DIR prefix for Clang cross compilation"](https://github.com/kdrag0n/proton_zf6/commit/6e87fec9a3df5) manually.

If the error still continues to appear and you have an arm64 kernel that includes vdso32, you will also need to cherry-pick ["arm64: vdso32: Fix '--prefix=' value for newer versions of clang"](https://github.com/kdrag0n/proton_zf6/commit/68acd6966ac98) to fix the second vdso32 error. Merging linux-stable is also an option if you are on Linux 5.4 or newer.

### `Cannot use CONFIG_CC_STACKPROTECTOR_STRONG: -fstack-protector-strong not supported by compiler`

This error is actually a result of stack protector checks in the Makefile that cause the real error to be masked. It indicates that one or more unsupported compiler flags have been passed to Clang. Disable `CONFIG_CC_STACKPROTECTOR_STRONG` in favor of `CONFIG_CC_STACKPROTECTOR_NONE` temporarily (make sure you don't keep this change permanently for security reasons) and Clang will output the flag that is causing the problem.

In downstream kernels, the cause is usually big.LITTLE CPU optimization flags that have been added to the Makefile unconditionally. The correct solution is to check `cc-name` for the current compiler and adjust the optimization flags accordingly — big.LITTLE for GCC and little-only for Clang. See ["Makefile: Optimize for sm8150's Kryo 485 CPU setup"](https://github.com/kdrag0n/proton_zf6/commit/f45e4ffbecd1c059aa49d8a119b50ee84d7f9d0f) for an example implementation of this.

### `scripts/gcc-version.sh: line 25: aarch64-linux-gnu-gcc: command not found`

This is caused by unconditional invocations of `gcc-version.sh` in CAF's camera_v2 driver. The recommended solution is to cherry-pick [Google's fix](https://android.googlesource.com/kernel/msm/+/9b3a54e388fae0fcc5ea64a4c612936baae44fce) from the Pixel 2 kernel, which simply removes the faulty invocations as they were never useful to begin with.

Note that these errors are harmless and don't necessarily need to be fixed, but nonetheless, ignoring them is not recommended.

### `undefined reference to 'stpcpy'`

This is caused by a libcall optimization added in Clang 12 that optimizes certain basic `sprintf` calls into `stpcpy` calls. The correct fix for this is to cherry-pick ["lib/string.c: implement stpcpy"](https://github.com/kdrag0n/proton_zf6/commit/cec73f0775526), which adds a simple implementation of `stpcpy` to the kernel so that Clang can use it.

### `clang: /lib/x86_64-linux-gnu/libc.so.6: version GLIBC_2.36 not found (required by clang)`

This means that your linux distro is running old glibc libs. As Neutron clang is built on latest ArchLinux image, its linked against latest glibc libs. This breaks compatibility for users using older glibc.

Currently to fix this issue you can do two things:
Either upgrade your glibc package if your distro is providing an update.

OR

Try using this workaround [script](https://gist.github.com/dakkshesh07/240736992abf0ea6f0ee1d8acb57a400) which once executed would download latest [glibc package from ArchLinux](https://archlinux.org/packages/core/x86_64/glibc) repo, put it under `$HOME/glibc` and and patch your clang binaries using `patchelf` to use the ArchLinux glibc libs rather relying on your host glibc libs.
