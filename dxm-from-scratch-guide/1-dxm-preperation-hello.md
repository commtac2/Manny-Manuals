# The Prep

## Prerequisites

The following is required

1.  Raspberry PI 5

    The Older Raspberry PIs are great, but the komon-dei repository provided
    is cooked specifically for the Raspberry PI 5 architecture.

2.  Existing Linux System on A Drive

    We're Going to need a Linux system from which we can boot, and run commands.
    We'll be preparing our new environment from this environment.

    You can use an existing Deus Ex Machina you have laying around, but if not
    A good start here might be the Raspberry PI Official Operating System.
    The Raspberry PI Imager is available for download at https://www.raspberrpi.com/software,
    with instructions at https://www.raspberrypi.com/documentation/computers/getting-started.html#raspberry-pi-imager.

    Burn the Raspberry PI OS onto a drive, plug it in, follow the onscreen setup, and you have yourself a Linux System. 
    But we're about to make an even better one.

3.  A Fresh Drive for DxM Installation

    This can be a USB drive, an SD card, or an NVME drive if an M2 Hat is available.
    I recommend having a spare drive for backup of files you actually care about because who 
    knows when that SD Card is going to give.

4.  A USB Keyboard

    For Typing Purposes. And A Mouse if you want a desktop profile.

5.  A Raspberry PI Real Time Clock Battery

    So the Raspberry PI 5 does have an Internal Quartz Crystal that's capable of keeping time.
    The problem is that there's no internal battery to keep the clock running while there's no power.
    Raspberry PI 5 sells these little RTC Batteries (SC1163) clocks that serve this purpose.

## Overview

Hello.

In this Section we're going to be doing some Prep on our drives before Cooking.

Here is an Overview of the steps we're going to be taking in this section,

1. First, we're going to have to wipe our drive and make sure it's clean.
2. Second, we're going to partition our drive into discrete memory blocks.
3. Third, we'll create the file systems on the drive.
4. Fourth, we'll mount the root partition.
5. Fifth, we'll download and unpack the Gentoo system onto the Root File System we just created
6. Sixth, we'll download and copy over our komon-dei repository to the newly unpacked system.

## Identifying Partitions and Partition UUIDs

Alright so first I guess we have to prepare the drive.

When we plug in the SD Card, or USB into the Raspberry PI we get a unique identifier for the drive.

You can check with,

```bash
lsblk
```

Identify your drive (maybe by size) and there should be a unique identifier
given by the system like `/dev/sda`, or `/dev/mmcblk0`.
When we create additional partitions they will also appear in `lsblk` as
`/dev/sda1`, `/dev/sda2`, or `/dev/mmcblk0p1`, `/dev/mmcblk0p2`. 
These identifiers will be used below when preparing the drive.

Let's assume an SD Card, so `/dev/mmcblk` will be used for the rest of this guide.

Each Partition has a Universal Unique Identifier (UUID) associated.
The UUIDs for each partition can be seen with,

```bash
lsblk -f
```

These UUIDs will be relevant later when setting up fstab, dracut, and the cmdline.txt.

## Wiping the Drive

Now I like to start with a nice clean medium.

Be **very** careful as these are destructive operations that will wipe your drive. 
Make especially sure you choose the right drive please.
Any Data on the selected Drive will be absolutely IRRECOVERABLE.
So Please be careful.

## Testing for Bad Blocks

First you probably wanna check for Bad or Corrupt Memory Blocks.
This can be be done with,

```bash
badblocks -wsv -t random /dev/<your-device-here>
```

The above command will destructively fill your device with a random pattern,
read it back, and make sure every block is good by verifying 
what it read is what it wrote.

## Filling with Random Data

I like to fill up MY drives with Random Data just to make sure she's clean.
This can be accomplished with,

```bash
dd if=/dev/random of=/dev/<your-device-here> bs=4M status=progress
```

This reads as,

```text
dump from Input File /dev/random into Output File /dev/<your-device-here>
with a block size of 4M and I wanna see the status while you do it.
```

Alternatively, you can fill it up with zeroes by running,

```bash
dd if=/dev/zero of=/dev/<your-device-here> bs=4M status=progress
```

### Small Note on Entropy and Source of Randomness

I'm a Mechanical Engineer By Education so let's talk Entropy.

Above we sourced our random numbers from `/dev/random`.
What is /dev/random? 

It's a Special File that spits out a stream of Cryptographically Secure PseudoRandom numbers seeded with Entropy. 
Where does this Entropy Come From? 
The Environment.
A mouse wiggle, a keyboard stroke, the roaring fans, etc.
There's only so much entropy so an entropy "pool" was kept.
I pity the Country, I pity the State, and the Mind of the Man, type Shit.

Back in the day /dev/random was different from /dev/urandom as /dev/urandom  would **NOT** block when the entropy pool was running low.
Nowadays I think they're the same.
Whenever possible we use /dev/random not for what it is, but for what it represents.

If you ever wanna do something like print random characters you could,

```bash
head -c 1000 /dev/random | base64
```

### Generating Your Own Random Numbers (Only For Fun)

Generating Cryptographically Secure PsuedoRandom Numbers is hard.
Let's Start with a NON Cryptographically secure Pseudo-Random Number Generator.

A Linear Congruential Generator is Classic and will give you Uniformly Distributed Numbers between 0 and 1 given a Seed.

The below Python Script shows an example,

```python
from collections.abc import Generator

## Parameters from Numerical Recipes but do as you wish
def lcg(seed, modulus = 2**32, a = 1664525, c = 1013904223):
    while True:
        seed = (a*seed + c) % modulus
        yield seed/modulus

if __name__ == "__main__":
    mySeed = 217
    mine = lcg(mySeed)
    print(next(mine))
    print(next(mine))
    print(next(mine))
    print(next(mine))
```

If you have numbers between 0, and 1 and some Monte Carlo Luck you can rule the world.
Or generate almost any distribution (https://en.wikipedia.org/wiki/Inverse_transform_sampling).
Now Let's Anneal.

## Formatting the Partition

Now we have to partition the drive.
All A partition really is, is a defined block of memory on the drive with some additional metadata like type.

DOS/GPT refer to the layout of the partition table we're going to be creating.
First thing to note here is that we're going to go with the older DOS style partition table instead of GPT. 

Why DOS?

The Raspberry Pi 5 builds the Bootloader into the EEPROM.
That Bootloader only accepts DOS, and no GPT formatted drives. 
What this means for us in practice is no drives over 2TB, and up to four primary partitions.
People have reported success with GPT drives on the Raspberry PI 5 but I don't know and I don't trust it. 
Might as well do things right.

Two Partitions will be created on the drive.

1. The First partition will be the Boot partition with type `W95 FAT32 (LBA)` and 256M of memory.
2. The Second partition will be the Root partition with type `Linux` and its size will take up the rest of the drive.

The boot partition will be of type `0c` in hexadecimal, while the root partition will be of type `83`.
More partitions/different scheme can be created if you want to venture off.
For example LVM is an option too.

In order to format the drive, as described above start off with,

```bash
fdisk /dev/<My-Drive>
```

And enter the prompts accordingly.
Your sequence of commands might look something like,

```bash
## Create a new DOS Table
o
## Create A new primary partition from starting 
## block onwards for 256M worth of blocks.
n, p, <leave-blank/enter>, +256M 
## Create A new primary partition from last unused block
## on until the last block.
n, p, <leave-blank/enter>, <leave-blank/enter>
## Change The Type Of The Boot Partition to 0c (W95 FAT32 (LBA))
t, 1, 0c
## Change The Type Of The Root Partition to Linux
t, 1, 83
## Print To Make sure Everything looks ok
p
## Actually Write the proposed table to the drive
w
## Or ReDo it over at anytime with q and quit. h to display help text.
```

You should get a partition scheme like this if we print within fdisk,

```bash
Device         Boot  Start       End   Sectors   Size Id Type
/dev/mmcblk0p1        2048    526335    524288   256M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      526336 249737215 249210880 118.8G 83 Linux
```

If you check `lsblk` again, as done previously, you should now see two entries
under the original device file `/dev/<your-device>.`

Please note that making the boot partition type `0c` is **very important**.
The Raspberry Pi 5 will refuse to boot off anything other than W95 FAT32 (LBA).

## Encrypting the Root Partition File System

I Don't like people knowing my stuff and encrypting the root file system is a very good start.

In order to encrypt we'll be using LUKS.
LUKS Specifications and Details here if Curious: https://gitlab.com/cryptsetup/-/wikis/LUKS-standard/on-disk-format.pdf

In order to see which encryption ciphers are available run

```bash
cryptsetup benchmark
```

Modern Processors are Highly Optimized for AES and Symmetric Ciphers with good key lengths are still in theory HIGHLY
unbreakable even by Quantum Computers according to Published Works.
But it's also Deeply Studied by the NSA so tu nunca sabe.

The Drive can then be encrypted with a passphrase as follows,

```bash
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512
--hash sha512 --pbkdf argon2id --pbkdf-memory 1048576 --pbkdf-parallel 2
--verify-passphrase --use-random /dev/<root-partition>
```

The above command will,

* Encrypt the root partition using the AES Cipher with a 512 bit key size.
* Use the SHA512 hash where Appropriate.
* Ensure the security of the drive will be kept with a chosen password,
  and scrambled with the argon2id password based key derivation function (pbkdf)
  which we'll allocate 1G of memory and 2 cores for the operation. 
  Cryptsetup will correspondingly approximate an appropriate number of rounds for the pbkdf.
* Use /dev/random for your random numbers.
* Verify the given passphrase before committing to it.

Please note, that even though we use a 512 bit key, it'll functionally be 256 bits.
Key size for XTS mode is twice that for other modes for the same security level.

An Encrypted Keyfile can also be used instead of a PassKey, but that's a story for another day.

A ***very*** import point is to backup the header.
If that Header gets damaged in any ways that drive is absolutely not recoverable.

```bash
cryptsetup luksHeaderBackup /dev/<Device> --header-backup-file save-me
```

The Drive can then be Opened and Mapped to the Name `dxm` with,

```bash
cryptsetup open /dev/<Device> dxm
```

Which will create a handle to the device as `/dev/mapper/dxm`.
For the remainder of this guide we will refer to this device by this `/dev/mapper/dxm` handle
instead of say `/dev/mmcblk0p2`.

## Creating The FileSystems

Now sitting before us, are just formatted blocks of memory with no FileSystems.
Let's create the following FileSystems:

* Boot Partition - For Boot we will use a Very FAT 32 Filesystem
* Root Partition - For Root Partition use ext4 FileSystem as its expected by the PI 5 Bootloader.

Something Like the Following Commands will do it

```bash
mkfs.vfat -F 32 /dev/mmcblk0p1
mkfs.ext4 /dev/mapper/dxm
```

## Mounting The Root Partition

So Now we Have the Drives Partitioned, and Both Partitions have FileSystems.
But is the File System Available For the System to View?
No.
Not until we Mount it.

To mount the root partition at the `/mnt/gentoo` mount point something like this,

```bash
mount --mkdir /dev/mapper/dxm /mnt/gentoo
```

Now the contents of the FileSystem should be available at `/mnt/gentoo`.
It Should be Empty Save for a lost+found directory.
Let's Create a File, just so that they know to return it to me,

```bash
echo "Me Em" | sudo tee /mnt/gentoo/lost+found/return-to > /dev/null
```

## Installing the Stage 3 Tarball

### What is A Stage File and Where Do I Get It?

A Stage File, also known as a Stage Tarball, is an archive containing a minimal Gentoo environment. 
A Stage 3 Tarball contain an almost complete Gentoo system minus, 
* kernel
* bootloader files 
* our selected packages.

Stage 3 Tarballs are available at https://distfiles.gentoo.org/releases/arm64/autobuilds

For example, 

```text
https://distfiles.gentoo.org/releases/arm64/autobuilds/current-stage3-
arm64-openrc/stage3-arm64-openrc-20250323T232020Z.tar.xz
```

Points to the `stage3-arm64-openrc-20250323T232020Z.tar.xz` Stage 3 Tarball with a Timestamp of `20250323T232020Z` which reads as,

> March 23 2025 at 23:20.20 UTC Time.

Please note that the corresponding `.sha256`, and `.asc` files as well.
The `.sha256` will be used to verify that the contents of the file were unaltered on download.
The `.asc` will be used to verify that the file came from who we're expecting.

After updating the above link with the latest timestamp (check https://distfiles.gentoo.org/releases/arm64/autobuilds),
downloading these files might look like:

```bash
wget 
https://distfiles.gentoo.org/releases/arm64/autobuilds/current-stage3-arm64-
openrc/stage3-arm64-openrc-20250323T232020Z.tar.xz

wget https://distfiles.gentoo.org/releases/arm64/autobuilds/current-stage3-
arm64-openrc/stage3-arm64-openrc-20250323T232020Z.tar.xz.sha256

wget https://distfiles.gentoo.org/releases/arm64/autobuilds/current-stage3-arm64-openrc/stage3-
arm64-openrc-20250323T232020Z.tar.xz.asc
```

The above utilizes the wget linux command line utility to make the HTTPS request.

### Little TAR/XZ Referesher

* What does `.tar.xz` file extension mean?

Means our stage 3 archive has been tarred and compressed.

It's really two file extensions in series.

* `.tar` - This is a collection of files (or not) that has been tar'd.
  The collection of files was *archived* into a single file, so that the single file really represents the collection.
  This allows us to pack a bunch of files into 1 file.
  And if you think about it, it makes sense. 
  A stage 3 is a mostly Linux system and all the files therein, and all we have right now is a single file.

* `.xz` - after tarring the file is compressed with xz.

### How Do I Verify My Stage File is Unaltered From Download?

Answer: You verify that the sha256 sum of the .tar.xz matches the sha in the .sha256 file.

For example,

```bash
sha256sum stage3-arm64-openrc-20250323T232020Z.tar.xz
```

Should match,

```bash
cat stage3-arm64-openrc-20250323T232020Z.tar.xz.sha256
```

### What is a .asc file?

A .asc file contains a detached signature.
This is a cryptographic signature created from a file, but stored separately from the original file.

The signature file contains;

* The signature data itself (the cryptographic hash of the original file, encrypted with the signer's private key)
* Information about which key was used to create the signature
* Timestamp Information
* Other Metadata

### How Do I Verify that My Stage File comes from A Legitimate Source?

We check the GPG Key in order to make sure it came from who we expect.
In this case `releng@gentoo.org`.

First Let's Retrieve the Keys, and Import them into our GPG Keychain.

```bash
wget -O - https://qa-reports.gentoo.org/output/service-keys.gpg | gpg --import
```

Now we Can Verify our Stage 3 File By Running,

```bash
gpg --verify stage3-arm64-openrc-20250323T232020Z.tar.xz.asc stage3-arm64-openrc-20250323T232020Z.tar.xz.asc
```

Make sure to verify the signature matches that found at https://www.gentoo.org/downloads/signatures/.
Just In Case.

### Unpacking the Stage File

Now we have to unpack our stage file onto our root FileSystem.

Unpacking the Stage File goes like this,

```bash
tar xpvf stage3-arm64-openrc-20250323T232020Z.tar.xz  --xattrs-include="*.*" --numeric-owner -C /mnt/gentoo
```

This will,

1. Decompress the xz Compressed File
2. Expand the `.tar` archive,
3. Place the resulting files in `/mnt/gentoo`.

And Ta-Da, we now have a mostly complete Linux system in `/mnt/gentoo`.

## Introducing Komon DEI

Next from wherever you have it handy copy over komon-dei into the system we are preparing.

You Can Download The Komon DEI Repository at Github Over Here: https://github.com/commtac2/komon-dei

We're going to need it local to the system we're cooking.
komon-dei is what's called a Gentoo Repository or "Overlay" plus a Few Miscellaneous Files for Convenience.

The komon-dei overlay will make profiles available to the Gentoo System Later for Installation. 
These profiles will Dictate which Packages are added to the System Set of packages,
and How They're to be Compiled and Installed.

```bash
cp -vR /local/path/to/komon-dei /mnt/gentoo/var/db/repo
cp -vR 
/local/path/to/komon-dei/komon-config-files/misc-komon-dei-files/repos.conf 
/mnt/gentoo/etc/portage
```

## Onwards

Now it's time to install and enter the new system.
That'll be next.
