# velocloud-hacks

Let's install Debian on the VeloCloud Edge 510

Thanks to `Fabián Rodríguez` for his work on the original [OPNsense guide](https://dulib.re/wiki/doku.php/opnsenseonvelocloudedge510) and `retry` for notes about starting the Debian installer.

## What We'll Need

- A PC to write the installer USB drive
- A USB drive (1GB+)
- A mini-USB cable for serial console access
- An ethernet cable for Internet access

## Flash Coreboot & Disable the Watchdog

### Connect to the Serial Console

Remove the small metal plate on the back of the router to the right of the network ports (small phillips head screw), exposing a mini-USB port. Connect it to our PC and open a terminal emulator at `115200 baud, 8N1`. 

```bash
sudo screen /dev/ttyUSB0 115200
```

Connect a network cable to GE4 on the VeloCloud from any port to our switch, then plug in power to the VeloCloud.

### Login to VeloCloud OS

After power is applied, wait a few minutes for boot and press Enter a few of times until we see the `vc-edge login:` prompt. Log in as `root` with password `VeloHelloXXX` where `XXX` is the last 3 characters of the serial number on the bottom of the device.

If the password doesn't work, reboot and watch for a prompt `press f and ENTER` for fail-safe mode, then run:

```bash
mount_root
echo -e "ourpassword\nourpassword" | passwd root
```

Then reboot and log in.

### Download and Flash Coreboot

We need to replace the UEFI with something open-source that will let us install a different OS, so Coreboot to the rescue!

```bash
cd /root/firmware
wget https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/firmware/2017-4-10-coreboot.rom
cd ..
./dmi-tool -u firmware/2017-4-10-coreboot.rom
./dmi-tool -w -p EDGE510 -v 1
flashrom -p internal -w ./firmware/2017-4-10-coreboot.rom
```

Wait for `VERIFIED` before continuing.

### Disable the Watchdog Timer (CANNOT BE SKIPPED)

```bash
i2cset -y 1 0x24 0x00 0x00
i2cset -y 1 0x24 0x01 0x00
```

## Prepare the Debian Installer USB

Do these steps on our PC.

### Download the Debian netinst ISO

Download the latest Debian amd64 netinst ISO from https://www.debian.org/distrib/netinst

At the time of this writing:

```bash
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.4.0-amd64-netinst.iso
```

### Write the USB Drive

Find the USB drive's device name by running `lsblk` and identify it by size, do not write it to the system disk. Then write the image:


```bash
sudo dd if=debian-*-amd64-netinst.iso of=/dev/sdX bs=4M status=progress
sudo sync
```

Replace `sdX` with the actual USB device.

## Install Debian

Now we're read to install!

### Boot the Edge 510

- Unplug power from the Edge 510
- Plug the USB stick into either USB port
- Plug a network cable from any ethernet port into our network switch
- Plug power back in and watch the serial console launched earlier

### Launch Installation with Boot String

When we see the installer boot menu appear on the serial console, press `Esc`. A `boot:` prompt will appear

At the `boot:` prompt, enter:

```bash
install console=ttyS1,115200n8
```

The Debian installer will start and all output will come through the serial console.

### Perform the Installation

Work through the installer screens. A few notes:

- For the target disk, select the ~8GB eMMC (`/dev/mmcblk0`), not the USB stick.
- For partitioning, use simple ext4 (no LVM/encryption as the drive is so small). We may also want to not use a dedicated swap partition.
- When choosing software to install, select only `SSH server` and `standard system utilities` as we will not have a desktop environment on this system since there is no display.
- You will get a note about missing drivers for the WiFi card. The installer does not have the drivers but the installed system should have them, so ignore this message.
- The installation can be slow in parts. If you see a solid blue screen, wait a few minutes or use arrow keys to safely try to repaint the screen without actually inputting anything.

### Finish and Reboot

Select `Finish the installation` and remove the USB drive when prompted. The installer will reboot the device automatically. 

If you keep the serial session open you can log into the machine from here on boot. Otherwise, SSH in using the device's IP address.

### Sources

- https://dulib.re/wiki/doku.php/opnsenseonvelocloudedge510
