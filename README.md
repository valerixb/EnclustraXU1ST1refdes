<!-- TOC --><a name="enclustra-xu1-st1-reference-design"></a>
# Enclustra XU1 + ST1 reference design
Changes, porting and customizations of Enclustra reference design for XU1 module (XU1-6EG-1I-D11E-R4.1) + ST1 baseboard.

Documentation for the original Enclustra reference design is found here: <br>
https://github.com/enclustra/Mercury_XU1_ST1_Reference_Design

Original Enclustra Yocto layers are found in repo: <br>
https://github.com/enclustra/meta-enclustra-amd

Hardware documentation:<br>
http://download.enclustra.com/

At the time of these notes (october 2026), Enclustra reference design targets vivado v2024.2 + yocto with petalinux (SDK) flow with xsa for hardware export; this repo is a port to v2025.2 + yocto with EDF flow with SDT (system device tree) for hardware export.


<!-- TOC --><a name="contents"></a>
# Contents
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Enclustra XU1 + ST1 reference design](#enclustra-xu1-st1-reference-design)
- [Contents](#contents)
- [Installation procedure](#installation-procedure)
   * [Directory structure](#directory-structure)
   * [Clone git directory](#clone-git-directory)
   * [Setup Vivado project](#setup-vivado-project)
   * [Setup yocto](#setup-yocto)
      + [Init repo using xilinx edf manifest](#init-repo-using-xilinx-edf-manifest)
      + [Import Yocto custom configuration](#import-yocto-custom-configuration)
      + [Import Vivado project](#import-vivado-project)
      + [Build linux image](#build-linux-image)
      + [Make an SD card from the compiled image](#make-an-sd-card-from-the-compiled-image)

<!-- TOC end -->


<!-- TOC --><a name="installation-procedure"></a>
# Installation procedure
All the steps needed to setup projects for vivado and yocto (vitis is not used), starting from the files found in this repository, are now described.

<!-- TOC --><a name="directory-structure"></a>
## Directory structure
Make a base directory for your project in your local filesystem where it is convenient for you; in this directory we will later create this subdirectories (instructions provided in following sections):
- `gitcode` : local mirror of this git repo
- `vivado-prj` : base directory for the Vivado design of the PL (programmable logic part of FPGA). This directory can be moved to any location.
- `yocto-prj` : base directory to compile linux image with boot binaries, kernel and rootfs. This directory cannot be be moved to other locations, as it contains absolute paths; if you want to move it, you must start a new yocto project with repo init etc.

<!-- TOC --><a name="clone-git-directory"></a>
## Clone git directory
When you clone a remote git repo to your local directory, it takes the name of the repo; we will then rename it to "gitcode" for clarity:
```
cd <base dir>
git clone git@github.com:valerixb/EnclustraXU1ST1refdes.git
mv EnclustraXU1ST1refdes gitcode
```

<!-- TOC --><a name="setup-vivado-project"></a>
## Setup Vivado project
In this git repo there is a backup of the Vivado project for the FPGA PL in directory `vivado_prj_backup`; it contains a tcl script that recreates the project. The script itself provides useful info in the comments and in the "print_help" method. Anyway, one can just open Vivado from the local `vivado_prj_backup` directory and run the script from vivado tcl command line:
```
source ./ME-XU1-6EG-1I-D11E.tcl
```
Then save the project into the `vivado-prj` directory that you created previously.
Note that Vivado project directory can be moved, as it does not contain absolute paths.


<!-- TOC --><a name="setup-yocto"></a>
## Setup yocto
<!-- TOC --><a name="init-repo-using-xilinx-edf-manifest"></a>
### Init repo using xilinx edf manifest
Instructions to start a local yocto project using Xilinx EDF as a base (note the manifest selection by option "-m"); I put everything on a new branch called "xu1st1v2025p2".
```
cd <base dir>
mkdir -p yocto-prj
cd yocto-prj
repo init -u https://github.com/Xilinx/yocto-manifests.git -b rel-v2025.2 -m default-edf.xml
repo sync
repo start xu1st1v2025p2 --all
source edf-init-build-env
```
Note that the script performs an automatic `cd ./build`, so your shell is now in build subdirectory. From now on it is assumed that the `edf-init-build-env` has been sourced.


<!-- TOC --><a name="import-yocto-custom-configuration"></a>
### Import Yocto custom configuration
Just copy (merge) the content of the directory `<base dir>/gitcode/yocto` into `<base dir>/yocto-prj`. It's a customization of `<base dir>/build/conf` files plus the addition of new layers in the `source` directory; note that these layers have been ported to v2025.2, so they differ from the ones found in the original Enclustra repo. In `build` dir there are also some useful shell scripts to automate common yocto procedures (build, make a SD card from the image).


<!-- TOC --><a name="import-vivado-project"></a>
### Import Vivado project
Export .xsa from vivado (include bitstream, but do not include binary, which is the format used for dynamic PL reconfig); then generate the system device tree:
```
cd <base dir>/vivado-prj
sdtgen -xsa Mercury_XU1_ST1.xsa \
       -dir output_sdt \
       -user_dts "../yocto-prj/sources/meta-enclustra-module/recipes-bsp/device-tree/files/zynqmp_enclustra_common.dtsi \
                  ../yocto-prj/sources/meta-enclustra-module/recipes-bsp/device-tree/files/zynqmp_enclustra_mercury_xu1.dtsi \
                  ../yocto-prj/sources/meta-enclustra-baseboard/recipes-bsp/device-tree/files/zynqmp/enclustra_mercury_st1.dtsi \
                  ../yocto-prj/sources/meta-enclustra-baseboard/recipes-bsp/device-tree/files/zynqmp/system-user.dtsi"

cd <base dir>/yocto-prj
source edf-init-build-env
```
(note: now we are in `build` subdir)
```
bitbake-layers create-layer ../sources/meta-vivadoprj
bitbake-layers add-layer ../sources/meta-vivadoprj
```
Remove the auto generated `sources/meta-vivadoprj/recipes-example` directory, as we won't use that.

Now we import the SDT (=vivado design) into yocto invoking `gen-machine-conf` :
```
MACHINE=refdes-xu1-st1 gen-machine-conf parse-sdt \
    --hw-description ../../vivado-prj/output_sdt \
    -c ../sources/meta-vivadoprj/conf \
    --require-machine refdes-xu1-st1 \
    --machine-overrides enclustra-sd \
    --machine-name refdes-xu1-st1
```

Note that I did not use the `-l ./conf/local.conf` option to avoid messing up `local.conf`, which already contains everything is needed.



<!-- TOC --><a name="build-linux-image"></a>
### Build linux image
As always, start initializing the yocto environment:
```
cd <base dir>/yocto-prj
source edf-init-build-env
```
Then run the shell script:
```
./yoctix-build
```
It performs all the necessary steps to make clean and build an image with bootbin, kernel and rootFS.

<!-- TOC --><a name="make-an-sd-card-from-the-compiled-image"></a>
### Make an SD card from the compiled image
Insert a SD card with size of 32 GB minimum; in the script it is assumed that it is seen as `/dev/sde` (change it if it's different on your computer). As always, start initializing the yocto environment:
```
cd <base dir>/yocto-prj
source edf-init-build-env
```
then run the shell script:
```
./yoctix-makeSD
```
This will partition the SD and copy all the necessary files (it takes some time). Then eject the SD card and put it into the ST1 baseboard. You are now ready to boot.

As this uses standard Xilinx EDF Yocto configuration, the login ID is `amd-edf` and the password is defined by the user at first login. To switch to super user (root), do `sudo su` at command prompt, using the password you entered for the `amd-edf` user.

Before shutting down, remember to issue the command `sudo poweroff`.
