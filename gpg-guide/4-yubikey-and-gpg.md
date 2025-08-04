# Yubikeys and gnupg

## Verify detection of yubikey

```bash
gpg --card-status
```

## Enter Card Edit Menu

In order to enter the card edit menu please,

```bash
gpg --edit-card
```

and if you have any questions just type help in the menu.
`admin` might be needed for the full list of commands.

## Exporting Keys to the Yubikey

```bash
gpg --expert --edit-key <Something>
>keytocard
> key 1
```

and just make sure the appropriate daemons are in place.
