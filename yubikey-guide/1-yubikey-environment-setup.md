# Yubikey Environment Setup

To configure the Yubikey we'll be using Yubico's ykman.
Yubico's ykman is a cross-platform application for managing and configuring
YubiKeys. It has a command line interface (CLI) that uses a Python 3.6
(or later) library.

## Creating An Environment for yubikey

Dei Ex Machina Cum with yubikey-manager preinstalled.

yubikey-manager can also be installed in a virtual environment with,

```bash
python3 -m venv yubikey-environment
source yubikey-environment/bin/activate
pip3 install yubikey-manager
```

## Setting The Yubikey Lock Code

First of all, if you want to reset the whole Yubikey before anything,

```bash
ykman config reset
```

There is one master PIN that allows enabling or disabling of individual apps.
One Note: This Lock Code is going to be 32 hex digit.
Let's set that first,

```bash
ykman config set-lock-code
```

and now let's Enable/Disable Services we do/don't want.
Let's say we want to enable FIDO, and OpenPGP but disable everything else,

```bash
ykman config usb --disable HSMAUTH --force
ykman config usb --disable PIV --force
ykman config usb --disable OTP --force
ykman config usb --disable U2F --force
ykman config usb --disable OATH --force
ykman config usb --enable FIDO2 --force
ykman config usb --enable OPENPGP --force
```

You can go ahead and confirm with,

```bash
ykman info
```

## Setting the FIDO2 Pin

We'll be using FIDO2 for a lot of 2FA, so let's do it.

The FIDO Application can be reset with

```bash
ykman fido reset --force
```

```bash
ykman fido info
ykman fido access change-pin
```

### Using FIDO

If you want to check out which credentials are in place,

```bash
ykman fido credentials list
```

## Setting the OATH Password

This is cool for storing up to 32 TOTP (TIme Based One Time Passwords), from
which you can generate the 2FA codes used by many websites.

Let's set the pin,

```bash
ykman oath access change
```

### Using OATH

Adding a secret to the yubikey

```bash
ykman oath accounts add --oath-type TOTP --touch --issuer blah some-name
```

and then to list accounts,

```bash
ykman oath accounts list
```

## Setting OTP Pin

First we can wipe the existing slots if they are programmed,

```bash
ykman otp delete 1 --force
ykman otp delete 2 --force
```

Then both slots can be configured with a US Keyboard as follows
(enter or no enter. up to you),

```bash
ykman otp static --keyboard-layout US --no-enter --force 1
ykman otp static --keyboard-layout US --no-enter --force 2
```

to lock in both slots,

```bash
ykman otp settings --new-access-code - 1
ykman otp settings --new-access-code - 2
```

where you will be prompted for a 12 digit hexadecimal.

## Setting OpePGP Keys

You can either set up gpg keys locally then import them onto the yubikey
or you can generate them directly on the yubikey.

Downside of generating directly on the yubikey: you can never export the
key. They stay on the yubikey and that's final.

There's also a user pin and an admin pin here that have to be set,

```bash
ykman openpgp access change-pin
ykman openpgp access change-admin-pin
```

## Setting PIV

I usually disable but theres a PIN, PUK (Pin Update Key), and one more Admin management key.

I Believe There are Four slots where you can store x509 certificates.

