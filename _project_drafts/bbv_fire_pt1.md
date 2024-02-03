---
layout: page
title: Experimenting with the BeagleV-Fire
date: 2024-01-13 00:00:00-0000
description: My experience getting started with a BeagleV-Fire SoC (RiscV + FPGA) SBC.
img: <insert image>
importance: 1
category: Embedded Hardware
giscus_comments: false
giscus_repo: <repo name>
toc:
  sidebar: left
---
## Topics Covered

- My experience getting started with the BeagleV-Fire SoC development board.

## Background

I have been reading a lot about RISC-V over the last year and have been wanting to experiment with some of the many popular development boards. When I came across an article about the PolarFire System on Chip (SoC) design from Microchip I was instantly intrigued about the possibilities of combining the RISC-V with on chip FPGA resources. So I ordered a new BeagleV-Fire development board to tinker with it and try out some cross-platform development.

## My Journey

### Boot-up and Go

I started this experiment on a fresh installation of Ubuntu 22.04 which I then configured with my development preferences and utilities.

The BeagleV-Fire comes with a version of Ubuntu installed on the eMMC out of the box. It was able to connect it to the LAN, start the board up, and ssh into it in a few minutes with no problems. It even came with an older version of Rust pre-installed which was a nice surprise. I ran into a minor issue when trying to update Rust to the newest version because while ```cargo``` and ```rustc``` are installed, ```rustup``` is not. The eMMC is only 16 Gb so I can understand not wanting to include the package manager with the core functionalities. I went to update the Rust components but received an error because ```rustup``` detected an existing installation. I removed ```rustc``` with the package manager and the normal rustup installation proceeded without issue.

 ```console
 sudo apt remove rustc
 ```

With the latest version of Rust installed, I could created a new cargo package with a "Hello World" message and everything worked fine.

### Imaging the SoC

The MPFS250T (PolarFire SoC) is a little different than other devices I have worked with in that the eMMC and microSD port are multiplexed in the MPFS250T so that only one or the other can be accessed by the chip directly. The BeagleV-Fire designers chose the eMMC route for booting and OS storage and exposed the microSD port through the OS via a SPI-MMC device in the device tree. For bulk data storage I inserted a formatted microSD card and searched for it with a ```lsblk``` command where it showed up as ```mmcblk1```.

```console
beagle@beaglev:~$ lsblk
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mmcblk0      179:0    0  14.6G  0 disk
├─mmcblk0p1  179:1    0   684K  0 part
├─mmcblk0p2  179:2    0    60M  0 part /boot/firmware
└─mmcblk0p3  179:3    0  14.5G  0 part /
mmcblk1      179:8    0 119.1G  0 disk
└─mmcblk1p1  179:9    0 119.1G  0 part
mmcblk0boot0 179:16   0     4M  1 disk
mmcblk0boot1 179:24   0     4M  1 disk
```

Unfortunately, after trying everything could google could throw at me, I couldn't get the drive to mount properly for use as additional storage space. The discord support forum had several posts that the support for the sd card wasn't ready yet so maybe this will be working soon.

To access the eMMC for flashing a new OS image, the multiplexer needs to be switched to expose the eMMC storage through the USB-C port. This is done with a serial RS-232 connection on the debug header. I setup a Raspberry Pi Debug probe and hooked Rx, Tx, and Gnd up according to the [instructions](https://docs.beagle.cc/latest/boards/beaglev/fire/demos-and-tutorials/flashing-board.html#beaglev-fire-flashing-board) and opened a serial terminal with [Tio](https://github.com/tio/tio). Rebooting produces a progress bar and right as it completes there is a prompt to hit any key alphanumeric key (space doesn't work). When I timed it right, the HSS command prompt showed up and I could switch the eMMC to the USB-C. My Ubuntu installation detected it immediately and mounted it without issues. Flashing was straightforward with the following highlights:

- The most recent images can be found in this [repository](https://www.beagleboard.org/distros).
- The documentation implied that would find a ```*.img``` file but I didn't find one in the download.
- After poking around a bit I figured out that Balena etcher would accept the ```sdcard.img.xz``` file so that is what I used.
- After flashing, the first boot took a bit longer and appeared to hang after a verification step of some kind. This might or might not be intentional. Resetting the board proceeded normally on the second boot.

### Software Installation

 The PolarFire SoC requires Microchip's Libero Design Suite. When installing the suite there are a few gotchas that can derail the process and make it much more painful that necessary. I recommend following [the instructions here](https://docs.beagle.cc/latest/boards/beaglev/fire/demos-and-tutorials/mchp-fpga-tools-installation-guide.html#beaglev-fire-mchp-fpga-tools-installation-guide) as they mostly cover those gotchas. I will highlight what I learned from my experience.

- Install the Linux Standard Base (lsb) package. This is required for the licensing daemons or they output a misleading error that will lead down a rabbit hole.
- Create a directory to contain all the Microchip software. Something like ```~/Microchip``` will work well and will prevent manual modification of helpful script files.
- Do not install the Microchip software in the default location.
- Do install Libero, SoftConsole, Licenses, etc in your ```~/Microchip``` directory.
- Read the post-install instructions after installing SoftConsole. If that step is missed, the documentation can be found after the install at "~/Microchip/SoftConsole-v2022.2-RISC-V-747/documentation/softconsole/using_softconsole/post_installationhtml"
- Take advantage of the helpful script provided in the [Microchip FPGA Tools Setup](https://openbeagle.org/beaglev-fire/Microchip-FPGA-Tools-Setup) repo. Make sure to update it with your installation specific path details. It will then start the licensing daemons for you and configure the correct environment variables.

### FPGA Programming (Defaults)

The documentation advises against trying to create FPGA logic from a blank Libero project. Instead the [Gateware repository](https://openbeagle.org/beaglev-fire/gateware) provides helpful scripts that create a pre-configured Libero project for you that can be customized to your needs.

Before I create custom FPGA logic, it makes sense to build and test a default bit stream to learn the process. Even though I probably don't need local copies of [these repositories](https://openbeagle.org/beaglev-fire), having them does make searching for documentation and figuring out how things go together much easier. I start by cloning them into a top level folder and end up with a tree like this:

```console
./bbv_fire/
├── beaglev-fire
├── beaglev-fire-ubuntu
├── gateware
├── gateware-snapshots
├── hart-software-services
└── microchip-fpga-tools-setup
```

Next, I installed the Python 3 dependencies and tried to run the default build script.

```console
python3 build-bitstream.py ./build-options/default.yaml
```

The ```build-bitstream.py``` script worked well and provided useful output and guidance for problems. In my case, the ```setup-microchip-tools.sh``` script that I modified to point to my installation directory contained an invalid path so I had to double-check my ```PATH``` exports and compare them with the actual install paths. Once I had correct ```PATH``` variables, the build completed without any issues.

Performing a git status on the repo shows that the build process has generated a few new folders. The ```gateware/work/``` contains a pre-compiled HSS bootloader binary in addition to the script generated Libero project that executed the FPGA synthesis, routing, etc to generate the bitstream. The ```gateware/bitstream``` folder contains the actual bitstream output I am most interested in.

Now the [documentation](https://docs.beagleboard.org/latest/boards/beaglev/fire/demos-and-tutorials/gateware/gateware-full-flow.html) didn't provide a large amount of detail as to how to actually load the FPGA image but it does point to a script to execute on the BeagleV device at ```/usr/share/beagleboard/gateware```. I opened up that script and determined that it does the following things:

- Checks for sudo privileges.
- Checks the first argument to see if it is a valid directory.
- Checks for the presence of both ```LinuxProgramming/mpfs_bitstream.spi``` and ```LinuxProgramming/mpfs_dtbo.spi```.
- If both checks succeed, calls the ```update-gateware.sh``` script in ```/usr/share/microchip/gateware``` to do the heavy lifting.

So I used ```scp``` to copy my generated ```LinuxProgramming``` directory over to the home directory on the BeagleV device and ran the script:

```console
beagle@BeagleV:/usr/share/beagleboard/gateware$ sudo ./change-gateware.sh /home/beagle/
```

The BeagleV device took a few minutes to finish flashing and rebooting and everything worked fine. It did feel strange to me to provide the parent of the ```LinuxProgramming``` folder to the script so I may modify that so that it uses that folder directly.

Checking the serial console output from the reboot shows that the ```Design Name``` is the same as it was before the flashing process. This might be expected since I built the default configuration so I will check this again in the next step when I build custom FPGA logic.

```console
[5.779029] Design Info:
    Design Name: CI_DEFAULT_FD28A2CA1789CDC1137
    Design Version: 02.00.2
```

### FPGA Programming (Custom)

Now that I know more about how the gateware creation process works I want to "unwind the stack" a bit and see how I can customize my FPGA. I will note that the documentation, while very helpful where it exists, is often missing. Be prepared to dive into the scripts to see what they do. The ```build-bitstream.py``` is definitely worth a read and is almost better than a readme file.  

The ```build-bitstream.py``` needs a ```yaml``` configuration file so I did a quick search and compared between the ```default.yaml``` and the others to see how they work. They all had the ```HSS``` object in common and I don't want to modify that so it is safe to assume I can copy that section for my custom build. The also define a ```gateware``` datatype which is what I want to customize so I again compared them all. They shared a ```type: sources``` and listed various ```build-args```. Looking through the ```build-bitstream.py``` helped me to understand this better and it turns out that each top level object is a key and each one is parsed to check it's ```type```. Depending on the ```key.type``` different actions are performed. For ```type: git```, the repository is cloned into the ```gateware/sources/``` folder and ```*.zip``` files are extracted there. If ```type: sources``` no action is taken and the directory is added to the list of sources.

```yaml
HSS:
    type: git
    link: https://git.beagleboard.org/beaglev-fire/hart-software-services.git
    branch: develop-beaglev-fire
    board: bvf
gateware:
    type: sources
    build-args: "M2_OPTION:NONE CAPE_OPTION:VERILOG_TUTORIAL"
    unique-design-version: 9.0.2
```

Since no action is taken if ```type: sources```, the ```gateware/sources``` folder seemed like a good place to start looking for more information. The ```FPGA-Design``` sub-folder does have additional documentation on how to create a FPGA Libero project from a tickle script. The ```Readme.md``` file provides an example command and documentation on the various options available. Running the command will trigger Libero to build a FPGA project through ```*.tcl``` scripts using the provided options for configuration.

```console
libero SCRIPT:BUILD_BVF_GATEWARE.tcl "SCRIPT_ARGS: ONLY_CREATE_DESIGN M2_OPTION:NONE CAPE_OPTION:NONE"
```

Reading through the available ```SCRIPT_ARGS``` I notice that the ```CAPE_OPTION``` provides additional details on how to customize my FPGA:

>... Valid values are the directory names in ./script_support/components/CAPE. If you wish to create an alternate build option, add a new directory in ./script_support/components/CAPE using one of the existing ones as template. This is a good place to start if you want to play with FPGA digital logic.

Perfect! Now I need to figure out which one to use as a template to start from. Again I do a quick search and find that there is a ```Readme.md``` file in most of the ```FPGA_design/script_support/components/CAPE/*``` sub-folders that contain pin mapping documentation. I can compare these file in vscode or use a ```diff``` command to see what changes are made from the default. I remembered noticing the ```gateware/custom-fpga-design/my_custom_fpga_design.yaml``` configuration file in a earlier step and one of the ```build-args``` was ```VERILOG_TUTORIAL``` which seemed to bea good starting point for custom logic. So I start to dig around a bit more and find that in ```FPGA_design/script_support/components/CAPE/``` there is both a ```VERILOG_TUTORIAL``` and ```VERILOG_TEMPLATE``` folder. Either of which should be viable for cloning and customizing. Unfortunately, I didn't see a ```Readme.md``` file for either of these capes but they both have a ```HDL``` sub-folder that includes Verilog text files. It was pretty easy to view these in vscode to see and compare them the defaults.

For experimentation I decided to start with a clone of the ```VERILOG_TEMPLATE``` folder and see if I could modify it as I would with custom logic so that it produces the same output as the ```VERILOG_TUTORIAL```.

```console
$ cp -r ./VERILOG_TEMPLATE ./CUSTOM_CAPE
```

Once the template is copied, I open the new folder up in vscode and perform a search for ```VERILOG_TEMPLATE``` and replace it with my new name ```CUSTOM_CAPE```. In this case, it only affected the the ```ADD_CAPE.tcl``` script.
Next I needed to create the Libero project so I can write, simulate, and synthesize my custom design.

```console
aven:~/bbv_fire/gateware/sources/FPGA-design$ libero SCRIPT:BUILD_BVF_GATEWARE.tcl "SCRIPT_ARGS: ONLY_CREATE_DESIGN PROJECT_LOCATION:\"~/repos/riscv-fpga/my_custom_cape\" TOP_LEVEL_NAME:\"my_custom_cape\" M2_OPTION:NONE CAPE_OPTION:CUSTOM_CAPE"
```

Once the script is complete, I can open Libero and then open the newly created project in the folder I specified in the script. From here, I created a new HDL source file with the ```Create HDL``` wizard and named it ```my_logic.v```. I am only interested in "simulating" the actual FPGA logic design process for now so instead of creating HDL from scratch, I copied the logic from ```VERILOG_TUTORIAL/HDL/blinky.v``` and updated the ```CAPE.v``` file to connect the wires. I double-checked the work with a ```diff``` command on my HDL folder and the tutorial to make sure I updated everything right. Then I executed the design flow and went through synthesis, place & route, etc. without any issues.

There were several hundred warnings generated about "User defined pragma syn_black_box" declarations but from what I could tell, were possibly due to weird formatting of the code generated ```polarfire_syn_comps.v``` file. They didn't seem to have any real impact on my custom logic.

The next step was to copy my newly created and tested HDL files back into my cloned cape. Note that the script that generated the project also creates extra HDL files that weren't in the original ```CUSTOM_CAPE/HDL``` folder. When I run the ```build-bitstream.py``` script these should get re-generated so I don't think they need to be copied. I did copy them over anyway to see what would happen and it didn't seem to break anything. Checking the output ```work``` directory didn't indicate any problems. These files include:

- miv_*.v
- apb_arbiter.v
- AXI4_address_shim.v

Now I am ready to create my own ```custom_cape.yaml``` file from a copy of ```gateware/custom-fpga-design/my_custom_fpga_design.yaml```. I update the ```build-args``` with the options that I want then take care to set ```unique-design-version``` to something new. There are a few things to note here:

- The FPGA bitstream loading application will skip the flashing process if the ```Design Version``` on the FPGA matches what is already stored on the FPGA. So this is something that will need to be updated every build.
- The ```build-bitstream.py``` script will look for "custom-fpga-design" in the path, and if the ```*.yaml``` file is stored there it will specify the design version based on a hash of the git repo. In practical terms, this means that if my ```custom_cape.yaml``` is located in a folder named "custom-fpga-design" the design name and version will be the same from build to build and the FPGA loading operation will be skipped. I can work around this if I place my configuration file in a different folder or by making a commit to the ```gateware``` repository between builds.

```yaml
# gateware/custom_cape.yaml
HSS:
    type: git
    link: https://git.beagleboard.org/beaglev-fire/hart-software-services.git
    branch: develop-beaglev-fire
    board: bvf
gateware:
    type: sources
    build-args: "TOP_LEVEL_NAME:\"my_custom_cape\" M2_OPTION:NONE CAPE_OPTION:CUSTOM_CAPE"
    unique-design-version: 0.0.1
```

With those changes in place I can kick off a build...

```console
gateware$ python3 build-bitstream.py ./custom_cape.yaml
```

When the build completes, I check the ```gateware/bitstream/LinuxProgramming``` folder and see I have newly created bitfiles. Just like before I use SCP to copy the ```LinuxProgramming``` folder over to the home directory on the BeagleV and execute the ```change-gateware.sh``` script. The serial output from the BeagleV then shows I have the correct design name and version installed.

```console
[2.09806] Design Info:
    Design Name: CUSTOM_CAPE_A5E630863F21B808BF
    Design Version: 00.00.2
```

I then take a look at the board and see user LED 5 flashing on and off like I would expect! I took a scope to the output to check the timing and noticed that when the pin is in a "High" state it is actually super noisy around 1.5 volts. I surmise there is a problem with the logic that is causing the pin to turn on and off faster than my scope can measure. I will have to investigate further when I make part 2.

## Wrapping Up

## Next Steps

The logic doesn't appear to be working quite right so I need to investigate that and figure out what is going on.
The workflow I have so far is functional but has room for improvement. I would like to have my FPGA design in a standalone project within it's own repository so I am thinking I could do the following:

- Start off with newly created ```*.yaml``` file with a new ```key``` for the high level details of a new project such as PATH and cape to copy.
- Create my own python script that clones the ```VERILOG_TEMPLATE``` cape and updates the tickle script automatically with the new name.
- Then further update that script so that it creates a standalone project from the same ```*.yaml``` file that I use for programming the device.
- I could then modify the existing python scripts to copy the HDL folder from a ```type: external_source```.

## Additional Resources

- [Tio](https://github.com/tio/tio)
- [Top-Level BeagleV-Fire Repository](https://openbeagle.org/beaglev-fire)
- [Gateware Repository](https://openbeagle.org/beaglev-fire/gateware)
