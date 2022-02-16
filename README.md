# Chromium Development / Porting for RISC-V

## Setup cross-compile toolchain

### LLVM / Clang

1. Build a cross-compile toolchain for llvm and clang.
```
$ sudo apt-get -y install binutils build-essential libtool texinfo gzip zip unzip patchutils curl git make cmake ninja-build automake bison flex gperf grep sed gawk python bc zlib1g-dev libexpat1-dev libmpc-dev libglib2.0-dev libfdt-dev libpixman-1-dev 

$ mkdir ~/riscv
$ cd ~/riscv
$ mkdir _install
$ export PATH=`pwd`/_install/bin:$PATH
$ hash -r

# gcc, binutils, newlib, qemu
$ git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
$ pushd riscv-gnu-toolchain
$ ./configure --prefix=`pwd`/../_install --enable-multilib
$ make -j`nproc` linux
$ make -j`nproc` build-qemu
$ popd

# llvm
$ git clone https://github.com/llvm/llvm-project.git riscv-llvm
$ pushd riscv-llvm
$ ln -s ../../clang llvm/tools || true
$ mkdir _build
$ cd _build
$ cmake -G Ninja -DCMAKE_BUILD_TYPE="Release" \
  -DBUILD_SHARED_LIBS=True -DLLVM_USE_SPLIT_DWARF=True \
  -DCMAKE_INSTALL_PREFIX="../../_install" \
  -DLLVM_OPTIMIZED_TABLEGEN=True -DLLVM_BUILD_TESTS=False \
  -DDEFAULT_SYSROOT="../../_install/riscv64-unknown-linux-gnu" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="riscv64-unknown-linux-gnu" \
  -DLLVM_TARGETS_TO_BUILD="RISCV" \
  ../llvm
$ cmake --build . --target install
$ popd
```


## Chromium

1. Follow the Chromium official build instruction for Linux host to get the source code.
https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/linux/build_instructions.md

2. Install additional build dependencies (by using the script available within the source code.)

3. Edit the .gclient file in //chromium top level. This is to disable gclient from syncing NaCL code base.
```
solutions = [
  {
    "name": "src",
    "url": "https://chromium.googlesource.com/chromium/src.git",
    "managed": False,
    "custom_deps": {},
    "custom_vars": { "checkout_nacl": False },
  },
]
```

4. Run the Chromium-specific hooks, this will download additional toolchain / binaries that we might need for the build later.
```
$ cd ~/chromium/src
$ gclient runhooks
```

5. Set up the build configurations in args.gn
```
$ mkdir -p out/riscv64
$ vim out/riscv64/args.gn
target_os="linux"
target_cpu="riscv64"

is_component_build = true
is_debug=false
symbol_level=0
v8_symbol_level=0
blink_symbol_level=0

# Disable broken features
use_gnome_keyring=false
enable_nacl=false
treat_warnings_as_errors=false
fatal_linker_warnings=false
enable_dav1d_decoder = false
enable_reading_list=false
enable_vr=false
enable_swiftshader=false
supports_subzero=false
enable_libaom=false
enable_openscreen=false
enable_jxl_decoder=false

# For clang
is_clang = true
clang_base_path = getenv("HOME") + "/riscv/_install"
clang_use_chrome_plugins = false
use_gold = false
use_lld = true
```

6. Apply the patches.

7. Setup ffmpeg.
```
$ cd third_party/ffmpeg
$ ./chromium/scripts/build_ffmpeg.py linux riscv64
$ ./chromium/scripts/generate_gn.py
$ ./chromium/scripts/copy_config.sh
```

8. Run the build and start resolving build issues.
```
$ autoninja -C out/riscv64 chrome
```

## Debugging

1. Generate dependency tree from GN
```
$ gn desc out/risc64 chrome --tree
```
