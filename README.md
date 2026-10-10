# cardputer-firmware

Firmware for Cardputer Linux (M5Stack Cardputer ADV). Versions are listed in [index.txt](index.txt).
The device installs them with the `SystemUpdate` package from brew:

```bash
brew update
brew install SystemUpdate
SystemUpdate            # pick a version, Enter, y
```

Each image is a release asset. The index holds its SHA-256 and size, and the device checks both
before it switches to the new firmware. The newest version is `v0.1.0.1` (stable). The older
`v0.1(beta)` is still listed, in case you want to go back.
