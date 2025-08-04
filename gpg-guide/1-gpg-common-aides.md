# GPG COmmon Operations

## Listing Keys

In order to list the public keys you have,

```bash
gpg --list-keys
```

or to list your private keys you own/created,

```bash
gpg --list-secret-keys
```

## What is a Fingerprint?

A fingerprint is a short sequence of hex that uniquely
identifies a gpg public key. It is created by applying a hash function over
the public key data.

Fingerprints can be listed with,

```bash
gpg --list-keys --fingerprint
or
gpg --fingerprint
```

## Importing Keys

```bash
gpg --import <some-dot-asc-or-sig-binary>
```

When importing keys you'll also usually want to set the trust and certification 
levels,

```bash
gpg --edit-key FINGERPRINT
> trust
> sign
```

## Exporting Keys

In order to export a public key please run,

```bash
gpg --export --armor --output some-key.asc <key-id>
```

or to export private keys (be careful when doing so).

```bash
gpg --export-secret-keys --armor --output private_key.asc <key-id> 
```

when backing up keys, you probably also want to backup the revocation cert,

```bash
gpg --output revoke.asc --gen-revoke YOUR-KEY-ID
```

## Deleting Keys

```bash
gpg --delete-key <key-id>
```

And for a private key,

```bash
gpg --delete-secret-key <key-id>
```

## Signing and Verifying

* In order to sign a file,

```bash
gpg --sign file.txt
```

## Certifying

so to sign a key we're going to want to run either

```bash
gpg --sign-key KEY_ID
```

OR

```bash
gpg --edit-key KEY_ID
> sign
```

and then make sure the right certification level is chosen.
