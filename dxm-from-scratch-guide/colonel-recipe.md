# Colonel Recipe

## Step 1

You're going to want to start with your normal everyday bcm2712_defconfig.

You can get you one by running the following in `/usr/src/linux`,

```bash
make bcm2712_defconfig
```

## Step 2

Then the Colonel goes in and Adds some Essential Herbs

* Remove UEFI Boot - The Raspberry PI 5 Relies on Das Built-In-BootLoader Yah?
* Remove ARM 8.3+ Features - Useless as the Cortex a76 is only 8.2 (mostly)
* Remove Non Essential Errata - The bcm2712_defconfig comes with Errata Irrelevant to the Cortex a76
* Disable Virtualization - Keep It Real (Goes Wrong?)
* KSPP - Enable Most Recommended Settings from Kernel Self Protection Program as per https://kspp.github.io/Recommended_Settings

## Step 3

Then the Colonel goes in and adds Essential Spices.
Amongst Them,

* Switch To Tickless Clock
* Change Preemption Model as needed
* Build In ext4, xfs, btrfs, fuse, vfat, exfat file systems
* SHA512 For Module Signing.
* Remove Default Kernel Command String

## Step 4

Serve.
Enjoy.

---

Warm Regards,
The Cool Pool Colonel
