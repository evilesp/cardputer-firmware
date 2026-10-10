# cardputer-firmware

Firmware for Cardputer Linux (M5Stack Cardputer ADV). Versions are listed in [index.txt](index.txt).

The terminal has the update commands built in (no brew package):

```bash
SystemUpdate            # newest stable version: downloads it, checks it, asks [y/N]
SystemUpdateBeta        # newest version including betas
```

Each image is a release asset. The index holds its SHA-256 and size, and the device checks both
before it asks whether to restart into the new firmware. Only `y` installs; any other answer
discards the downloaded image.

Stable: `v0.1.0.1`. Beta for testing: `v0.1.0.3(beta)` (use `SystemUpdateBeta`). The older
`v0.1(beta)` is still listed, in case you want to go back.
