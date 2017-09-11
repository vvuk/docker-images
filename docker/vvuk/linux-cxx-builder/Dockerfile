FROM ubuntu:latest

# ------------------------------------------------------------------------------

ARG EMSCRIPTEN_SDK=sdk-tag-1.37.21-64bit
ARG BUILD_DATE
ARG VCS_REF
ARG CMAKE=https://cmake.org/files/v3.9/cmake-3.9.2-Linux-x86_64.sh
ARG NODEJS=https://nodejs.org/dist/v6.11.3/node-v6.11.3-linux-x64.tar.xz

# ------------------------------------------------------------------------------

ENV EMSDK /emsdk_portable
ENV EM_CONFIG ${EMSDK}/.config
ENV EM_CACHE ${EMSDK}/.cache
ENV EM_PORTS ${EMSDK}/.ports
ENV EMSCRIPTEN ${EMSDK}/sdk

# ------------------------------------------------------------------------------

LABEL maintainer="vladimir@pobox.com" \
      org.label-schema.name="linux-cxx-builder" \
      org.label-schema.description="This image includes gcc, clang, and Emscripten." \
      org.label-schema.url="https://github.com/vvuk/docker-images" \
      org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vcs-url="https://github.com/vvuk/docker-images" \
      org.label-schema.docker.dockerfile="/docker/vvuk/linux-cxx-builder/Dockerfile"

# ------------------------------------------------------------------------------

RUN echo "\n## Start building" \
    \
&&  echo "\n## Update and install packages" \
&&  apt-get -qq -y update && apt-get -qq install -y --no-install-recommends \
        wget \
        git-core \
        ca-certificates \
        build-essential \
        python \
        make \
    \
&&  wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key -q | apt-key add - \
&&  echo "deb http://apt.llvm.org/xenial/ llvm-toolchain-xenial-4.0 main" > /etc/apt/sources.list.d/llvm.list \
&&  apt-get -qq -y update && apt-get -qq install -y --no-install-recommends \
        clang-4.0 lldb-4.0 \
    \
&&  echo "\n## Installing CMake" \
&&  wget -O cmake.sh $CMAKE -q \
&&  mkdir /opt/cmake \
&&  printf "y\nn\n" | sh cmake.sh --prefix=/opt/cmake > /dev/null \
&&  ln -s /opt/cmake/bin/cmake /usr/local/bin/cmake \
    \
&&  echo "\n## Installing nodejs" \
&&  wget -O nodejs.tar.xz $NODEJS -q \
&&  tar -x -J -C /usr/local --strip-components 1 -f nodejs.tar.xz \
&&  ln -sf /usr/local/bin/node /usr/local/bin/nodejs \
    \
&&  echo "\n## Fetching emsdk" \
&&  git clone --depth 1 https://github.com/juj/emsdk.git $EMSDK && cd $EMSDK \
&&  echo $EMSCRIPTEN_SDK | sed  's/.\+-\([0-9]\+\.[0-9]\+\.[0-9]\+\).\+/\1/g' > emscripten-tags.txt \
    \
&&  echo "\n## Compile Emscripten" \
&&  ./emsdk install --build=MinSizeRel $EMSCRIPTEN_SDK \
    \
&&  echo "\n## Generate standard configuration" \
&&  ./emsdk activate $EMSCRIPTEN_SDK \
&&  ./emsdk construct_env > /dev/null \
    \
&&  echo "\n## Re-Configure Emscripten" \
&&  printf "import os\n"                                                >> $EM_CONFIG \
&&  printf "LLVM_ROOT='$EMSDK/llvm/clang/bin'\n"                        >> $EM_CONFIG \
&&  printf "NODE_JS='/usr/local/bin/node'\n"                            >> $EM_CONFIG \
&&  printf "EMSCRIPTEN_ROOT='$EMSCRIPTEN'\n"                            >> $EM_CONFIG \
&&  printf "EMSCRIPTEN_NATIVE_OPTIMIZER='$EMSDK/optimizer/optimizer'\n" >> $EM_CONFIG \
&&  printf "SPIDERMONKEY_ENGINE=''\n"                                   >> $EM_CONFIG \
&&  printf "V8_ENGINE=''\n"                                             >> $EM_CONFIG \
&&  printf "TEMP_DIR='/tmp'\n"                                          >> $EM_CONFIG \
&&  printf "COMPILER_ENGINE=NODE_JS\n"                                  >> $EM_CONFIG \
&&  printf "JS_ENGINES=[NODE_JS]\n"                                     >> $EM_CONFIG \
&&  cat $EM_CONFIG \
    \
&&  echo "\n## Move folders according to new structure" \
&&  cat ~/.emscripten | sed -rn "s/LLVM_ROOT='(.*)'/\1/p" | xargs dirname | xargs -i mv {} {}/../clang \
&&  cat ~/.emscripten | sed -rn "s/LLVM_ROOT='(.*)'/\1/p" | xargs dirname | xargs dirname | xargs -i mv {} $EMSDK/llvm \
&&  cat ~/.emscripten | sed -rn "s/EMSCRIPTEN_ROOT='(.*)'/\1/p" | xargs -i mv {} $EMSCRIPTEN \
&&  cat ~/.emscripten | sed -rn "s/EMSCRIPTEN_NATIVE_OPTIMIZER='(.*)'/\1/p" | xargs dirname | xargs -i mv {} $EMSDK/optimizer \
&&  rm -fr $EMSDK/emscripten \
&&  rm -fr $EMSDK/clang \
&&  rm -fr $EMSDK/emsdk_set_env.sh \
&&  rm -fr ~/.emscripten* \ 
    \
&&  echo "\n## Compile Binaryen" \
&&  $EMSCRIPTEN/embuilder.py build binaryen \
&&  mkdir $EMSDK/binaryen \
&&  mv $EM_PORTS/binaryen/binaryen-*/bin $EMSDK/binaryen/bin \
&&  printf "BINARYEN_ROOT='$EMSDK/binaryen'\n"  >> $EM_CONFIG \
    \
&&  echo "\n## Create links" \
&&  for f in $EMSDK/llvm/clang/bin $EMSDK/binaryen/bin $EMSDK/optimizer $EMSDK $EMSCRIPTEN; do \
        find $f -maxdepth 1 -type f -executable | xargs -i bash -c 'ln -sf {} /usr/local/bin/$(basename {})'; \
    done \
    \
    \
&&  echo "\n## Cleaning up Emscripten Compilation" \
&&  mv llvm/src/emscripten-version.txt emscripten-version.txt \
&&  rm -rf llvm/src \
&&  rm -rf llvm/*/lib \
&&  rm -rf llvm/*/docs \
&&  rm -rf llvm/*/tools \
&&  mkdir -p llvm/src \
&&  mv emscripten-version.txt llvm/src/emscripten-version.txt \
    \
&&  find . -name "CMakeFiles" -type d -prune -exec rm -fr {} \; \
&&  find . -name "CMakeCache.txt" -exec rm {} \; \
&&  find . -name "*.o" -exec rm {} \; \
&&  find . -name "*.a" -exec rm {} \; \
&&  find . -name "*.inc*" -exec rm {} \; \
&&  find . -name "*.gen.tmp" -exec rm {} \; \
&&  rm -fr .git bin node/*_32bit \
&&  rm -rf sdk/.git \
&&  rm -rf sdk/tests \
&&  rm -rf sdk/site \
&&  rm -rf sdk/docks \
&&  rm -rf sdk/media \
&&  rm -rf zips \
    \
&&  rm -fr /opt/cmake /usr/local/bin/cmake /cmake* \
    \
&&  echo "\n## Internal testing" \
&&  cd / \
&&  emcc --version \
&&  nodejs --version \
&&  node --version \
&&  python --version \
&&  which asm2wasm \
&&  which llvm-ar \
&&  which emsdk \
&&  which emcc \
&&  which em++ \
    \
&&  emcc --clear-cache --clear-ports \
    \
&&  echo "\n## Compile sample code" \
&&  mkdir -p /tmp/emscripten_test && cd /tmp/emscripten_test \
&&  printf '#include <iostream>\nint main(){std::cout<<"HELLO FROM DOCKER C++"<<std::endl;return 0;}' > test.cpp \
&&  em++ -O2 test.cpp -o test.js && nodejs test.js \
&&  em++ test.cpp -o test.js && nodejs test.js \
&&  em++ test.cpp -o test.js -s WASM=1 \
&&  cd / \
&&  rm -fr /tmp/emscripten_test \
    \
&&  echo "\n## Cleaning up" \
&&  apt-get -y clean \
&&  apt-get -y autoclean \
&&  apt-get -y autoremove \
    \
&&  echo "\n## Remove unused Debian files" \
&&  rm -rf /var/lib/apt/lists/* \
&&  rm -rf /var/cache/debconf/*-old \
&&  rm -rf /usr/share/doc/* \
&&  rm -rf /usr/share/man/?? \
&&  rm -rf /usr/share/man/??_* \
    \
&&  echo "\n## Create mock entrypoint (compatibility)" \
&&  printf '#!/bin/bash\nexec "$@"\n'                           > /entrypoint \
&&  chmod 777 /entrypoint \
    \
&&  echo "\n## Done"

WORKDIR /src
