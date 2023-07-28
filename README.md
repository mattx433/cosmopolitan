![Cosmopolitan Honeybadger](usr/share/img/honeybadger.png)

[![build](https://github.com/jart/cosmopolitan/actions/workflows/build.yml/badge.svg)](https://github.com/jart/cosmopolitan/actions/workflows/build.yml)
# Cosmopolitan (Build on Windows)

**This is an (unsupported) fork that makes it possible to build Cosmopolitan natively on a Windows system.**

## Build instructions
Note that you should be using the Command Prompt (`cmd.exe`) and not Windows PowerShell.
```
curl -LO https://github.com/mattx433/cosmopolitan/archive/refs/heads/build-on-windows.zip
tar -xvf build-on-windows.zip
REM Alternatively, assuming you have Git for Windows installed:
REM git clone https://github.com/mattx433/cosmopolitan cosmopolitan-build-on-windows -b build-on-windows
cd cosmopolitan-build-on-windows
mkdir o\third_party\gcc
cd o\third_party\gcc
curl -LO https://github.com/ahgamut/musl-cross-make/releases/download/z0.0.0/gcc11.zip
tar -xvf gcc11.zip
del gcc11.zip
cd bin
REM This needs to be set for gcc to work
set PATH=%PATH%;%CD%
cd ..\..\..\..\
build\bootstrap\make.com V=0 -j8
```

## Build notes
- Building will take a while, as process creation and file operations are expensive on Windows.
  Expect to wait at least 30 minutes, even if you have a fast processor.
- CTRL+C during the build process is unreliable, so it's best not to use it. Issues include:
  - Incomplete files being left on disk. This can be spotted with `file not recognized`, `file is truncated`, `file is empty` errors,
    in which case you can delete the bad file with `del` or in worse situations, delete its directory with `rd /s`.
  - `make.com` might get stuck after all processes have finished. In this case, open Task Manager and kill the make process.
- `failed to move output file` errors often happen when running make with a high job count. Either try again or restart with `-j1`.
- These warnings are safe to ignore:
  - `Makefile:91: please run ape/apeinstall.sh if you intend to use landlock make`
  - `make.com: sleep: command not found on $PATH: ENOENT`
## Other notes
- Remember that you should use backslashes when typing the path to a binary, and regular slashes when typing paths in arguments.
  For example: `build\bootstrap\make.com o//tool/net/dig.com`
- After running make or any Cosmopolitan program, your console window will probably work oddly:
  - Arrow keys will output the escape sequence into the console.
  - Opening the command history window with F7 won't work.
  - Backspace will delete entire words, you can work around this by holding Ctrl.
- The following modifications have been done to make the build work on Windows:
  - `build/bootstrap` - Binaries have been updated to the latest as of July 28, 2023.
    They can be download independently [here](https://justine.lol/cosmo-bootstrap.zip).
  - `build/definitions.mk` - Build tool names have been adjusted for Actually Portable GCC.
  - `test/libc/calls/readlinkat_test.c` and `test/libc/calls/symlinkat_test.c` - Removed as creating symlinks as a regular user is unreliable.
    - Optionally, `libc/calls/symlinkat-nt.c` can be replaced with [this copy](https://justine.lol/symlinkat-nt.c) for some debugging.
  - `test/libc/mem/test.mk` - `assimilate.com` arguments have been changed to force assimilating one binary to ELF.
  - `test/libc/stdio/posix_spawn_test.c` - Removed, as the `wait4` system call implementation is messy on Windows and does not pass the test.
  - `test/tool/net/lunix_test.lua` - Removed, as an asseration fails due to `chmod` and `stat` not returning consistent outputs.
  - `third_party/python/Lib/test/signalinterproctester.py` - Removed, for similar reasons as `posix_spawn_test.c`
  - `third_party/python/python.mk` - Removed the above test from the list of tests to run.

# Cosmopolitan
[Cosmopolitan Libc](https://justine.lol/cosmopolitan/index.html) makes C
a build-once run-anywhere language, like Java, except it doesn't need an
interpreter or virtual machine. Instead, it reconfigures stock GCC and
Clang to output a POSIX-approved polyglot format that runs natively on
Linux + Mac + Windows + FreeBSD + OpenBSD + NetBSD + BIOS with the best
possible performance and the tiniest footprint imaginable.

## Background

For an introduction to this project, please read the [αcτµαlly pδrταblε
εxεcµταblε](https://justine.lol/ape.html) blog post and [cosmopolitan
libc](https://justine.lol/cosmopolitan/index.html) website. We also have
[API documentation](https://justine.lol/cosmopolitan/documentation.html).

## Getting Started

It's recommended that Cosmopolitan be installed to `/opt/cosmo` and
`/opt/cosmos` on your computer. The first has the monorepo. The second
contains your non-monorepo artifacts.

```sh
sudo mkdir -p /opt
sudo chmod 1777 /opt
git clone https://github.com/jart/cosmopolitan /opt/cosmo
cd /opt/cosmo
make -j8 toolchain
ape/apeinstall.sh  # optional
mkdir -p /opt/cosmos/bin
export PATH="/opt/cosmos/bin:$PATH"
echo 'PATH="/opt/cosmos/bin:$PATH"' >>~/.profile
sudo ln -sf /opt/cosmo/tool/scripts/cosmocc /opt/cosmos/bin/cosmocc
sudo ln -sf /opt/cosmo/tool/scripts/cosmoc++ /opt/cosmos/bin/cosmoc++
```

You've now successfully installed your very own cosmos. Now let's build
an example program, which demonstrates the crash reporting feature:

```c
// hello.c
#include <stdio.h>
#include <cosmo.h>

int main() {
  ShowCrashReports();
  printf("hello world\n");
  __builtin_trap();
}
```

To compile the program, you can run the `cosmocc` command. It's
important to give it an output path that ends with `.com` so the output
format will be Actually Portable Executable. When this happens, a
concomitant debug binary is created automatically too.

```sh
cosmocc -o hello.com hello.c
./hello.com
./hello.com.dbg
```

You can use the `cosmocc` toolchain to build conventional open source
projects which use autotools. This strategy normally works:

```sh
export CC=cosmocc
export CXX=cosmoc++
./configure --prefix=/opt/cosmos
make -j
make install
```

The Cosmopolitan Libc runtime links some heavyweight troubleshooting
features by default, which are very useful for developers and admins.
Here's how you can log system calls:

```sh
./hello.com --strace
```

Here's how you can get a much more verbose log of function calls:

```sh
./hello.com --ftrace
```

If you want to cut out the bloat and instead make your executables as
tiny as possible, then the monorepo supports numerous build modes. You
can select one of the predefined ones by looking at
[build/config.mk](build/config.mk). One of the most popular modes is
`MODE=tiny`. It can be used with the `cosmocc` toolchain as follows:

```sh
cd /opt/cosmo
make -j8 MODE=tiny toolchain
```

Now that we have our toolchain, let's write a program that links less
surface area than the program above. The executable that this program
produces will run on platforms like Linux, Windows, MacOS, etc., even
though it's directly using POSIX APIs, which Cosmopolitan polyfills.

```c
// hello2.c
#include <unistd.h>
int main() {
  write(1, "hello world\n", 12);
}
```

Now let's compile our tiny actually portable executable, which should be
on the order of 20kb in size.

```sh
export MODE=tiny
cosmocc -Os -o hello2.com hello2.c
./hello2.com
```

Let's say you only care about Linux and would rather have simpler tinier
binaries, similar to what Musl Libc would produce. In that case, try
using the `MODE=tinylinux` build mode, which can produce binaries more
on the order of 4kb.

```sh
export MODE=tinylinux
(cd /opt/cosmo; make -j8 toolchain)
cosmocc -Os -o hello2.com hello2.c
./hello2.com  # <-- actually an ELF executable
```

## ARM

Cosmo supports cross-compiling binaries for machines with ARM
microprocessors. There are two options available for doing this.

The first option is to embed the [blink virtual
machine](https://github.com/jart/blink) by adding the following to the
top of your main.c file:

```c
__static_yoink("blink_linux_aarch64");  // for raspberry pi
__static_yoink("blink_xnu_aarch64");    // is apple silicon
```

The benefit is you'll have single file executables that'll run on both
x86_64 and arm64 platforms. The tradeoff is Blink's JIT is slower than
running natively, but tends to go fast enough, unless you're doing
scientific computing (e.g. running LLMs with
`o//third_party/ggml/llama.com`).

Therefore, the second option is to cross compile aarch64 executables,
by using build modes like the following:

```sh
make -j8 m=aarch64 o/aarch64/third_party/ggml/llama.com
make -j8 m=aarch64-tiny o/aarch64-tiny/third_party/ggml/llama.com
```

That'll produce ELF executables that run natively on two operating
systems: Linux Arm64 (e.g. Raspberry Pi) and MacOS Arm64 (i.e. Apple
Silicon), thus giving you full performance. The catch is you have to
compile these executables on an x86_64-linux machine. The second catch
is that MacOS needs a little bit of help understanding the ELF format.
To solve that, we provide a tiny APE loader you can use on M1 machines.

```sh
scp ape/ape-m1.c macintosh:
scp o/aarch64/third_party/ggml/llama.com macintosh:
ssh macintosh
xcode-install
cc -o ape ape-m1.c
sudo cp ape /usr/local/bin/ape
```

You can run your ELF AARCH64 executable on Apple Silicon as follows:

```sh
ape ./llama.com
```

## Source Builds

Cosmopolitan can be compiled from source on any Linux distro. First, you
need to download or clone the repository.

```sh
wget https://justine.lol/cosmopolitan/cosmopolitan.tar.gz
tar xf cosmopolitan.tar.gz  # see releases page
cd cosmopolitan
```

This will build the entire repository and run all the tests:

```sh
build/bootstrap/make.com
o//examples/hello.com
find o -name \*.com | xargs ls -rShal | less
```

If you get an error running make.com then it's probably because you have
WINE installed to `binfmt_misc`. You can fix that by installing the the
APE loader as an interpreter. It'll improve build performance too!

```sh
ape/apeinstall.sh
```

Since the Cosmopolitan repository is very large, you might only want to
build a particular thing. Cosmopolitan's build config does a good job at
having minimal deterministic builds. For example, if you wanted to build
only hello.com then you could do that as follows:

```sh
build/bootstrap/make.com o//examples/hello.com
```

Sometimes it's desirable to build a subset of targets, without having to
list out each individual one. You can do that by asking make to build a
directory name. For example, if you wanted to build only the targets and
subtargets of the chibicc package including its tests, you would say:

```sh
build/bootstrap/make.com o//third_party/chibicc
o//third_party/chibicc/chibicc.com --help
```

Cosmopolitan provides a variety of build modes. For example, if you want
really tiny binaries (as small as 12kb in size) then you'd say:

```sh
build/bootstrap/make.com m=tiny
```

Here's some other build modes you can try:

```sh
build/bootstrap/make.com m=dbg       # asan + ubsan + debug
build/bootstrap/make.com m=asan      # production memory safety
build/bootstrap/make.com m=opt       # -march=native optimizations
build/bootstrap/make.com m=rel       # traditional release binaries
build/bootstrap/make.com m=optlinux  # optimal linux-only performance
build/bootstrap/make.com m=fastbuild # build 28% faster w/o debugging
build/bootstrap/make.com m=tinylinux # tiniest linux-only 4kb binaries
```

For further details, see [//build/config.mk](build/config.mk).

## Cosmopolitan Amalgamation

Another way to use Cosmopolitan is via our amalgamated release, where
we've combined everything into a single static archive and a single
header file. If you're doing your development work on Linux or BSD then
you need just five files to get started. Here's what you do on Linux:

```sh
wget https://justine.lol/cosmopolitan/cosmopolitan-amalgamation-2.2.zip
unzip cosmopolitan-amalgamation-2.2.zip
printf 'main() { printf("hello world\\n"); }\n' >hello.c
gcc -g -Os -static -nostdlib -nostdinc -fno-pie -no-pie -mno-red-zone \
  -fno-omit-frame-pointer -pg -mnop-mcount -mno-tls-direct-seg-refs -gdwarf-4 \
  -o hello.com.dbg hello.c -fuse-ld=bfd -Wl,-T,ape.lds -Wl,--gc-sections \
  -Wl,-z,common-page-size=0x1000 -Wl,-z,max-page-size=0x1000 \
  -include cosmopolitan.h crt.o ape-no-modify-self.o cosmopolitan.a
objcopy -S -O binary hello.com.dbg hello.com
```

You now have a portable program.

```sh
./hello.com
bash -c './hello.com'  # zsh/fish workaround (we patched them in 2021)
```

If `./hello.com` executed on Linux throws an error about not finding an
interpreter, it should be fixed by running the following command (although
note that it may not survive a system restart):

```sh
sudo sh -c "echo ':APE:M::MZqFpD::/bin/sh:' >/proc/sys/fs/binfmt_misc/register"
```

If the same command produces puzzling errors on WSL or WINE when using
Redbean 2.x, they may be fixed by disabling binfmt_misc:

```sh
sudo sh -c 'echo -1 >/proc/sys/fs/binfmt_misc/status'
```

Since we used the `ape-no-modify-self.o` bootloader (rather than
`ape.o`) your executable will not modify itself when it's run. What
it'll instead do, is extract a 4kb program (the [APE loader](https://justine.lol/apeloader/))
to `${TMPDIR:-${HOME:-.}}` that maps your program into memory without
needing to copy it. The APE loader must be in an executable location
(e.g. not stored on a `noexec` mount) for it to run. See below for
alternatives:

It's possible to install the APE loader systemwide as follows.

```sh
# System-Wide APE Install
# for Linux, Darwin, and BSDs
# 1. Copies APE Loader to /usr/bin/ape
# 2. Registers w/ binfmt_misc too if Linux
ape/apeinstall.sh

# System-Wide APE Uninstall
# for Linux, Darwin, and BSDs
ape/apeuninstall.sh
```

It's also possible to convert APE binaries into the system-local format
by using the `--assimilate` flag. Please note that if binfmt_misc is in
play, you'll need to unregister it temporarily before doing this, since
the assimilate feature is part of the shell script header.

```sh
$ file hello.com
hello.com: DOS/MBR boot sector
./hello.com --assimilate
$ file hello.com
hello.com: ELF 64-bit LSB executable
```

Now that you're up and running with Cosmopolitan Libc and APE, here's
some of the most important troubleshooting tools APE offers that you
should know, in case you encounter any issues:

```sh
./hello.com --strace   # log system calls to stderr
./hello.com --ftrace   # log function calls to stderr
```

Do you love tiny binaries? If so, you may not be happy with Cosmo adding
heavyweight features like tracing to your binaries by default. In that
case, you may want to consider using our build system:

```sh
make m=tiny
```

Which will cause programs such as `hello.com` and `life.com` to shrink
from 60kb in size to about 16kb. There's also a prebuilt amalgamation
online <https://justine.lol/cosmopolitan/cosmopolitan-tiny.zip> hosted
on our download page <https://justine.lol/cosmopolitan/download.html>.

## GDB

Here's the recommended `~/.gdbinit` config:

```gdb
set host-charset UTF-8
set target-charset UTF-8
set target-wide-charset UTF-8
set osabi none
set complaints 0
set confirm off
set history save on
set history filename ~/.gdb_history
define asm
  layout asm
  layout reg
end
define src
  layout src
  layout reg
end
src
```

You normally run the `.com.dbg` file under gdb. If you need to debug the
`.com` file itself, then you can load the debug symbols independently as

```sh
gdb foo.com -ex 'add-symbol-file foo.com.dbg 0x401000'
```

## Alternative Development Environments

### MacOS

If you're developing on MacOS you can install the GNU compiler
collection for x86_64-elf via homebrew:

```sh
brew install x86_64-elf-gcc
```

Then in the above scripts just replace `gcc` and `objcopy` with
`x86_64-elf-gcc` and `x86_64-elf-objcopy` to compile your APE binary.

### Windows

If you're developing on Windows then you need to download an
x86_64-pc-linux-gnu toolchain beforehand. See the [Compiling on
Windows](https://justine.lol/cosmopolitan/windows-compiling.html)
tutorial. It's needed because the ELF object format is what makes
universal binaries possible.

Cosmopolitan officially only builds on Linux. However, one highly
experimental (and currently broken) thing you could try, is building the
entire cosmo repository from source using the cross9 toolchain.

```sh
mkdir -p o/third_party
rm -rf o/third_party/gcc
wget https://justine.lol/linux-compiler-on-windows/cross9.zip
unzip cross9.zip
mv cross9 o/third_party/gcc
build/bootstrap/make.com
```

## Discord Chatroom

The Cosmopolitan development team collaborates on the Redbean Discord
server. You're welcome to join us! <https://discord.gg/FwAVVu7eJ4>

## Support Vector

| Platform        | Min Version | Circa |
| :---            | ---:        | ---:  |
| AMD             | K8 Venus    | 2005  |
| Intel           | Core        | 2006  |
| Linux           | 2.6.18      | 2007  |
| Windows         | 8 [1]       | 2012  |
| Mac OS X        | 15.6        | 2018  |
| OpenBSD         | 7           | 2021  |
| FreeBSD         | 13          | 2020  |
| NetBSD          | 9.2         | 2021  |

[1] See our [vista branch](https://github.com/jart/cosmopolitan/tree/vista)
    for a community supported version of Cosmopolitan that works on Windows
    Vista and Windows 7.

## Special Thanks

Funding for this project is crowdsourced using
[GitHub Sponsors](https://github.com/sponsors/jart) and
[Patreon](https://www.patreon.com/jart). Your support is what makes this
project possible. Thank you! We'd also like to give special thanks to
the following groups and individuals:

- [Joe Drumgoole](https://github.com/jdrumgoole)
- [Rob Figueiredo](https://github.com/robfig)
- [Wasmer](https://wasmer.io/)

For publicly sponsoring our work at the highest tier.
