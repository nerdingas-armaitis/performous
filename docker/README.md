This folder contains Dockerfiles which will install all dependencies needed to build [Performous](https://github.com/performous/performous/wiki/Building-and-installing-from-source).  

These containers are to be used as `base images` to provide higher-level builds and packages and to produce artifacts for downstream consumption. These containers **do not** provide a running version of `Performous` or contain the project source in any usable form.  

These containers are built automatically during our CI/CD workflow, and are used to support Linux Distros that are no available by default from Github Actions.  

## Building containers
The build is a pretty standard `docker build`, just make sure you explicitly call out a `Dockerfile` with `-f Dockerfile.<distro>` and supply the correct distro version as a `build-arg`:  
```sh
docker build -t performous-docker-build:ubuntu22.04 -f Dockerfile.ubuntu --build-arg OS_VERSION=22.04 .
```

Currently supported distros are:
- Ubuntu (20.04, 22.04)
- Fedora (34, 35, 36)
- Debian (10, 11)

## Running the containers
Once the `base-image` has been built, the container can be run interactively to build `Performous`:  
```sh
docker run -it performous-docker-build:ubuntu22.04
```  

From there, you can [follow the build instructions](https://github.com/performous/performous/wiki/Building-and-installing-from-source#downloading-and-installing-the-sources) to build performous.  


`build_performous.sh` is included in the containers for testing builds and creating OS packages.
```
Usage: ./build_performous.sh -a (build with all build systems)

Optional Arguments:
  -b <Git Branch>: Build the specified git branch, tag, or sha
  -p <Pull Request #>: Build the specified Github Pull Request number
  -g : Generate Packages
  -r <Repository URL>: Git repository to pull from
  -R : Perform a 'Release' Cmake Build (Default is 'RelWithDebInfo')
  -h : Show this help message
```

To build a pull request using just cmake:
```
docker run performous-docker-build:ubuntu22.04 ./build_performous.sh -c -p 626
```

One-Liner to generate packages and copy them to `/tmp` on the running system:
```
mkdir /tmp/performous-packages && docker run --rm --mount type=bind,source=/tmp/performous-packages,target=/performous/packages performous-docker-build:ubuntu22.04 /bin/bash -c './build_performous.sh -c -R -g && cp performous/build/*.deb /performous/packages'
```

### build executable from current directory
```
docker run --rm \
--mount type=bind,source=/tmp/performous-packages,target=/performous/packages \
--mount type=bind,source=./,target=/performous/host,readonly \
performous-docker-build:ubuntu22.04 /bin/bash -c \
'cp -r /performous/host /performous/local \
&& ./build_performous.sh -c -D /performous/local -R -g \
&& cp /performous/local/build/performous /performous/packages/'
```

### build appimage from current directory
```
docker run --rm \
--mount type=bind,source=/tmp/performous-packages,target=/performous/packages \
--mount type=bind,source=./,target=/performous/host,readonly \
performous-docker-build:ubuntu22.04 /bin/bash -c \
'export PACKAGE_VERSION="testing" \
&& cp -r /performous/host /performous/local \
&& cd /performous/local \
&& sed -i s/@@VERSION@@/${PACKAGE_VERSION}/ ./AppImageBuilder.yml \
&& mkdir build \
&& cd build \
&& cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_WEBSERVER=ON -DENABLE_WEBCAM=ON -DSELF_BUILT_CED=ALWAYS -DPERFORMOUS_VERSION=$PACKAGE_VERSION .. \
&& make -j$(nproc) install DESTDIR=../AppDir \
&& cd .. \
&& /usr/local/bin/appimage-builder --appimage-extract-and-run --recipe ./AppImageBuilder.yml --skip-test \
&& cp /performous/local/Performous-${PACKAGE_VERSION}-x86_64.AppImage /performous/packages/'
```
