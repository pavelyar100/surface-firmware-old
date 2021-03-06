# Firmware updates for linux-surface

Usually if you want to install firmware updates on your surface, you *have* to 
reinstall Windows or dualboot it. There is no way to install firmware updates
from within Linux.

However, there is a tool called `fwupd`. It is a daemon that is used together
with the [LVFS](https://fwupd.org) (Linux Vendor Firmware Server). Of course
Microsoft isn't part of the LVFS, but we can use the fwupd tool manually.

Microsoft has packaged their firmware updates as UEFI Capsules, which is a 
technology that `fwupd` already implements. The only thing that needs to be 
done is convert them into a format that `fwupd` can read. Again, it comes handy
that `fwupd` already supports the format that is used by Windows Update (to make
it easier for vendors to support LVFS, I assume). The only thing that needs to 
be added is a small file that contains `fwupd` specific metadata.

**This is not related to Microsoft or the LVFS in any way.**

## Receiving firmware updates through fwupd automatically
I have setup a remote for `fwupd`, that you can add to your system to recieive
firmware updates automatically (either through the commandline or a `fwupd`
frontend like GNOME Software). To install it, run these commands:

**You are flashing your UEFI firmware on an extremely locked down UEFI.
I am not responsible for damages to your hardware. I don't know if a recovery
mode exists. Please be careful!**

```bash
# Add my gpg key to fwupd.
$ sudo wget -O /etc/pki/fwupd-metadata/GPG-KEY-Linux-Surface-Firmware https://tmsp.io/fs/repos/fwupd/linux-surface/key.asc

# Add the remote config to fwupd
$ sudo wget -O /etc/fwupd/remotes.d/linux-surface.conf https://tmsp.io/fs/repos/fwupd/linux-surface/linux-surface.conf

# List available updates
$ fwupdmgr get-updates
```

In case you don't trust the automatic update you can also download the `.cab`
files [manually](https://tmsp.io/fs/repos/fwupd/linux-surface/firmware) and
install them yourself.

## How to get the firmware files
Ironically, an out of the box Windows installation has about as much support for
surface as Linux: No battery stats, no touchscreen etc. Microsoft provides a 
gigantic driver package, that also contains the latest firmware. You can 
dowload it here: https://support.microsoft.com/en-us/help/4023482/surface-download-drivers-and-firmware-for-surface

I downloaded the one for Surface Book 2, because thats what I have. I will try
to add the other ones gradually, but since I can't test them, contributions are
welcome!

When you downloaded the MSI file, you need to extract it first. You need to 
install the program `msiextract`

```bash
# For APT based systems
$ sudo apt install msitools

# For Arch based systems
$ git clone https://aur.archlinux.org/msitools.git
$ cd msitools
$ makepkg -si

# alternatively
$ yay -S msitools

# For Fedora
$ sudo dnf install msitools
```

Then extract the MSI you downloaded

```bash
$ msiextract SurfaceBook2_Win10_18362_1904009_0.msi
```

Inside the extracted folder, there will be a `Firmware` folder. You have to 
integrate the folder into the `firmware` folder of this repository, like I did
for SB2 (i.e. make a folder for your model, and then copy the contents into that).

## Writing metdata
First, you should read the official guide from LVFS about the metadata files:
https://fwupd.org/lvfs/docs/metainfo

For the most part you can simply copy the examples, or the files that already
exist.

The hardware GUID, the version number and the version timestamp can be scraped 
from the `.inf` file inside of the firmware directory.

```ini
DriverVer=04/18/2019,389.2706.768.0
```

First entry is the date when the driver was released, you have to convert that
date into an unix timestamp, and then enter it into the metadata file. The
second number might appear like a version number, and it is one, but it is not
the one that gets read by `fwupd` for checking if the version is already
installed. That number is on another line, encoded as a hexadecimal number.

```ini
[Firmware_AddReg]
HKR,,FirmwareVersion,%REG_DWORD%,0x616A4B00
```

`fwupd` has some logic to interpret that number as a triplet version number,
which I replicated in the `getver.sh` script:

```bash
$ ./getver.sh 0x616A4B00
97.106.19200
```

This is the version number you have to enter in the metadata file, like this:

```xml
<release version="97.106.19200" timestamp="1555538400"></release>
```

For consistency, this is the only place where that number is used. The changelog
contains Microsofts version number, so you can search for it on the official
website.

Now, look for something that looks like this:

```inf
[Firmware_AddReg]
HKR,,FirmwareId,,{6726B589-D1DE-4F26-B2D7-7AC953210D39}
```

The text between the two curly brackets is the hardware GUID for the firmware.
You need to convert it into lowercase (https://convertcase.net helps with that),
and then put it here:

```xml
<firmware type="flashed">6726b589-d1de-4f26-b2d7-7ac953210d39</firmware>
```

This allows fwupd to tell you if a package can be installed on your hardware, or
not.

Make sure to update the category in the metadata file, depending on what is being
installed: `X-System` for UEFI updates, `X-ManagementEngine` for Intel ME updates
and `X-Device` for everything else.

## Building
When you are done, invoke the `bundle.sh` script with the path of the firmware
you want to bundle for `fwupd`. For example, to bundle all files for SB2 at the
same time, you would do this:

```bash
$ ./bundle.sh firmware/SB2/*
```

The final bundles will be saved inside of the folder `out`. For details on how
to build them manually, please look at the bundle script, or just ping me on 
[Gitter](https://gitter.im/linux-surface)

## Installing the firmware bundles manually
The bundle script will produce one `.cab` file per firmware update. You can 
install them with `fwupdmgr`, the commandline interface for `fwupd`. `fwupd` 
should come preinstalled with Ubuntu and Fedora, if not please install it.
Arch users should consult the Arch Wiki entry about `fwupd`.

**You are flashing your UEFI firmware on an extremely locked down UEFI. 
I am not responsible for damages to your hardware. I don't know if a recovery
mode exists. Please be careful!**

```bash
$ fwupdmgr install <path/to/cab/file>
```

## Driver Firmware
As part of my adventures I also took the opportunity to update the driver firmware
that jakedays setup script installs to the latest versions from the MS driver 
package / the linux-firmware repository. You can find it in the `driver` folder.
