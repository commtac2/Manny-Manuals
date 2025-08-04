# Cryptsetup Overview


## BenchMarking

If you Just Wanna See what's Up,

```bash
cryptsetup benchmark
``` 

## Formatting the Drive

```bash
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 --hash sha512 --pbkdf argon2id --pbkdf-memory 1048576 --pbkdf-parallel 2 --verify-passphrase --use-random /dev/<device-of-your-choice>
```

This will

* Encrypt the device with AES with a 512 bit Key.
* Use the Sha512 Algorithm where appropriate.
* Ensure the Security of the Drive will be kept with the chosen password, and scrambled with argon2id.
  We specify 1G of Memory and 2 cores.
  Cryptsetup will colloborate and approximate an appropriate numebr of rounds for the pbkdf.
* Use /dev/random for the Random Number Sourcing
* Verify the Given Passphrase before Commiting to It.

Please Note, XTS cute the Key Size in Half so 512 bit Key -> Functionally 256 bits.
Feel Free to use an Encrypted Keyfile Instead.

## Header Backup

You probably very much want to backup the Header,

```bash
cryptsetup luksHeaderBackup /dev/<your-device> --header-backup-file save-me
```

## Opening the Drive

The Luks Encrypted Drive can be handled just like any Drive.
You can create a handle to the device as `/dev/<some-name-you-wanna-map-to>`

```bash
cryptsetup open /dev/<device-of-your-choosing> <some-name-you-wanna-map-to>
```
