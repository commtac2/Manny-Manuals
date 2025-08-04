# Encrypting and Decrypting

## Asmmetric Encryption

To use a certain user's Public Key to encrypt some file.txt we can use,

```bash
gpg --encrypt --output some-file.gpg --recepient <Receipient-Address> file.txt
```

where `<Recepient-Address>` can be,

1. Email Address
2. Key ID, or Fingerprint

Now this is going to produce a binary file.
If you want to produce a base64 encoded version of the file,
which might be easier to post, share, etc you can provide the `--armor`
flag which will produce Base64 Encoded ASCII instead.

For example,

```bash
gpg --encrypt --armor --output some-file.asc --recepient <Addr> file.txt
```

## Symmetric Encryption

```bash
gpg --symmetric --output encryptedFile.txt.gpg --cipher-algo AES256 file.txt
```

## Decrypting

For smmetrically encrypted files it'll prompt you for a passphrase,

```bash
gpg --decrypt file.txt.gpg --output file.txt
```

For asymetrically encrypted files it'll search your keyring for the
corresponding private key.
