# [superbird-bulkcmd](https://github.com/frederic/superbird-bulkcmd)


[Spotify Car Thing](https://carthing.spotify.com/) (superbird) resources to access U-Boot shell over USB. Not a bug, it is a ["feature"](https://miro.medium.com/max/1200/1*KDfUqn6c66axcbsTPPWSpQ.jpeg).

![Hacked Car Thing](https://i.imgur.com/VRjOR5v.jpg)

*Note: this method has been tested on the factory firmware (device never used/updated : App Version 0.24.107 - OS Version 6.3.29), but should work on all firmware versions released as of this article's writing.*

# Disclaimer
You are solely responsible for any damage caused to your hardware/software/keys/DRM licences/warranty/data/cat/etc...

# Requirements
- A Car Thing (superbird) without USB password
- Either a USB A to C, or a C to C cable
- A PC running some flavor of 64-bit GNU Linux
- `libusb-dev` installed

# FAQ
Does this process void my warranty on this device?
- Probably, assume so.

Can I OTA afterwards?
- If you don't perform any persistent change, probably yes.
- But if you disable dm-verity and modify on-device partitions, OTA updates will fail, though given this device is EOL, we don't expect further OTA updates.

Can I still use stock features ?
- Yes! Perfectly normal and usable, this just enables root access and ADB.

Can I go back to stock after installing custom OS's or messing up the stock image?
- Theoretically, if you have a good eMMC dump, the U-Boot shell should allow you to restore the partitions. **But this has not been tested thoroughly!**

# Files
- /bin/: prebuilt set of required tools
  - [update](https://github.com/khadas/utils/blob/master/aml-flash-tool/tools/linux-x86/update): Client for the USB Burning protocol implemented in Amlogic bootloaders
- /images/: prebuilt images to upload via USB
- /initrd/: files to customize the initrd image
- /scripts/: scripts used to simplify interactions with the devices

# Guide : U-Boot shell over USB (*USB burning mode*)
1. Unplug the Car Thing from everything
2. Clone/Download this repo locally, and change your shell's directory to it & ensure you `libusb-dev` installed
3. Hold buttons 1 & 4 on the case, and plug the Car Thing into your PC via USB

The host should see a new USB device connection in `dmesg` like this one:
```text
usb 1-1: New USB device found, idVendor=1b8e, idProduct=c003, bcdDevice= 0.20
usb 1-1: New UDSB device strings: Mfr=1, Product=2, SerialNumber=0
usb 1-1: Product: GX-CHIP
usb 1-1: Manufacturer: Amlogic
```
4. Release the button once this device has been detected by host computer.
5. Execute script [scripts/burn-mode.sh](scripts/burn-mode.sh) to boot U-Boot in *USB burning mode*.
A new USB device appears on host side :
```
usb 1-1: New USB device found, idVendor=1b8e, idProduct=c003, bcdDevice= 0.07
usb 1-1: New USB device strings: Mfr=0, Product=0, SerialNumber=0
```
6. Execute the following commands to enable U-Boot shell at every boot.

**WARNING: This step modifies the *env* partition. Changes are persistent, so it shall be executed only once.**
```shell
./bin/update bulkcmd 'amlmmc env'
./bin/update bulkcmd 'setenv storeargs ${storeargs} run update\;'
./bin/update bulkcmd 'env save'
```
7. Reboot the device by unplugging and re-plugging the USB connection.

After this modification, the device always boots in *USB burning mode* (U-Boot shell over USB) when connected to USB host : you can execute U-Boot commands using the [update](bin/update) tool, but you can't see any output (unless you open the device to connect the UART interface).

*Note: if not connected to USB host, the device continues default boot sequence.*

## Boot kernel from USB to enable ADB access
Once the device in *USB burning mode*, the script [scripts/upload-kernel.sh](scripts/upload-kernel.sh) can upload a Linux kernel image and boot it.
The init-ramdisk includes an *initd* script that starts the ADB server.
System partition is not modified, so this is not persistent.

The ADB interface appears as a new USB device on the host:
```
usb 1-2: new high-speed USB device number 18 using xhci_hcd
usb 1-2: New USB device found, idVendor=18d1, idProduct=4e40, bcdDevice= 2.23
usb 1-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
usb 1-2: Product: Superbird
usb 1-2: Manufacturer: Spotify
usb 1-2: SerialNumber: 123456
```

*Note: There is a script that is intended to be run from a UART shell included in scripts that will enable persistent ADB, but is not reccomended, as it will remove the abillity to OTA update. You can find that script [here](scripts/enable-adb.sh.client).*

# Additional Scripts (for advanced use-cases)

## To Be Executed from U-Boot shell
- [scripts/dump.sh](scripts/dump.sh) : Dump eMMC over USB
- [scripts/uart-shell.sh](scripts/uart-shell.sh) : Modify *env* partition to enable Linux root shell over UART
(**note: Access to UART port requires to open the device**) - see the "Utilizing UART" section below for more information on that.
- [scripts/disable-avb2.sh](scripts/disable-avb2.sh) : Modify *env* partition to disable `AVB2` & `dm-verity`
- [scripts/uboot-continue.sh](scripts/uboot-continue.sh) : Exit U-Boot *USB Burning mode* and continue default boot sequence
- [scripts/hacked-bootlogo.sh](scripts/hacked-bootlogo.sh) : Modify the device's bootlogo to the one shown in the header of this writeup.

## Init-Ramdisk Customizations
- [scripts/extract-cpio.sh](scripts/extract-cpio.sh) : Extract files from system partition dump to create initrd image
- [scripts/pack-initrd.sh](scripts/pack-initrd.sh) : Pack custom initrd image

## To Be Executed from an ADB (root) Shell on Device
- [scripts/enable-adb.sh.client](scripts/enable-adb.sh.client) : Modify the local file-system to start the ADB daemon on each boot. This will remove the abillity to OTA update the device, and there are no factory images yet - *MAKE SURE YOU USE THE DUMP SCRIPT ABOVE BEFORE UTILIZING THIS.* To use, open the script, and copy paste each command into an existing ADB (root) shell, or push to the device (renaming to `.sh`), and execute it.

# Utilizing UART
The UART pin-out is as shown below:
![Car Thing UART Pin-out](https://i.imgur.com/LpP9VgB.jpg)

After hooking up to your UART adapter, you can r

For many developmnet cases, you may want more persistent access to the UART pins, you can remove the sticker on the rear of the device, dissasemble, remove the rear heat-shield, and then file out part of the case, as shown below
![Car Thing UART Setup](https://i.imgur.com/vpUnuvx.jpg)

# Credits
- Frédéric Basse (frederic) & Nolen Johnson (npjohnson): The "exploit", writeup, debugging/developing/theorizing the methodologies used.
- Sean Hoyt (deadman): The awesome hacked-logo image.

# Relevant Device Source Code
- U-Boot: [superbird-uboot](https://github.com/spsgsb/uboot/tree/buildroot-openlinux-201904-g12a)
- GNU/Linux: [superbird-linux](https://github.com/spsgsb/kernel-common)
