# beagleV-yocto-image
# BeagleV-Ahead Yocto Build.

This document records the work I did on the BeagleV-Ahead in the same
sequence I followed it, from preparing the Yocto environment to building
the image.

------------------------------------------------------------------------

## 1. Workspace and sources

I used this workspace layout:

```text
~/yocto/
├── bitbake/
├── openembedded-core/
├── meta-riscv/
└── build-beaglev-ahead/
```

I cloned BitBake, OpenEmbedded-Core, and the RISC-V BSP layer:

```bash
mkdir -p ~/yocto
cd ~/yocto
git clone https://git.openembedded.org/bitbake
git clone https://git.openembedded.org/openembedded-core
git clone https://github.com/riscv/meta-riscv.git
```

All repositories must use compatible branches. Mixing a Yocto release branch
with an incompatible newer `meta-riscv` branch caused parsing errors in an
earlier attempt. For reproducibility, the exact revisions can be saved with:

```bash
git -C ~/yocto/bitbake rev-parse HEAD
git -C ~/yocto/openembedded-core rev-parse HEAD
git -C ~/yocto/meta-riscv rev-parse HEAD
```
------------------------------------------------------------------------

## 2. Preparing Python

I used Python 3.10.14 through `pyenv`.

``` bash
pyenv shell 3.10.14
python3 --version
```

The output was:

``` text
Python 3.10.14
```

------------------------------------------------------------------------

## 3. Creating the BeagleV-Ahead build environment

From the Yocto workspace, I initialized a separate build directory for
the board:

``` bash
cd ~/yocto
source openembedded-core/oe-init-build-env build-beaglev-ahead
```

This created/entered:

``` text
~/yocto/build-beaglev-ahead
```

I then made sure that the build included the required layers:

I checked them with:

``` bash
bitbake-layers show-layers
```

------------------------------------------------------------------------

## 4. Selecting the target board

I configured the target machine in:

``` text
~/yocto/build-beaglev-ahead/conf/local.conf
```

using:

``` conf
MACHINE = "beaglev-ahead"
```

I verified that BitBake was actually using this machine with:

``` bash
bitbake -e core-image-minimal | grep '^MACHINE='
```

which showed:

``` text
MACHINE="beaglev-ahead"
```

The target system was RISC-V 64-bit:

``` text
TARGET_SYS = riscv64-oe-linux
```

Allow root login without a password

To prevent the `Login incorrect` problem I initially encountered, I added this
line to `local.conf`:

```conf
EXTRA_IMAGE_FEATURES += "allow-empty-password empty-root-password allow-root-login"
```
------------------------------------------------------------------------

## 5. Limiting the build to four parallel jobs
 I
limited the parallelism in `local.conf`:

``` conf
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j 4"
```
------------------------------------------------------------------------

## 6. Building `core-image-minimal`

I started the image build with:

``` bash
bitbake core-image-minimal
```

The build completed successfully.

All tasks were executed successfully.

The important point here is that the Yocto build itself was successful.

------------------------------------------------------------------------

