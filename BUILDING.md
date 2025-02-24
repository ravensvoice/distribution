# Developing and Building JELOS

JELOS is a fairly unique distribution as it is *built to order* and only enough of the operating system and applications are built for the purpose of booting and executing emulators and ports.  Developers and others who would like to contribute to our project should read and agree to the [Contributor Covenant Code of Conduct](https://github.com/JustEnoughLinuxOS/distribution/blob/main/CODE_OF_CONDUCT.md) and [Contributing to JELOS](https://github.com/JustEnoughLinuxOS/distribution/blob/main/CONTRIBUTING.md) guides before submitting your first contribution.


## Filesystem Structure
We have a simple filesystem structure adopted from parent distributions CoreELEC, LibreELEC, etc.

```
.
├── build.JELOS-DEVICE.ARCHITECTURE
├── config
├── distributions
├── Dockerfile
├── licenses
├── Makefile
├── packages
├── post-update
├── projects
├── release
├── scripts
├── sources
└── tools
```

**build.JELOS-DEVICE.ARCHITECTURE**

Build roots for each device and that devices architecture(s).  For ARM devices JELOS builds and uses a 32bit root for several of the cores used in the 64bit distribution.

**config**

Contains functions utilized during the build process including architecture specific build flags, optimizations, and functions used throughout the build workflow.

**distributions**

Distributions contains distribution specific build flags and parameters and splash screens.

**Dockerfile**

Used to build the Ubuntu container used to build JELOS.  The container is hosted at [https://hub.docker.com/u/justenoughlinuxos](https://hub.docker.com/u/justenoughlinuxos)

**licenses**

All of the licenses used throughout the distribution packages are hosted here.  If you're adding a package that contains a license, make sure it is available in this directory before submitting the PR.

**Makefile**

Used to build one or more JELOS images, or to build and deploy the Ubuntu container.

**packages**

All of the package set that is used to develop and build JELOS are hosted within the packages directory.  The package structure documentation is available in [PACKAGE.md](PACKAGE.md)

**post-update**

Anything that is necessary to be run on a device after an upgrade should be added here.  Be sure to apply a guard to test that the change needs to be executed before execution.

**projects**

Hardware specific parameters are stored in the projects folder, anything that should not be included on every device during a world build should be isolated to the specific project or device here.

**release**

The output directory for all of the build images.

**scripts**

This directory contains all of the scripts used to fetch, extract, build, and release the distribution.  Review Makefile for more details.

**sources**

As the distribution is being built, package source are fetched and hosted in this directory.  They will persist after a `make clean`.

**tools**

The tools directory contains utility scripts that can be used during the development process, including a simple tool to burn an image to a usb drive or sdcard.

## Building JELOS
Building JELOS requires an Ubuntu 20.04 host with approximately 800GB of free space for a full world build.  Other Linux distributions may be used when building using Docker.

### Building with Docker
Building JELOS is easy, the fastest and most recommended method is to use Docker.  At this time building the distribution using Docker is only known to work on a Linux system.  To build JELOS use the table below.

| Device | Dependency | Docker Command |
| ---- | ---- | ---- |
|RG552||```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-RG552```|
|RG503||```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-RG503```|
|RG353P|RG503|```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-RG353P```|
|RG351P||```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-RG351P```|
|RG351V|RG351P|```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-351V```|
|RG351MP|RG351P|```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-RG351MP```|
|x86_64||```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-X86_64```|
|ALL DEVICES||```PYTHON_EGG_CACHE="`pwd`/.egg_cache" make docker-world```|

> Devices that list a dependency require the dependency to be built first as that build will be used as the root of the device you are building.

### Building Manually
To build JELOS manually, you will need several prerequisite packages installed.

```
sudo apt install gcc make git unzip wget \
                xz-utils libsdl2-dev libsdl2-mixer-dev libfreeimage-dev libfreetype6-dev libcurl4-openssl-dev \
                rapidjson-dev libasound2-dev libgl1-mesa-dev build-essential libboost-all-dev cmake fonts-droid-fallback \
                libvlc-dev libvlccore-dev vlc-bin texinfo premake4 golang libssl-dev curl patchelf \
                xmlstarlet patchutils gawk gperf xfonts-utils default-jre python xsltproc libjson-perl \
                lzop libncurses5-dev device-tree-compiler u-boot-tools rsync p7zip libparse-yapp-perl \
                zip binutils-aarch64-linux-gnu dos2unix p7zip-full libvpx-dev bsdmainutils bc meson p7zip-full \
                qemu-user-binfmt zstd parted
```

Next, build the version of JELOS for your device.  See the table above for dependencies.  If you're building for the RG351V, RG351P will be built first to provide the build root dependency.  To execute a build, run `make {device}`

```
make RG351V
```

### Building a single package
It is also possible to build individual packages.
```
DEVICE=RG351V ARCH=aarch64 ./scripts/clean busybox
DEVICE=RG351V ARCH=aarch64 ./scripts/build busybox
```

> Note: Emulation Station package build requires additional steps because its source code located in a separate repository, see instructions inside, [link](https://github.com/JustEnoughLinuxOS/distribution/blob/main/packages/ui/emulationstation/package.mk).

### Special env variables
For development build, you can use the following env variables to customize the image. Some of them can be included in your `.bashrc` startup shell script.

**SSH keys**
```
export JELOS_SSH_KEYS_FILE=~/.ssh/jelos/authorized_keys
```
**WiFi SSID and password**
```
export JELOS_WIFI_SSID=MYWIFI
export JELOS_WIFI_KEY=secret
```

**Screenscraper, GamesDB, and RetroAchievements**

To enable Screenscraper, GamesDB, and RetroAchievements, register at each site and apply the api keys in ~/developer_settings.conf. This configuration is picked up by EmulationStation during the build.

```
export SCREENSCRAPER_DEV_LOGIN="devid=DEVID&devpassword=DEVPASSWORD
export GAMESDB_APIKEY="APIKEY"
export CHEEVOS_DEV_LOGIN="z=DEVID&y=DEVPASSWORD"
```

### Creating a patch for a package
It is common to have imported package source code modifed to fit the use case. It's recommended to use a special shell script to built it in case you need to iterate over it. See below.

```
cd sources/wireguard-linux-compat
tar -xvJf wireguard-linux-compat-v1.0.20211208.tar.xz
mv wireguard-linux-compat-v1.0.20211208 wireguard-linux-compat
cp -rf wireguard-linux-compat wireguard-linux-compat.orig

# Make your changes to wireguard-linux-compat
mkdir -p ../../packages/network/wireguard-linux-compat/patches/RG503
# run from the sources dir
diff -rupN wireguard-linux-compat wireguard-linux-compat.orig >../../packages/network/wireguard-linux-compat/patches/RG503/mychanges.patch
```

### Creating a patch for a package using git
If you are working with a git repository, building a patch for the distribution is simple.  Rather than using `diff`, use `git diff`.
```
cd sources/emulationstation/emulationstation-098226b/
# Make your changes to EmulationStation
vim/emacs/vscode/notepad.exe
# Make the patch directory
mkdir -p ../../packages/ui/emulationstation/patches
# Run from the sources dir
git diff >../../packages/ui/emulationstation/patches/005-mypatch.patch
```

After patch is generated, one can rebuild an individual package, see section above. The build system will automatically pick up patch files from `patches` directory. For testing, one can either copy the built binary to the console or burn the whole image on SD card.

### Building an image with your patch
If you already have a build for your device made using the above process, it's simple to shortcut the build process and create an image to test your changes quickly using the process below.
```
# Update the package version for a new package, or apply your patch as above.
vim/emacs/vscode/notepad.exe
# Export the variables needed to complete your build, we'll assume you are building for the RG503, update the device to match your configuration.
export OS_VERSION=$(date +%Y%m%d) BUILD_DATE=$(date)
export PROJECT=Rockchip DEVICE=RG503 ARCH=aarch64
# Clean the package you are building.
./scripts/clean emulationstation
# Build the package.
./scripts/build emulationstation
# Install the package into the build root.
./scripts/install emulationstation
# Generate an image with your new package.
./scripts/image mkimage
```
