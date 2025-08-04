# Random

## Generate with urandom
How to generate random base64 characters

```bash
head -c 12 /dev/urandom | base64
```

will give you 12 bytes from /dev/urandom which is 16 characters.

if you want hex

```bash
head -c 12 /dev/urandom | xxd -p
```

## Generate with OpenSSL

Generating 12 characters of base64,

```bash
openssl rand -base64 12
```

Generating 12 characters of hex,

```bash
openssl rand -hex 12
```

## Symmetric Encryption with OpenSSL

lets do symmetric

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -in plaintext.txt -out plaintext.txt.encrypted
openssl enc -d -aes-256-cbc -pbkdf2 -in plaintext.txt.encrypted -out plaintext.txt
```

to view available ciphers

```bash
openssl list -cipher-algorithms
```
