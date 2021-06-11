# meta-xt-prod-devel

## Content
This document is long enough to has it's own content. So below you will find:
* What is meta-xt-prod-devel
* Quick start
  * Some basic and most used commands
* Usage and setup
  * Serial over USB
  * SSH over ethernet
  * What to do "on board"
* How to build
  * Build configs
  * Few build hints
* How to install
  * Write to microSD card
  * Write to eMMC
* How to prepare board (loaders and u-boot settings)
* Supported boards
* References

Important note.
All PC-related steps described bellow are intended to be run on Linux, and were tested on Ubuntu 18.
Windows and MacOS are not supported, sorry.

## What is meta-xt-prod-devel

This is meta layer that allows you to run Linux+Linux or Linux+Android at same time under Xen hypervisor on some Renesas' Gen3 R-CAR boards.
List of fully or conditionaly supported boards can be found at the end of this document.
Taking into account that you are interested in these sources, we suppose that you have some understanding what is Xen hypervisor.
In case you need additional info look into last section of this document - "References".
What prod-devel is?
* it is some reference implementation that demonstrate simultanious run of few OS on same hardware
Why Xen hypervisor?
* Xen can reboot guest OS promptly in case of some failure happen
* hardware may be shared between few OS

## Quick start

As far as this part is just quickstart, we will not dive deep into settings, options and details.
Also for simplicity we will build prod-devel with DomU (linux as guest OS) for StarterKit on Kingfisher.

> Important note.
> This product contains proprietary sources (like graphic drivers and stuff).
> As result, either you need to have access to proprietary repositories or you need to be provided with prebuilt binaries.

So, let's quickstart...

At first you need to create small configuration file, let's name it `my-devel-u.cfg` with following content:
```
[git]
xt_history_uri = ""
xt_manifest_uri = git@github.com:xen-troops/meta-xt-products.git
[path]
workspace_base_dir = /<your_storage>/work_devel
workspace_storage_base_dir = /<your_storage>/storage
workspace_cache_base_dir = /<your_storage>/work_devel/cache
[local_conf]
XT_GUESTS_INSTALL = "domu"
XT_GUESTS_BUILD = "domu"
# following line is used only if you are provided with prebuilt binaries
XT_RCAR_EVAPROPRIETARY_DIR = /path_to_folder/with_prebuilt_archives
```
If you use prebuilt binaries for build, then you need to use option `XT_RCAR_EVAPROPRIETARY_DIR`.
If you build with proprietary sources - this option is not needed.

Pay attention that you need to use drive with about 500GB of free space.

As far as meta-xt-prod-devel is meta layer, it's not intended to be build by itself, but as part of bigger system.
So, you need to use our tool to build required system.

```
git clone git@github.com:xen-troops/build-scripts.git
cd build-scripts
python ./build_prod.py --machine h3ulcb-4x2g-kf --product devel \
    --config my-devel-u.cfg --with-local-conf --with-do-build
```
Again, important comment here.
> If you are building using proprietary sources - use `--product devel-src`
> if you are building using prebuilt binaries - use `--product devel`.

Clean build takes about 5-6 hours on iCore7 with 32GB and SSD. Take into account that you build three operation systems.

If everything is fine (and you see no red lines with errors) - go to `<workspace_base_dir>/build/deploy` folder and run
```
./mk_sdcard_image.sh -p . -d ./image.img -c gen3
```

You have image that can be flashed to microSD card using `dd`.
```
sudo dd if=./image.img of="your_sd_card_like_/dev/sdX" bs=1M status=progress
```
or bmaptool which is strongly recommended due to higher speed.
```
sudo bmaptool copy --bmap image.img.bmap ./imgage.img your_sd_card_like_/dev/sdX
```
But if you want to have fastest speed of running - use internal eMMC.
This step is a little bit more complex than `dd` or `bmaptool` so you need to follow to section "How to install".

### Some basic and most used commands
Inside Dom0
```
xl list                           - list running domains
xl console DomD                   - switch to domD's console
xl destroy DomU                   - destroy domU
xl create /xt/dom.cfg/domu.cfg -c - create domU and switch to it's console
```
Inside domD/domU
```
Ctrl+5                            - return to dom0 from domD/domU
```

## Usage and setup
As far as it is reference implementation we will accent on developer's usage. You can connect to domain by:
1) serial over USB
2) SSH over ethernet

### Serial over USB
You need to connect your PC to board by USB-microUSB cable.

Use your favorite serial console (minicom, cu etc).

Set port to: 115200, 8N1, software flow control.

Connect your PC to board's micro USB connector named "Debug serial". If needed - use detailed and labeled photo in wiki https://github.com/xen-troops/meta-xt-prod-devel/wiki

Turn on board and see messages are running.

### SSH over ethernet
At first determine IP that is assigned to board.

If serial already works - you can login to domD and check:
```
dom0 login: root
# xl console DomD
domd login:root
# ifconfig
```
Or check your DHCP server. For example:
```
systemctl status isc-dhcp-server
```

Connect to board
```
ssh root@<board_ip>
```
After that you will be logged straight into DomD.
Also you can connect to DomU, just use port 23:
```
ssh -p 23 root@<board_ip>
```

### What to do "on board"
Use `root` to login to dom0 or domD (yes, it's just demo, and yes, we will close this security hole).

Main program inside dom0 is `xl`. You can use
```
xl list                            ## to see list of active domains
xl console DomD                    ## to switch to DomD's console (press Ctrl-5 to return to Dom0)
xl destroy DomU
xl create -c /xt/dom.cfg/domu.cfg  ## to create domain using specified config
```
Detailed manual for `xl` you can find at https://xenbits.xen.org/docs/unstable/man/xl.1.html

DomD is quite regular Linux so you can use lots of regular Linux tools to do lots of regular Linux things.

Also pay attention that domD is home for backend drivers for guest domains like DomU or DomA. This means that domU/domA is working with real hardware using drivers in domD. And if you stop network in domD, domU/domA will be isolated from world as well.

If you want to return to dom0 - use `Ctrl-5`.


## How to build
Due to complex dependecies between different projects, you need to use special python script to build final product. Script will pull required layers and set configuration files.
So you need to get script from github and edit few lines in config.
Either clone repo
```
git clone https://github.com/xen-troops/build-scripts.git
```
or just download and unpack archive
```
wget https://github.com/xen-troops/build-scripts/archive/master.zip
```
Folder contains two python files (script itself), and configs for different products and specific cases.

### Build configs
You need to use small configuration file for build. Few examples are provided with `build_scripts` project.
Config file looks like this:
```
[git]
xt_history_uri = ""
xt_manifest_uri = git@github.com:xen-troops/meta-xt-products.git
[path]
workspace_base_dir = /<your_storage>/work_devel
workspace_storage_base_dir = /<your_storage>/storage
workspace_cache_base_dir = /<your_storage>/work_devel/cache
[local_conf]
XT_GUESTS_INSTALL = "domu"
XT_GUESTS_BUILD = "domu"
# following line is used only if you are provided with prebuilt binaries
#XT_RCAR_EVAPROPRIETARY_DIR = /path_to_folder/with_prebuilt_archives
```
In most case you modify `[path]` part with your local paths.

Options specified in `[local_conf]` part are copied "as is" into `local.conf` of some domains.
Variables `XT_GUESTS_INSTALL` and `XT_GUESTS_BUILD` may have following values:
  - "domu" for Linux as guest OS
  - "doma" for Android as guest OS
  - empty sting for no guest OS. In this case you will get Dom0 and DomD only.
Variable `XT_RCAR_EVAPROPRIETARY_DIR` is used only if we build using prebuilt binaries. If we build using proprietary sources - it is not neeed.
Variable `AOS_VIS_PLUGINS` may be used with Android. It need proprietary sources, and you can just leave it empty.

Now you can run script:
```
python ./build_prod.py --machine <MACHINE> --product devel[-src] [--branch master] --config my-devel-u.cfg --with-local-conf --with-do-build
```
List of supported machines can be found at the end of this doc.
Option `product` may be `devel` for prebuilt binaried or `devel-src` fro build with proprietary sources.
Option `branch` is `master` by default, but it allows to specify exact release branch: `REL-v4.0`, `REL-v5.0`, `REL-v6.0` etc.
See list of branches on github repo for possible options.

### Few build hints
Pay attention that mentioned above command will erase `workspace_base_dir` and start clear build.

* If case of interruption you can just re-run script keeping all fetched sources:
```
python ./build_prod.py --machine h3ulcb-4x2g-kf --product devel --config my-devel-u.cfg --continue-build
```

* You can get "essence"/"core" of product, apply to it some pathes/modifications and continue build:
```
python ./build_prod.py --machine h3ulcb-4x2g-kf --product devel --config my-devel-u.cfg --with-local-conf
pushd <workspace_base_dir>/meta-xt-prod-devel
# do something, apply patches
popd
python ./build_prod.py --machine h3ulcb-4x2g-kf --product devel --config my-devel-u.cfg --continue-build
```

* To clear build products and intermediate files:
```
rm -rf <workspace_cache_base_dir>
cd <workspace_base_dir>/build
# remove everything except conf/ folder
rm -rf bitbake.lock buildhistory/ cache/ deploy/ log/ shared_rootfs/ tmp/
```

## How to install

### Write to microSD card
As already mentioned in Quick start section, you can use `dd` or `bmaptool`.
```
sudo dd if=./image.img of="/dev/sd?" bs=1M status=progress
```
or bmaptool which is strongly recommended due to higher speed
```
sudo bmaptool copy --bmap image.img.bmap ./imgage.img /dev/sd?
```

### Write to eMMC
We have two ways to put binaries onto eMMC: using SD card or using TFTP and NFS.

#### Write to eMMC using SD card
Detailed instruction with manual and scripted option is provided on wiki page
https://github.com/xen-troops/meta-xt-prod-devel/wiki/How-to-flash-image-to-eMMC-from-SD-card

#### Write to eMMC using network
We need to assume that you have TFTP and NFS servers working on your PC.
TODO Add page to wiki and put link here

## How to prepare board (loaders and u-boot settings)
TODO This section is under development now.

## Supported boards
List of supported MACHINE_NAMEs:
```
MACHINE_NAME         Board name         SoC       RAM  DomU support             DomA support
------------------------------------------------------------------------------------------------------
salvator-x-h3-4x2g   Salvator-X         H3 ES3.0  8GB
salvator-x-m3        Salvator-X         M3        4GB  prod-devel-unsafe only   Not supported for DomA
salvator-xs-m3-4x2g  Salvator-XS        M3        8GB  Supported from REL 7.0   WIP
salvator-xs-h3-4x2g  Salvator-XS        H3 ES3.0  8GB
salvator-xs-m3n      Salvator-XS        M3N       2GB                           Not supported for DomA
h3ulcb-4x2g          H3ULCB             H3 ES3.0  8GB
h3ulcb-4x2g-kf       H3ULCB KingFisher  H3 ES3.0  8GB
```

## References
* If you are not aware what is meta layer - you may read Yocto's documentation at https://www.yoctoproject.org/docs/
* If you want to understand what is hypervisor and guest OS https://xenproject.org/help/documentation/
* A lot of information regarding Renesas Gen3 boards https://elinux.org/Category:R-Car_Gen3
* Info regarding Renesas' Yocto releases https://elinux.org/R-Car/Boards/Yocto-Gen3/v5.1.0

