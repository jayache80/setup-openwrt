# Setting up OpenWRT

This document details random steps, thoughts, and notes that I gathered while
putting OpenWRT on router devices in my network.

## Get an image

Lots of common devices have pre-built images available for download at the
[OpenWRT Table of Hardware: Firmware Downloads][1]. There are several
different types of images.

### Officially released images

**"Install" aka `factory`**: Use this type of image if you are flashing the
image over to the device using the OEM firmware flashing utility or via TFTP.
It'd probably be the case that you're putting OpenWRT on this device for the
very first time. Do not use this type of image when trying to do an upgrade of
an existing OpenWRT installation (using OpenWRT's LuCI (GUI) interface or by
running `sysupgrade` in an SSH session); doing so would "soft brick" the
device.

**"Upgrade" aka `sysupgrade`**: Use this type of image if the device is already
running OpenWRT. If you are using OpenWRT's LuCI (GUI) interface to upgrade
OpenWRT (or runnign `sysupgrade` in an SSH session), this is the type of image
to use. Incidentally, a `sysupgrade` will attempt to backup and restore
configuration file changes you've made (but may miss some custom configuration
files or user-installed packages that it doesn't know about). If you try to use
a `sysupgrade` image with the OEM firmware flashing utility (or via TFTP) then
it likely won't get recognized and will just continue to the normal boot.

### Snapshot images

Snapshot images are the latest builds of the OpenWRT codebase. They may come
with newer features and security updates but also have the risk of being less
stable. In some cases, your device may be newly supported by OpenWRT such that
only snapshot images exist until an official version which supports such a
device is released.

Note: Snapshot images do not come with the LuCI (GUI) interface. You have to
install that separately, which can be easily done using the `opkg` package
manager.

I don't like snapshot images because you can only reliably use the `opkg`
package manager to install additional packages for a short period of time
before a new snapshot image comes out at which point all bets are off. This is
contrast to official releases where you can reliably use `opkg` to install
additional packages at any point in time in the future (assuming the servers
are still up and the release is still supported).

**"Snapshot Install" aka `factory`**: See `factory` image notes above.

**"Snapshot Upgrade" aka `sysupgrade`**: See `sysupgrade` image notes above.

## Flash the image

When you download an image, view the "Device Techdata" article associated with
your device, which should be listed along with the various images in the
[OpenWRT Table of Hardware][1]. The "Installation method(s)" will give some
device-specific details about how to flash the image over to the device. For
example, an important detail I learned from there is this: if using TFPT to
flash a factory image to a TP-Link Archer A7 v5, the factory image file name
should be renamed to be "ArcherC7v5_tp_recovery.bin" before the OEM firmware
will accept it.

Assuming you're flashing OpenWRT to a new device for the first time, you'll
want a `factory` image.

### Using OEM GUI

I'm using a TP-Link Archer A7, which is Amazon's rebranding of the TP-Link
Archer C7. The model version of this device is 5.8, as indicated by a sticker
with the serial number on the bottom of the device's case.

Find out the default configuration of the network and IP address of your router
device, and log into it via HTTP (as HTTPS is likely disabled or not
configured). My device's OEM default IP address is 192.168.0.1.

Make sure you have Javascript enabled in your browser for this. You'll probably
be asked to set a password. Once I logged into there, I navigated to Advanced
-> System Tools -> Firmware Upgrade

I tried to use this OEM firmware upgrade GUI utility to upgrade the firmware to
the OpenWRT image. First, I pointed the utility at the originally named
`factory` image file, and then to one that has been renamed per the
installation instructions to "ArcherC7v5_tp_recovery.bin" (even though the file
name shouldn't matter when using the OEM GUI utility). Both of those didn't
work. The GUI kept reporting: "Invalid file type". Maybe you'd have better luck
with a different device, but no joy with the Archer A7 v5. So, I'll be using
TFTP.

### Using TFTP

On many devices, you can boot in "recovery mode" which likely boots U-Boot such
that it looks for a specific file on a TFTP server at a specific IP address
(like a PXE boot). My device (Archer A7 v5) looks for
"ArcherC7v5_tp_recovery.bin".

> Note: It's expected that the name of the image file that your device's
> recovery mode looks for is specific to the device. However, a curious thing
> about the TP-Link Archer A7 recovery mode is that the image must be named
> "Archer**C7**v5_tp_recovery.bin" even though it is an **A7** device.  The A7
> is simply a rebranding of the C7 and is almost an identical device, but it
> *does* have a different flash layout so it's not a perfect clone (meaning you
> can't mix and match A7 and C7 firmware images). Yet, the U-Boot configuration
> must still have the "C7" string baked in. Funny.

Get a TFTP server running and serving up the image. I'm running Arch and
installed the server with:

    sudo pacman -S tftp-hpa

By default, it serves up files from:

    /srv/tftp

Make sure the image file is named correctly and plop it into that directory:

    /srv/tftp
    └── ArcherC7v5_tp_recovery.bin

This particular device will expect the TFTP server to have an IP address of
192.168.0.66 so configure your Ethernet interface to have a static IP address
of that:

    sudo ip addr add 192.168.0.66/24 dev enp0s25

Make sure the Ethernet interface is up. Perhaps explicitly run:

    sudo ip link set enp0s35 up

> Note: `enp0s25` is the Ethernet interface that will be directly connected the
router device which we will be flashing.

It's probably wise to disable other Ethernet interfaces and get off any
Wireless networks during this process. Otherwise routing might be confusing or
incorrect.

Now, start the TFTP daemon:

    systemctl start tftpd

Let's boot the router device into recovery mode which will automatically load
the image via TFTP and flash it to the device.

1. Start with the router off.
1. Connect an Ethernet cable from the Ethernet interface of the computer
   running the TFTP server to the switch section of your router device.
1. With the router device powered off, press and hold down the reset button.
   You may need to use a pencil or some type of pokey thing to press that
   little reset button.
1. While holding down the reset button, power on the router device. Continue to
   hold down the reset button.
1. After 10 seconds, release the reset button.
1. Flashing should take about 1 minute or so. On my device, the power indicator
   blinked continuously during the flash, stopped blinking when it was done,
   and immediately rebooted into OpenWRT.

> Note: The process of booting into recovery mode is specific to each device
> and may differ for other devices.

Once OpenWRT boots, the router's default IP address will be 192.168.1.1/24 and
DHCP will be enabled, handing out addresses on that subnet.

## Configuration

You can do lots of things with the LuCI GUI. However, it's nice (and sometimes
nicer) to do things via SSH (command line) as well.

The default configuration has the Wi-Fi radios *disabled* by default. So
you'll need the Ethernet cable connected until you get that stuff configured.

Navigate to http://192.168.1.1 in your web browser to start configuring things
using the LuCI GUI, or SSH to 192.168.1.1 with the username `root`. By default,
there's no password set so you can SSH straight in (or log into LuCI) without
entering a password.

The following only details using the command line as the GUI should be pretty
self-explanatory.

### SSH in

SSH should be disabled by default. Log in as `root`:

    ssh -l root 192.168.1.1

### Set a password

    passwd

### Enable radios

    uci set wireless.radio0.country='US'
    uci set wireless.radio0.disabled='0'
    uci set wireless.default_radio0.ssid='OpenWRT5'
    uci set wireless.default_radio0.encryption='psk2'
    uci set wireless.default_radio0.key='password'

    uci set wireless.radio1.country='US'
    uci set wireless.radio1.disabled='0'
    uci set wireless.default_radio1.ssid='OpenWRT2.4'
    uci set wireless.default_radio1.encryption='psk2'
    uci set wireless.default_radio1.key='password'

    uci commit wireless
    wifi reload

### The rest

See all the OpenWRT docs for general guidance on how to configure a router. Use
`uci` to configure settings and `opkg` to install additional packages.

### Use existing configs

Pull existing configs from another OpenWRT installation by using the LuCI GUI:

    System -> Backup / Flash Firmware -> Backup: Generate Archive

This will give you a tarball to download. And on the OpenWRT installation to
where you'd like to migrate those configurations:

    System -> Backup / Flash Firmware -> Backup: Upload Archive...

And upload the tarball you previously downloaded.

You may notice that not all your configuration is backed up, especially random
scripts and non-standard config. This is because OpenWRT will only back up what
it knows about (standard config files) and will ignore things that aren't on
its "radar". You can tell OpenWRT about additional files to back up by
appending this file:

    /etc/sysupgrade.conf

You can also add files entries to this file using LuCI:

    System -> Backup / Flash Firmware -> Configuration

Also, back up additional files by adding files (containing file paths to files
that you'd like backed up) to this directory:

    /lib/upgrade/keep.d/

OpenWRT should know how to backup config files associated with
user-installed packages as inspected using this command:

    opkg list-changed-conffiles

However, it may be possible that some config files still aren't captured. So,
verify. Using LuCI, you can see a "rendering" of all the files it will collect:

    System -> Backup / Flash Firmware -> Configuration -> Open list...

Incidentally, this is the method that OpenWRT uses to back up and restore your
configuration when doing a `sysupgrade` (upgrading an existing OpenWRT
installation to a new version). It's important to note that after a
`sysupgrade` you still have to go and install any additional packages you
installed using `opkg` because those do not get saved off in your backup; only
the *configuration* of such packages would get backed up and restored. You
would also need to enable services of such packages if the service doesn't
default to being enabled.

[1]: https://openwrt.org/toh/views/toh_fwdownload
