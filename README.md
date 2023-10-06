
# Build ONNXRUNTIME (wasm shared lib)

## Issues

- https://github.com/microsoft/onnxruntime/issues/17780
- https://github.com/microsoft/onnxruntime/issues/17781

## Build

```sh
git clone --depth 1 --branch v1.14.1 https://github.com/microsoft/onnxruntime.git modules/onnxruntime
docker run -it -v $(pwd)/modules/onnxruntime:/onnxruntime -w /onnxruntime --platform linux/amd64 debian:12.1
```

In docker shell:

```sh
uname -m # must say x86_64 !
apt-get update
apt-get install -y git python3 build-essential cmake autoconf autogen libtool pkg-config ragel
git config --global pull.rebase false
git config --global --add safe.directory /onnxruntime

rm -rf ./cmake/external/emsdk
git clone https://github.com/emscripten-core/emsdk.git ./cmake/external/emsdk
./cmake/external/emsdk/emsdk install 3.1.46
./cmake/external/emsdk/emsdk activate 3.1.46

sed -i 's/run_subprocess(\[emsdk_file/#\0/g' ./tools/ci_build/build.py

./build.sh --config Release --build_wasm_static_lib --skip_tests --enable_wasm_debug_info --disable_wasm_exception_catching --disable_rtti --emsdk_version 3.1.46
```

...builds ~2h and produces artifact `build/Linux/Release/libonnxruntume_webassembly.a`

# Build Piper (native) not required step

```sh
git clone --depth 1 https://github.com/rhasspy/piper.git
docker run -it -v $(pwd)/piper:/piper -w /piper debian:11.3 # docker c9f6ab693611
apt-get update
apt-get install -y git python3 build-essential cmake autoconf autogen libtool pkg-config ragel
make clean
cmake -Bbuild -DCMAKE_INSTALL_PREFIX=install
cmake --build build --config Release

echo "hello world" | ./build/piper --model ./model/en_US-lessac-medium.onnx --output_file welcome.wav --espeak_data ./build/p/src/piper_phonemize_external-build/ei/share/espeak-ng-data/

./build/piper --input '[{"text":"My json text", "output_file":"o1.wav"},{"text":"second line", "output_file":"o2.wav"}]' --model ./model/en_US-lessac-medium.onnx --output_file welcome.wav --espeak_data ./build/p/src/piper_phonemize_external-build/ei/share/espeak-ng-data/
```

# Build Piper (wasm)

## Build

```shell
git clone --depth 1 https://github.com/rhasspy/espeak-ng.git modules/espeak-ng
git clone --depth 1 -b wide.video https://github.com/wide-video/piper.git modules/piper
git clone https://github.com/emscripten-core/emsdk.git modules/emsdk
docker run -it -v $(pwd):/piper-wasm -w /piper-wasm debian:11.3 # docker 4c7a815a878b
```

In docker shell:

```sh
apt-get update
apt-get install -y git python3 build-essential cmake autoconf autogen libtool pkg-config ragel
git config --global pull.rebase false

cd /piper-wasm/modules/espeak-ng
./autogen.sh
./configure
make

cd /piper-wasm/modules/emsdk
./emsdk install 3.1.46
./emsdk activate 3.1.46
source ./emsdk_env.sh
TOOLCHAIN_FILE=$EMSDK/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake
sed -i -E 's/int\s+(iswalnum|iswalpha|iswblank|iswcntrl|iswgraph|iswlower|iswprint|iswpunct|iswspace|iswupper|iswxdigit)\(wint_t\)/\/\/\0/g' ./upstream/emscripten/cache/sysroot/include/wchar.h

cd /piper-wasm/modules/piper
emmake make clean

emmake cmake -Bbuild -DCMAKE_INSTALL_PREFIX=install -DCMAKE_TOOLCHAIN_FILE=$TOOLCHAIN_FILE -DBUILD_TESTING=OFF -G "Unix Makefiles"
emmake cmake --build build --config Release
# build fails but continue:
sed -i 's+/piper-wasm/modules/piper/build/p/src/piper_phonemize_external-build/e/src/espeak_ng_external-build/src/espeak-ng+/piper-wasm/modules/espeak-ng/src/espeak-ng+g' /piper-wasm/modules/piper/build/p/src/piper_phonemize_external-build/e/src/espeak_ng_external-build/CMakeFiles/data.dir/build.make

sed -i 's/using namespace std/\/\/\0/g' /piper-wasm/modules/piper/build/p/src/piper_phonemize_external-build/e/src/espeak_ng_external/src/speechPlayer/src/speechWaveGenerator.cpp

rm /piper-wasm/modules/piper/build/p/src/piper_phonemize_external/lib/onnxruntime-linux-aarch64-1.14.1/lib/libonnxruntime.so
cp /piper-wasm/libonnxruntime_webassembly_v1.14.1_emsdk_3.1.46 /piper-wasm/modules/piper/build/p/src/piper_phonemize_external/lib/onnxruntime-linux-aarch64-1.14.1/lib/libonnxruntime.so

# in /Users/jozefchutka/dev/wide.video/piper-wasm/modules/piper/build/p/src/piper_phonemize_external-build/Makefile
# comment out blocks:
# piper_phonemize_exe/fast:
# piper_phonemize_exe/preinstall:

# in /Users/jozefchutka/dev/wide.video/piper-wasm/modules/piper/build/p/src/piper_phonemize_external-build/CMakeFiles/Makefile2
# comment out:
# all: CMakeFiles/piper_phonemize_exe.dir/all
# preinstall: CMakeFiles/piper_phonemize_exe.dir/preinstall

# in /Users/jozefchutka/dev/wide.video/piper-wasm/modules/piper/build/p/src/piper_phonemize_external-build/cmake_install.cmake
# comment out block:
# if("x${CMAKE_INSTALL_COMPONENT}x" STREQUAL "xUnspecifiedx" OR NOT CMAKE_INSTALL_COMPONENT)

# to update main.cpp.o
emmake cmake --build build --config Release

cd /piper-wasm/modules/piper/build

# see original generated flags 
# /usr/bin/cmake -E cmake_link_script CMakeFiles/piper.dir/link.txt --verbose=1

/piper-wasm/modules/emsdk/upstream/emscripten/em++  -Wall -Wextra -Wl,-rpath,'$ORIGIN' -Wall -Wextra -Wl,-rpath,'$ORIGIN' @CMakeFiles/piper.dir/objects1.rsp -o piper.js @CMakeFiles/piper.dir/linklibs.rsp -lworkerfs.js -O3 -s INVOKE_RUN=0 -s MODULARIZE=1 -s EXPORT_NAME="createPiper" -s EXPORTED_FUNCTIONS="[_main]" -s EXPORTED_RUNTIME_METHODS="[callMain, FS, WORKERFS]" -s WASM_BIGINT=1 -s INITIAL_MEMORY=64mb -s ALLOW_MEMORY_GROWTH=1 -s MAXIMUM_MEMORY=4gb -s ENVIRONMENT=worker -s STACK_SIZE=5MB -s DEFAULT_PTHREAD_STACK_SIZE=2MB --preload-file  p/src/piper_phonemize_external-build/ei/share/espeak-ng-data@/espeak-ng-data
```

# Docker Reattach

Reattach stdin for exited container:

```shell
docker ps -q -l              # find container ID (or discover via Docker desktop)
docker start e24313a5d869    # restart in the background
docker attach e24313a5d869   # reattach the terminal & stdin
```