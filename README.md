# cardputer-firmware

Firmware for Cardputer Linux (M5Stack Cardputer ADV). Versions are listed in [index.txt](index.txt),
with the SHA-256, the size, the release date and what changed compared with the previous version.

The terminal has the update commands built in (no brew package):

```bash
SystemUpdate            # newest stable version: downloads it, checks it, asks [y/N]
SystemUpdateBeta        # newest version including betas
SystemBackUpdate        # every version, betas too: type the number to install it
UpdateVersionInfo       # what changed, release date and size (newest, or UpdateVersionInfo 2)
```

Each image is a release asset. The index holds its SHA-256 and size, and the device checks both
before it asks whether to restart into the new firmware. Only `y` installs; any other answer
discards the downloaded image.

Stable: `v0.1.0.1`. Beta for testing: `v0.1.0.4(beta)` (`SystemBackUpdate` or `SystemUpdateBeta`).
The older `v0.1(beta)` and `v0.1.0.3(beta)` are still listed, in case you want to go back.
