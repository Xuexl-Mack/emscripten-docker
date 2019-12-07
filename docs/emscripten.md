# Docker: emscripten
Based on: **[trzeci/emscripten-slim](https://hub.docker.com/r/trzeci/emscripten-slim)**

[![Docker Pulls](https://img.shields.io/docker/pulls/trzeci/emscripten.svg?style=flat-square)](https://store.docker.com/community/images/trzeci/emscripten/) [![Size](https://images.microbadger.com/badges/image/trzeci/emscripten.svg)](https://microbadger.com/images/trzeci/emscripten/)

A complete container that is required to compile C++ code with [Emscripten](http://emscripten.org). The goal was to provide a container that includes the most popular development packages and it's also easy to extend.
Each tag was build from [Dockerfile](https://github.com/trzecieu/emscripten-docker/blob/master/docker/trzeci/emscripten/Dockerfile)

## Packages

### Manually installed:

|packages|version|
|---|---|
|`cmake`|**3.14.3**|

### System packages:

<!-- installed_packages -->

## Tag schema
|tag|description|
|--|--|
|`latest`|The default version (aka `latest`) points at [the latest tag release](https://github.com/emscripten-core/emscripten/releases) by Emscripten.|
|`sdk-tag-{VERSION}-64bit`| Tag release:<br>- **VERSION**: One of the official [Emscripten tag](https://github.com/emscripten-core/emscripten/tags) released since 1.34.1. i.e. `1.38.25`|
|`sdk-{BRANCH}-64bit`|Branch release:<br>- **BRANCH**: one of  `["incoming", "master"]`|

## Usage
From start every Emscripten command is available. For the instance: `emcc`, `em++`, `emmake`, `emar`, `node`, `asm2wasm` etc.

To compile a single file:
```bash
docker run --rm -v `pwd`:`pwd` trzeci/emscripten emcc helloworld.cpp -o helloworld.js
```

More elaborated Hello World:
```bash
# create helloworld.cpp
cat << EOF > helloworld.cpp
#include <iostream>
int main() {
  std::cout << "Hello World!" << std::endl;
  return 0;
}
EOF

# compile with docker image
docker run \
  --rm \
  -v $(pwd):$(pwd) \
  -u $(id -u):$(id -g) \
  trzeci/emscripten \
  emcc helloworld.cpp -o helloworld.js

# execute on host machine
node helloworld.js
```

Teardown of compilation command:

|part|description|
|---|---|
|`docker run`| A standard command to run a command in a container|
|`--rm`|remove a container after execution (optimization)|
|`-v $(pwd):$(pwd)`|Mounting current folder from the host system into mirrored path on the container<br>TIP: This helps to investigate possible problem as we preserve exactly the same paths like in host. In such case modern editors (like Sublime, Atom, VS Code) let us to CTRL+Click on a problematic file |
|`-u $(id -u):$(id -g)`|(1.37.23+) Run a container as a non-root user with the same UID and GID as local user. Hence all files produced by this are accessible to non-root users|
|`trzeci/emscripten`|Get the latest tag of this container|
|`emcc helloworld.cpp -o helloworld.js`|Execute `emcc` command with following arguments inside container, effectively compile our source code|

## Extending this image
If you would like to extend this image you have two choices:
### Extend with keeping base image
An example how to derive from base image and keep linux base container you can find here : [trzeci/emscripten/Dockerfile](https://github.com/trzecieu/emscripten-docker/blob/master/docker/trzeci/emscripten/Dockerfile).

All what you need is:
```Dockerfile
FROM trzeci/emscripten:sdk-tag-1.38.25-64bit
# A good practice is to don't use `latest` - even if some effort is made to make sure that `latest` version is stable

RUN ...
```
This way you inherit all settings of environment that are coming with this image and entrypoiont.

### Extend with changing base image
An example how to derive from base image and switch linux base container you can find here - [trzeci/emscripten-ubuntu/Dockerfile](https://github.com/trzecieu/emscripten-docker/blob/master/docker/trzeci/emscripten-ubuntu/Dockerfile).

An example that uses Docker multi-stage build:
```dockerfile
FROM trzeci/emscripten:sdk-tag-1.38.25-64bit as emscripten_base
# Target base image
FROM fedora
# Copy pre-compiled content of Emscripten SDK to target iamge
COPY --from=emscripten_base /emsdk_portable /emsdk_portable

# install required tools to run Emscripten SDK
RUN dnf install -y python python-pip ca-certificates

# Use entrypoint that comes with Emscripten. This is a way to to activate Emscripten Tools.
ENTRYPOINT ["/emsdk_portable/entrypoint"]
```
That's all! All you need is to copy content from `/emsdk_portable` to the same folder in your image.
Then it's important to use `/emsdk_portable/entrypoint` as it contains some nice fixes for non-root file access rights.
Alternatively you can also call:
```bash
. /emsdk_portable/emsdk_set_env.sh
```
in your entrypoint - it will work just fine!*

<sub>* In vast majority of cases, if it doesn't due `$PATH` override please get familiar with `Entrypoint-less` case</sub>

#### Entrypoint
Important step is to activate Emscripten SDK - so that tools are available for usage.
Most traditional way is to use entrypoint and this image provides one:
```dockerfile
ENTRYPOINT ["/emsdk_portable/entrypoint"]
```

#### Entypoint-less
In case when you can't override entrypoint or you have to have Emscripten Ready as a docker stage step, or activation script messes up with your system `$PATH` -  you might use an alternative method - let's say static activation.
For that instead of adding `ENTRYPOINT` you need to add:
```dockerfile
# Setup Emscripten Environment variables
ENV EMSDK /emsdk_portable
ENV EMSCRIPTEN=${EMSDK}/emscripten/sdk

ENV EM_DATA ${EMSDK}/.data
ENV EM_CONFIG ${EMSDK}/.emscripten
ENV EM_CACHE ${EM_DATA}/cache
ENV EM_PORTS ${EM_DATA}/ports

# Expose tools to system PATH
ENV PATH="${EMSDK}:${EMSDK}/emscripten/sdk:${EMSDK}/llvm/clang/bin:${EMSDK}/node/current/bin:${EMSDK}/binaryen/bin:${PATH}"
```

Basically you can use whatever base system of choice and copy content of `/emsdk_portable` from either `emscripten` or `emscripten-slim` and start use it.

## Build this image manually
### Using script
0. Pull the latest https://github.com/trzecieu/emscripten-docker
0. Use a helper command i.e `./build compile sdk-tag-1.38.31-64bit` which will build all variants of SDK ([emscripten-slim](https://hub.docker.com/r/trzeci/emscripten-slim/), [emscripten](https://hub.docker.com/r/trzeci/emscripten/), [emscripten-ubuntu](https://hub.docker.com/r/trzeci/emscripten-ubuntu/)) and perform tests on those.

### Manual building
0. Pull the latest https://github.com/trzecieu/emscripten-docker
0. Navigate to wanted flavor to compile inside `./docker` path, let's say `./docker/trzeci/emscripten`.
0. Execute singe docker command and provide wanted version to compile as an argument:
```bash
docker build -t my_little_emscripten_image --build-arg EMSCRIPTEN_SDK=sdk-tag-1.38.31-64bit .
```

## Reproduce old builds
As EMSDK and Emscripten and even this Docker Image are under an extensive development some scripts might change in meantime.

### Check used EMSDK version
To be extra accurate, you can check which version of [EMSDK](https://github.com/juj/emsdk) was used in a particular image. For older images you can check [a file](https://github.com/trzecieu/emscripten-docker/blob/master/emscripten_to_emsdk_map.md) otherwise for images 1.38.9+ execute a command `docker run --rm -it trzeci/emscripten:sdk-tag-1.38.9-64bit bash -c "git -C /emsdk_portable rev-parse HEAD"`
Found changeset should be used as `docker build` argument - `EMSDK_CHANGESET`

### Check used Dockerimage
#### Since 1.38.33
Dockerfiles are added to the image under `/emsdk_portable/dockerfiles` folder to get a copy of those please execute:
```bash
# following command will tar dockerfiles form image and export files under host's filesystem in 'dockerfiles' folder.
docker run --rm trzeci/emscripten-slim:sdk-tag-1.38.33-64bit  tar -c -C /emsdk_portable dockerfiles | tar xv
```
Then it's possible to have a full inspection to exact the same Docker file that were used for image building.

#### Older images:

**A**: Execute command line:
```bash
# example for 1.38.25
tag=sdk-tag-1.38.25-64bit
docker pull trzeci/emscripten:${tag} > /dev/null
docker inspect  --format '{{ index .Config.Labels "org.label-schema.vcs-ref"}}' trzeci/emscripten:${tag}

```
**B**: Use microbadger portal:
* Navigate to https://microbadger.com/images/trzeci/emscripten/
* Select wanted version
* Inspect `org.label-schema.vcs-ref` property that should hold changeset of `emscripten-docker` repository that was used during build process.

----

## Support
* **GitHub / Issue tracker (for docker image)**: https://github.com/trzecieu/emscripten-docker
* **GitHub / Issue tracker (for Emscripten)**: https://github.com/emscripten-core/emscripten
* **Docker Hub: emscripten**: https://hub.docker.com/r/trzeci/emscripten/
* **Docker Hub: emscripten-slim**: https://hub.docker.com/r/trzeci/emscripten-slim/

## History
<sub>(Please note that following history refers only to the history of this Docker Image and how it was build / what includes. For release notes of emscripten, please follow https://emscripten.org)</sub>

* **1.38.34**: Cumulative change
[#45](https://github.com/trzecieu/emscripten-docker/issues/45)[#46](https://github.com/trzecieu/emscripten-docker/issues/46) Adopt image to new Emscripten SDK
  * Starting from 1.38.33 Emscripten has switched project build system to waterfall, which has introduced some complication to building the image. The main problem was that no longer was possible to compile Emscripten from sources, with making sure that we have exactly the same components like Emscripten was released. For the instance WebAssembly/binaryen repo wasn't tagged with Emscripten version anymore.

[#44](https://github.com/trzecieu/emscripten-docker/issues/44) Significant refactoring of base image emscripten-slim. Please visit issue, to get extended context and motivation.
  * Improvements:
    * `/emsdk_portable` is fully moveable folder that can be used as a `COPY --from` source of multi stage build
    * `/emsdk_portable/dockerfiles` contains Dockerfile sources used to compile a particular image - so that it should be fairly easy to replicate and inspect content of images
    * `emsdk` should be fully functional tool now, so that can be used for upgrading bundled emscripten SDK or to install extra tools
    * Even further size optimization for slim image (From ~190MB to ~160MB) by:
      * stripping out symbols from node.js and emscripten-clang tools
      * change base image to `debiang:stretch-slim`
      * remove non-essential files from emscripten folder
    * Add enhanced entrypoint which also fixes files permission on created files
  * Breaking Changes:
    * Image no longer creates system symbolic links to every known tool.
      * Instead it adds folders to system `$PATH`
    * Image no longer preserves folder structure between versions, some tool might be placed in different location between versions
      * Instead it creates symbolic links in fixed locations that match old structure
    * `nodejs` is no longer symlinked (`node` should be used instead)
* **1.38.33**: Version ignored due problem with dependencies
* **1.38.30**: [#40](https://github.com/trzecieu/emscripten-docker/issues/40) Fixed image compilation problem caused by JRE backport package
* **1.38.22**: [#35](https://github.com/trzecieu/emscripten-docker/issues/35) upgrade to `cmake` 3.12.2
* **1.38.17**: Version ignored due problems with [Emscripten]
* **1.38.9**: `/emsdk_portable` will be preserved as a git repos (with valid version of changeset)
* **1.38.7**: Version removed due problems with [emsdk](https://github.com/juj/emsdk/pull/156)
* **1.37.29**: upgrade to `cmake` 3.7.2
* **1.37.23**: Added `curl`, `zip`, `unzip`, upgrade to openjdk-jre-8
* **1.37.21**: Fixed missing `ctest` command
* **1.37.21**: image includes `ssh` and cache of libc libcxx is fixed.
* **1.37.19**: image doesn't use entrypoint from the base image.
* **1.37.18**: it contains `perl` and `git` package
* **1.37.16**: images are compiled from singe [Dockerfile](https://github.com/trzecieu/emscripten-docker/blob/master/docker/trzeci/emscripten/Dockerfile).
* **1.37.10**: images are bundled with `java`
* **1.36.7**: images are bundled with `make` and `nodejs`
* **1.36.7**: images are bundled with `cmake` 3.6.3, images are build from generated [Dockerfiles](https://github.com/trzecieu/emscripten-docker/tree/f738f061c8068ec24124c37286eafec01d54a6ef/configs)
* **1.35.0**: images base on Debian
* **1.34.X**: images base on Ubuntu:15.10

## License
[![MIT](https://img.shields.io/github/license/trzecieu/emscripten-docker.svg?style=flat-square)](https://github.com/trzecieu/emscripten-docker/blob/master/LICENSE)
