# piper-wasm

WASM version of [piper](https://github.com/rhasspy/piper) powering [piper.wide.video](https://piper.wide.video/).

## Build WASM

Build takes ~5 minutes.

```sh
# Docker (optional)
docker run -it -v $(pwd):/wasm -w /wasm debian:11.3
apt-get update
apt-get install --yes --no-install-recommends build-essential cmake ca-certificates curl pkg-config git python3 autogen automake autoconf libtool

# Emscripten
git clone --depth 1 https://github.com/emscripten-core/emsdk.git /wasm/modules/emsdk
cd /wasm/modules/emsdk
./emsdk install 3.1.47
./emsdk activate 3.1.47
source ./emsdk_env.sh
TOOLCHAIN_FILE=$EMSDK/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake
sed -i -E 's/int\s+(iswalnum|iswalpha|iswblank|iswcntrl|iswgraph|iswlower|iswprint|iswpunct|iswspace|iswupper|iswxdigit)\(wint_t\)/\/\/\0/g' ./upstream/emscripten/cache/sysroot/include/wchar.h

# espeak-ng
git clone --depth 1 https://github.com/rhasspy/espeak-ng.git /wasm/modules/espeak-ng
cd /wasm/modules/espeak-ng
./autogen.sh
./configure
make

# piper-phonemize
git clone --depth 1 https://github.com/wide-video/piper-phonemize.git /wasm/modules/piper-phonemize
cd /wasm/modules/piper-phonemize
emmake cmake -Bbuild -DCMAKE_INSTALL_PREFIX=install -DCMAKE_TOOLCHAIN_FILE=$TOOLCHAIN_FILE -DBUILD_TESTING=OFF -G "Unix Makefiles" -DCMAKE_CXX_FLAGS="-O3 -s INVOKE_RUN=0 -s MODULARIZE=1 -s EXPORT_NAME='createPiperPhonemize' -s EXPORTED_FUNCTIONS='[_main]' -s EXPORTED_RUNTIME_METHODS='[callMain, FS]' --preload-file /wasm/modules/espeak-ng/espeak-ng-data@/espeak-ng-data"
emmake cmake --build build --config Release # fails on "Compile intonations / Permission denied", continue with next steps
sed -i 's+$(MAKE) $(MAKESILENT) -f CMakeFiles/data.dir/build.make CMakeFiles/data.dir/build+#\0+g' /wasm/modules/piper-phonemize/build/e/src/espeak_ng_external-build/CMakeFiles/Makefile2
sed -i 's/using namespace std/\/\/\0/g' /wasm/modules/piper-phonemize/build/e/src/espeak_ng_external/src/speechPlayer/src/speechWaveGenerator.cpp
emmake cmake --build build --config Release
```

Following artifacts are created in `/wasm/modules/piper-phonemize/build/`

- `piper_phonemize.data`
- `piper_phonemize.js`
- `piper_phonemize.wasm`

## Build piper-phonemize (native) not required step

```sh
git clone https://github.com/rhasspy/piper-phonemize.git
docker buildx build . -t piper-phonemize --output 'type=local,dest=dist'
# unpack ./dist/piper-phonemize_arm64.tar.gz into ./dist/piper_phonemize
# copy ./dist/piper_phonemize/lib/* to ./dist/piper_phonemize/bin
cd ./dist/piper_phonemize
docker run -it -v $(pwd):/piper-phonemize -w /piper-phonemize debian:11.3 
echo "Hello" | ./bin/piper_phonemize -l en-us --espeak-data ./share/espeak-ng-data
# {"phoneme_ids":[1,0,20,0,59,0,24,0,120,0,27,0,100,0,2],"phonemes":["h","ə","l","ˈ","o","ʊ"],"processed_text":"Hello","text":"Hello"}
```

## Docker Reattach

Reattach stdin for exited container:

```shell
docker ps -q -l              # find container ID (or discover via Docker desktop)
docker start e24313a5d869    # restart in the background
docker attach e24313a5d869   # reattach the terminal & stdin
```