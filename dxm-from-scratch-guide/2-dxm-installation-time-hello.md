# The Install

## Chroot Inside

We're going to want to enter the environment we just unpacked at `/mnt/gentoo`.
So How do we Enter This Linux Environment other than the one we boot'd from?

chroot is the answer. 

We borrow a few things from the host system, and then trap ourselves in a jail
where in our case `/mnt/gentoo` will appear as the root of our system.

## Show me Your Ren

First thing to do is copy over DNS information from the host linux system,
and make sure its available at `/mnt/gentoo/` (the system we're cooking up).

So go and fetch your resolve,

```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc
```

### Terminal Multiplexing

If tmux is available on your host system you should chroot in a
tmux shell so that you keep tabs on the Host System Environment which is VERY convenient.

For example you could do the following,

```bash
## Create the Session
tmux new -s the-sesh
## Create A Vertical Panel
ctrl-b+Shift-%,
## Create A Horizontal Panel
ctrl-b+Shift-"
## Shift between Panels
ctrl-b+Shift-(up,down,left,right arrows)
## Create a new Window
ctrl-b+c
```

If you forgot to do this later on and regret it,
you can check if there is another open tty on your system you can sign into with `ctrl+alt+F{2,3,.}`

### The Spell

Next We must Enter the Following in order to Bind our Host System to The System Under Brew,

```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```

And then we jump in,

```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="BUU ${PS1}"
```

We Are In.

## Mount Das Boot

Now that we're in the System Under Preparation, let's mount our Boot Partition.

The Raspberry PI 5 Expects our Boot Partition to be at `/boot/firmware` so that
is Where We Will Put It,

```bash
mount --mkdir /dev/<your-boot-device> /boot/firwmare
```

## Sync The Gentoo Ebuild Repository
 
The Komon DEI Repository has a Parent and that Parent is the Gentoo Ebuild Repository.

What Do I mean by an Ebuild?
Just a text file with a `.ebuild` extension that contains information
on how to compile and install package `xyz` at version `a.b.c`.

Why do we need a Parent?
Our Komon DEI Repository barely defines anything.
Most Ebuild information is hosted on the Gentoo Ebuild Repository.

This is kinda why I called it an "overlay" earlier.
Our komon-dei repository *overlays* on the Gentoo parent repository. 
The Gentoo Repo will fill in the gaps of any build information not defined in komon-dei.

### Choosing a Mirror

Before syncing, Let's define Our Mirror.
Our Mirror will determine where we choose to fetch our copies of the source code and ebuild metadata.
Gentoo Mirrors can be found at,

https://www.gentoo.org/downloads/mirrors/

Our Gentoo Mirror is going to be defined at /etc/portage/make.conf relative to our New System.
You can copy it over from komon-dei as,

```bash
cp -v /var/db/repos/komon-dei/komon-config-files/misc-komon-dei-files/make.conf /etc/portage/make.conf
```

Feel Free to Edit to the Mirror of Your Choosing.
Please be careful. A Bunch of Mirrors are Outdated.
The Default is set to OSU Open Source Lab,
But they might be getting DeFunded under the new Administration so I don't know.

```bash
nano /etc/portage/make.conf
```

Now we can Synchronize the Repository,

```bash
emerge-webrsync
```

### Profile Selection

Now that we've synchronized, let's select a Komon Profile.
This Profile will define the Compilation, and Installation of our yet to be Linux System.
You can see your profile options with,

```bash
eselect profile list
```

For Terminal environments please go with one of the `dxm` variants.
For Desktop environments please go with one of the `qxm` variants.

You can then select your profile with

```bash
eselect profile set <number-of-your choosing>
```

## The Kernel

If you remember, the Gentoo Stage File we unpacked previously was pretty much everything minus the

* kernel
* necessary bootloader configuration
* miscellaneous packages 

Well Now We Have to Cook a Kernel.

What is a kernel?

RedHat says,

```text
The Linux kernel is the main component of a Linux operating system and its
core interface between a computer hardware and its processes. It communicates
between the 2, managing resources as efficiently as possible.

The kernel is so named because-like a seed inside a hard shell-it exists within
the OS and controls all the major functions of the hardware, whether it's a 
phone, laptop, server, or any other kind of computer.
```

Basically It's the Interface between User Level Software, and the Underlying Hardware.

### Emerge The Kernel

We'll be using the Raspberry Pi Linux Kernel Branch.
This branch includes Raspberry PI specific patches.
Someone should try Latest Linux one day.

To emerge the Kernel Please Run,

```bash
emerge --ask --verbose sys-kernel/dxm-sources::komon-dei
```

The Kernel Source will then be available at `/usr/src/linux-<some-string>`

We rarely want to directly touch this `/usr/src/linux-<some-string>` directory directly. 
Instead we're going to create a symbolic link from `/usr/src/linux-<some-string>` to `/usr/src/linux`.

This might look something like this,

```bash
ln -sf /usr/src/linux-6.12.30_p20250601-dxm /usr/src/linux
```

Please be very careful with ending slashes when creating the Link.
Why create this Link? 
  * Because One Day when it comes time to update our Kernel, 
  we'll mostly just have to switch where that `/usr/src/linux` is pointing to.

### The Colonel's Recipe

Navigate over to `/usr/src/linux`, and it's time to start cooking.

The kernel configuration is saved within a file in `/usr/src/linux` called `.config`.
If you want the defaults for the Raspberry PI's bcm2712 processor you can run,

```bash
make bcm2712_defconfig
```

And that'll give you a `.config` in `/usr/src/linux`.

I also have a kernel recipe prepared available within the komon-dei repository
at `komon-dei/komon-config-files/linux-config/6.12.30/<desktop-config-v1 | server-config-v1>` which can be
copied over with a name change,

```bash
cp -v
/var/db/repos/komon-dei/komon-config-files/linux-config/6.12.30/<desktop-config-v1 | server-config-v1>
/usr/src/linux/.config
```

This Recipe is really just a bcm2712_config base with a few 'lil Tweek Tweaks.
A Description of the Colonels Recipe is available at Manny Manuals.

Any recipe, you choose you can see and or modify the settings by running `make menuconfig` from within `/usr/src/linux`.

### Making the Kernel and Installing

The kernel can now be compiled with the `make` command.

When running `make` we can specify how many processors with the -j flag.
Remember the Raspberry PI 5 has 4 cores, and we don't wanna overwhelm the system.
I suggest -j2 or -j3.
I personally usually do -j4 but I wouldn't recommend it.

```bash
make -j3
```

We can then install modules with,

```bash
make modules_install -j3
```

### Kernel In Boot

Okay so now that we cooked our kernel, we can copy it over to our boot partition.
Copy over the resultant kernel and rename it,

```bash
cp -v /usr/src/linux/arch/arm64/boot/Image /boot/firmware/dxm-kernel
```

### Primer on Device Trees, and Overlays

Raspberry Pi Firmware uses a Device Tree (DT) to describe hardware.
A Device Tree represents the hardware configuration as a hierarchy of nodes.
Device Trees are usually written in a textual form known as Device Tree Source
(DTS), stored in files with `.dts` suffixes. The compiled `.dts` file results
in a Device Tree Blob (DTB), and is stored in `.dtb` files.

What are Device Trees Overlays?

We modify the .dtb by applying optional components onto the base Device Tree
through overlays. These are the `.dtbo` files

### Copying Over Device Tree and Overlays

The Device Tree for the bcm2712 and the corresponding overlays can be
copied to the boot partition at `/boot/firmware` over from `/usr/src/linux`.

Let's first copy over the compiled Device Tree Blobs,

```bash
cp -v /usr/src/linux/arch/arm64/boot/dts/broadcom/bcm2712-rpi-5-b.dtb /boot/firmware

cp -v /usr/src/linux/arch/arm64/boot/dts/broadcom/bcm2712d0-rpi-5-b.dtb /boot/firmware
```

We copy over two device tree blobs: one for the BCM2712, and another for the BCM2712D0 variant (or stepping). 
The D0 stepping has a different layout and some more free silicon so they need different Device Tree Blobs.

Then we have to copy over the overlays.
Lets create the folder in /boot/firmware that the bootloader expects and then copy the files over,

```bash
mkdir /boot/firmware/overlays
cp -vR /usr/src/linux/arch/arm64/boot/dts/overlays/*.dtbo /boot/firmware/overlays
```

You probably want to copy the README over as well,

```bash
cp -vR  /usr/src/linux/arch/arm64/boot/dts/overlays/README /boot/firmware/overlays
```

## Pre World Configuration

### Configure Locale

You're going to want to configure your locale before installing the @world.

Let us modify `/etc/locale.gen`.
Uncomment the ISO, and UTF Character Format for your selected language.
For example,

```bash
en_US ISO-8859-1
en_US.UTF-8 UTF-8
```

Then Generate With,

```bash
locale-gen
```

Good. Now we can run eselect locale list, and set our chosen locale with,

```bash
eselect locale set <num>
```

You're generally going to want to set the UTF-8 Character Set.
Please reload the environment for the changes to take effect,

```bash
env-update && source /etc/profile && export PS1="FLEEFLY ${PS1}"
```

### Set the Hardware Clock

Please edit the file found at /etc/conf.d/hwclock accordingly,

```bash
# Set CLOCK to "UTC" if your Hardware Clock is set to UTC (also known as
# Greenwich Mean Time).  If that clock is set to the local time, then
# set CLOCK to "local".  Note that if you dual boot with Windows, then
# you should set it to "local".
clock="local"
```

### Set the KeyMaps

```bash
nano /etc/conf.d/keymaps
```

Modify as follows for English,

```bash
# Use keymap to specify the default console keymap.  There is a complete tree
# of keymaps in /usr/share/keymaps to choose from.
keymap="us"
```

## Installing The World

Alright.
It's time to install the World.
Please Remember, Gentoo is a source based distribution so we'll be compiling almost everything including
The GCC from scratch.

We can install the world with,

```bash
emerge --ask --verbose --update --deep --newuse @world
```

If you want a "mostly" offline experience you can, add `--fetchonly` to download the source before compiling.
This will allow you to unplug after Source Code Fetching.
The install will be interrupted by some 9999 (live ebuilds) which did not download initially,
but the installation can then be resumed after plugging back in momentarily to let the 9999 in.

```bash
emerge --ask --vebose --update --deep --fetchonly @world
emerge --ask --verbose --update --deep --newuse @world
emerge --resume #(plug in/unplug when necessary)
```

This'll take a couple days.
It might be useful (and necessary to setup some external swap here).

I like to keep htop and nethogs running in a separate terminal session for monitoring.

Make sure to keep a keyboard connected and press a few keys every once
in a while or wiggle a mouse so she gets some I/O and knows you're there.

## Configuring The Boot Partition

It's come time to configure the boot partition located at `/boot/firmware`.
The Raspberry PI is going to expect certain files in there such as that kernel
we cooked earlier, and a `config.txt`.

The komon-dei repository contains a couple of these files so that they can be easily copied over.
These files are available at `komon-config-files/boot-partition-files`.

### Licenses

We have to copy over the Broadcom, and Linux Licenses over to /boot/firmware since not doing so wouldn't be right.

```bash
cp -v komon-config-files/boot-partition-files/licenses/COPYING.linux /boot/firmware
cp -v komon-config-files/boot-partition-files/licenses/LICENCE.broadcom /boot/firmware
```

### Issues

Need em. 
issues.txt is a small file with whatever you want.
Edit it as you like or just copy it over directly,

```bash
cp -v komon-config-files/boot-partition-files/issue.txt /boot/firmware
```

### Config.txt

Next, we're going to want to bring the config.txt over.

Instead of the BIOS found on a conventional PC, Raspberry PI 5 has a confg.txt.
The config.txt does things like set the processor, GPU Frequency, and 
what's most importantly for us which kernel to boot from and which initramfs to use.

komon-dei provides an opinionated config.txt available over at
`komon-config-files/boot-partition-files/config.txt`. 
This config.txt disables bluetooth, wifi, unused interfaces to the PI,
slightly overclocks the GPU and Processor, and some miscellaneous things.

Please Feel Free to Review and Change it as you like.
If a name different than `dxm-kernel`, was chosen for the kernel, the config.txt should be modified as well.

Please copy it over and Edit Accordingly as per the Kernel and InitRamFs Name.
We'll be generating the initramfs in a little bit, which the help of the Drac.

```bash
cp -v komon-config-files/boot-partition-files/config.txt /boot/firmware/config.txt
```

### Cmdline.txt

When the system boots up, it's gonna look for a `cmdline.txt` for instructions
on things like where the root partition is.

This file is going to need some modification as we need to specify the partition UUID for our Root Drive.
First copy the file over,

```bash
cp -v komon-config-files/boot-partition-files/cmdline.txt
```

Get the UUID of the Root Partition with `lsblk -f`.

And then you're going to want to replace the root UUID within `cmdline.txt`, with your own UUID.
An example, might look like:

```bash
root=UUID=<YOUR-UUID-HERE>
```

A quick and dirty tip might be to 
* append the UUID to the cmdline`lsblk -f /dev/<your-device> >> cmdline.txt`,
* copy and paste the UUID of your choice into `root=UUID=<YOUR-UUID-HERE>` with the editor of your choosing 
* delete the pasted segment remembering to leave the file with a newline at the end so that it comes out nice.

## Configuring the fstab

So the root partition should be mounted at `/`, and the boot partition should be mounted at `/boot/firmware`.
But, how would the system know? 
Well it needs to check the fstab.
The fstab is going to specify which partitions with which UUIDs get mounted where.
The fstab will tell the system: 
  * "mount partition at UUID=X with given type at the specified mount point with the given options".

komon-dei includes a premade fstab, that can be copied over to /etc/fstab

```bash
cp -v komon-dei/komon-config-files/misc-komon-dei-files/fstab /etc/fstab
```

Just, as with the `cmdline.txt` we're going to want to modify the UUIDs within the fstab.

It should look something like this,

``` bash
UUID=<Your-boot-partition-uuid-here>    
## Below this refers to the opened LUKS device
UUID=<uuid-of-dev-mapper-dxm-partition>
```

again, these UUIDS can be found with,

```bash
lsblk -f
```

## The Initial File System

We're going to need something called an initramfs.

What is an initramfs? Well since we have an encrypted drive, we kinda have to
open it before the system can mount it.
If you think about it, we did this earlier with `luks open /dev/mmclbk0p2 gentoo`.

Now we're going to need an initial filesystem, from which we can open the drive.
An initramfs is exactly such a file system.
After the drive is opened, the root file system as specified in the `cmdline.txt`  can be mounted as per the fstab.

We're going to be using the tool `dracut` which will give us a minimal file system. 
Now we already installed dracut during the @world update but we still need to configure it.

komon-dei gives you such a configuration file at `komon-dei/komon-config-files/misc-komon-dei-files/dracut.conf`.

This file can be copied over to `/etc` which is where it belongs,

```bash
cp -v komon-dei/komon-config-files/misc-komon-dei-files/dracut.conf /etc/dracut.conf
```

Now this file too will have to be modified to point to the correct UUIDS.
`dracut.conf` will contain a line like,

```bash
kernel_cmdline+=" root=UUID=<uuid-of-dev-mapper-dxm-partition-here> 
rd.luks.uuid=<uuid-of-unopened-luks-drive-here> "
```

Please replace the placeholder values, as done with the cmdline.txt and fstab.

Next the initramfs can be generated,

```bash
dracut --force --kver <kernel-version> initramfs-dxm
cp -v initramfs-dxm /boot/firmware/initramfs-dxm
```

where,
  * kernel-version - This is the name of the directory over at /lib/modules/.
                     You can check and copy over.

Please modify the config.txt if a name other than initramfs-dxm is chosen.

## Setting Init Services

Our init system of choice is OpenRC.
OpenRC and its Daemons will, 
  * start necessary system services in the correct order at boot
  * manage them while the system is in use,
  * stop them at shutdown.

We will be adding services to our init service so that they automatically start when needed.

In my opinion it's kinda nice seeing what's running on my computer and being
explicit about the services rather than guessing which random application is sending out data
who knows where.

The programs described below are already installed via our profile.
We're just going to be adding them to the appropriate level so that our init
service OpenRC starts them up appropriately.

### DHCP

We need a DHCP service on our system.

What is DHCP? 
When we connect an ethernet cable to the Deus Ex Machina we get an IP Address on the Local Area Network (LAN).
Your Router and the Deus Ex Machina need to negotiate this.
This is called DHCP, and will be handled on our system via the dhcpcd deamon.

Let's start dhcpcd and add it to the default runlevel with,

```bash
rc-update add dhcpcd default
rc-service dhcpcd start
```

### System Logger

We're going to have Logs.
We will be utilizing the sysklogd system logger utility which was compiled and installed during the @world update.
System Logs will be available over at `/var/log/syslog`.
From time to time, you find me running `tail -f /var/log/syslog`.

Let's add the sysklogd deamon to the default runlevel with,

```bash
rc-update add sysklogd default
```

### Deamon

We're going to need a cron daemon. 
This deamon will execute commands on scheduled intervals,
and can be used for housecleaning tasks around the system and such.
Our choice of Cron deamon will be Cronie.

Let's add cronie to the default runlevel,

```bash
rc-update add cronie default
```

### Chrony

I skip this.
I don't trust the clocks like that.

### Nftables

So Nftables will be our firewall of choice.

Nftables is a subsystem of the Linux Kernel providing filtering and
classification of network packets.
nftables configuration will be held in /etc/nftables.conf

komon-dei provides `bigiron` which is a simple Nftables firewall configuration
that provides some very basic protections while being very restrictive and
only allowing HTTPS. Please modify nftables.conf as you wish.

Let's first copy it over from komon-dei

```bash
cp -v komon-dei/komon-config-files/misc-komon-dei-files/nftables.conf 
/etc/nftables.conf
```

and then there's one more file we need to modify. The Nftables daemon needs
to know from where to pick up the configuration.
There is a file at `/etc/conf.d/nftables`, and within that file there is a
variable called `NFTABLES_SAVE`. Please modify it as such,

```bash
NFTABLES_SAVE="/etc/nftables.conf"
```

and then add the daemon,

```bash
rc-update add nftables default
rc-service nftables start
```

### PCSC Daemon

In order to get smart card functionality for things like Yubikeys, we need a pcsc daemon.
On our system this is provided by `pcsc-lite`.

`pcsc-lite` was already installed during the @world update, but we still need to add the daemon to OpenRC. 
We will be adding the daemon with Hotplug support, so that pcscd only starts when we plug in an appropriate card reader.

Please append the following to `/etc/rc.conf`,

```bash
rc_hotplug="pcscd"
```

### Modifying the Inittab

The default inittab on Gentoo has this annoying `f0` that keeps respawing.
Please disable it over at /etc/inittab,

```bash
# Architecture specific features
#f0:12345:respawn:/sbin/agetty 9600 ttyAMA0 vt100
```

or you can copy it over from komon-dei

```bash
cp -v komon-dei/komon-config-files/misc-komon-dei-files/inittab /etc/inittab
```

## User Setup

We're going to want to setup at least two users.
* root: superuser with all available privledges on the Linux Machine
* manny: I am manny

### Root Setup

Let's give the root user a password,

```bash
passwd root
```

To prevent threat actors from logging in as root, you might want to delete the root password and disable login. 
Just make sure you do if after you 
make another user with pseudo-root privileges or you'll get locked out unless
you chroot back in and fix it.

```bash
passwd -l root
```

### Miscellaneous User Setup

Let's create a user called manny, and give them a password,

```bash
useradd -m -G users,wheel,plugdev manny
passwd manny
```

The plugdev group is necessary for FIDO2 functionality.

Please make sure to add the user to the sudoers file, in order to grant the user pseudo-root permissions.
The Sudoers file can be modified with,

```bash
visudo
```

## Configure Time

Okay so we have to,

* set the timezone
* set the date and time
* sync the hardware clock from the system clock.

so like this,

```bash
ln -sf /usr/share/zoneInfo/America/New_York /etc/localtime
date -s "2025-05-05 00:00:00"
hwclock --systohc
```

## Set the Hostname

Make absolutely sure your run, 

```bash
echo "dxm-by-em-is-the-best" > /etc/hostname
```

## Enabling Wireguard

Place your wireguard configuration over at `/etc/wireguard/wg0.conf`
Make sure to add the wireguard server to the allowed IPs. 

For example,

```bash
0.0.0.0/0, ::/0, 192.168.2.1/32
```

and then as sudo run,

```bash
sudo wg-quick up wg0
```

## Desktop Profiles

WE will be skipping installation of a display manager because they all suck.
If you want to start a desktop session please run,

```bash
dbus-run-session startplasma-wayland
```

### Jail

You might want to put everything in jail.
This can be accomplished by running

```bash
sudo firecfg
```

This will sandbox all programs for which firejail has a profile for (including firefox) which is nice.

Profiles and their associated Sandbox Rules can be viewed and changed over at /etc/firejail.

If you're going to be using smartcards in browser, the default sandbox won't allow it.
In that case, we will accept a little hole over at `/etc/firejail/firejail.config`
and change the default from yes, to no. 

```bash
browser-disable-u2f no
```

You can also link just firefox emanuelly with,

```bash
ln -sf /usr/bin/firejail /usr/local/bin/firefox
```

you can check the status with

```bash
firejail --tree
```
### GPG Settings

komon dei also offers some opionanted gpg settings over at
`komon-dei/komon-config-files/misc-linux-files/gpg/`.
If you're using smartcards, you're going to want to go ahead
and especially pat attention to that line in scdaemon.conf,

```bash
disable-ccid
```
It'll conflict with the pcsc-lite Daemon we're using to hold PCSC Context.

### Git Settings
Jealous? No Worries My Friend. I Understand Completely.
You too can be me.

Just copy the .gitconfig over from
`komon-dei/komon-config-files/misc-linux-files/gpg/` and saddle up with a fifth.

### Yubikey Manager

If you're using Yubikey, yubikey manager comes installed for convenience,

```bash
ykman info
```
should just work.
