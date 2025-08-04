# GPG Key Generation Guide

## First set the configuration

Setting the gpg configuration,

Mostly taken from the Gentoo GUide on Generating a GLEP 63 Based OpenPGP Key

## Create the new master Key

```bash
gpg --expert --full-generate-key
```

Let's then follow the steps, and only give the master key
capabilities to certify other keys.

Make sure to give a name and username and no comment.

## Creating Subkeys

You're usually going to want to create subkeys for signing and encryption

```bash
gpg --expert --edit-key <Something>
>addkey
```

and then you can choose to add an encryption or signing key.
Repeat twice, once for both encryption and signing.

## EXporting Keys to the Yubikey

```bash
gpg --expert --edit-key <Something>
>keytocard
> key 1
```

and just make sure the appropriate daemons are in place.

