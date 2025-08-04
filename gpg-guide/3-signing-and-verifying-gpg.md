# Signing and Verifying

## Signing

There are 3 primary ways to sign

1. Sign directly with your private key (produce a .gpg)
2. Produce a detached signature (produce a .sig file)
3. Produce a Clearsigned signature with the original text, if text
   (produces a .asc armored file)

For example the below will combine the original file, and the signature
into a single .gpg file. the original contents will be compressed.

```bash
gpg --output some-file.txt.gpg --sign <file.txt>
```

while the below will create a detached signature,

```bash
gpg --output some-file.txt.sig --detach-sign <file.txt> 
```

you can also attach `--armor` to the above to get base64 encoded ASCII

and then for a clearisgned where the signature is sort of appended
which is useful for text

```bash
gpg --output some-file.txt.asc --clear-sign <some-file>
```

And then in order to verify it

```bash
gpg --verify <file.sig> file
```
