# ESP Linux firmware

A Linux-style terminal for the **Guition JC4880P443C** (ESP32-P4, 4.3" 800x480 touch screen, microSD, ESP32-C6 for Wi-Fi and Bluetooth). It is not a Linux kernel: a firmware that behaves like Ubuntu Server, with a shell, `apt`, a graphical desktop and about 200 installable programs.

Versions are in [index.txt](index.txt), what changed in [changelog.txt](changelog.txt), the binaries in the [releases](https://github.com/evilesp/cardputer-firmware/releases). All versions are beta.

```bash
SystemUpdateBeta        # the newest version: downloads it, checks it, asks [y/N]
SystemBackUpdate        # back to the version before
UpdateVersionInfo       # what is installed and what is available
```

## What is in it
- **Boot** with the usual `[  OK  ]` lines, Wi-Fi and Bluetooth come up in the background.
- **Network:** `ping` `traceroute` `mtr` `fping` `nmap` `arp-scan` `arping` `tcpdump` (the frames of this device) `iperf` `whois` `dig` `nslookup` `curl` `wget` `nc` `ntpdate` `wakeonlan` `iw` `iwlist` `iwconfig` `nmcli` `ip` `route` `ss` `ipcalc` `mdns-scan` `ssdp-scan`
- **Bluetooth LE:** `bluetoothctl` `btscan` `hcitool` `hciconfig` `gatttool`. The radio has no classic Bluetooth, so no audio and no serial profile.
- **Desktop:** `apt install desktop`, then `desktop`. Windows, a launcher, and a terminal window that is the real console.
- **Languages**, as packages:

| Install | Run | Good to know |
|---|---|---|
| `apt install python3` | `python3 app.py` | no big integers, no pip packages with C code |
| `apt install node` | `node app.js` | no Proxy, typed arrays or native npm modules |
| `apt install gcc` | `gcc a.c -o a`, `./a` | no casts between pointer types, no bit layouts of unions |
| `apt install g++` | `g++ a.cpp -o a`, `./a` | a part of the STL |
| `apt install go` | `go run a.go` | goroutines are cooperative, no reflect or unsafe, no type check |
| `apt install rustc` | `rustc a.rs -o a`, `./a` | no borrow checker or type check, no crates |

These are not native compilers: this firmware cannot load machine code, so the source is translated to Lua and runs on the Lua of the firmware, much slower than compiled code. They cover a large part of each language, not all of it. Bash scripts run in the shell of the firmware.

A program in any of these languages installs like the others: an index line with `needs:python3` and a file that starts with `#!/usr/bin/env python3` (for C: `#!/usr/bin/env gcc -run`, Rust: `#!/usr/bin/env rustc -run`, Go: `#!/usr/bin/env go -run`).

## Touch
In programs that do not read the touch screen themselves: a drag gives the arrow keys, a tap Enter, a long press space, and the X in the corner ends the program. Board games take a tap on the square itself.

## Flashing over USB
Hold BOOT if the port does not answer, then flash a binary from the releases at offset 0x10000 with esptool, or use the update commands above once a version runs.

The older document [FIRMWARE.md](FIRMWARE.md) was written for the earlier Cardputer port: the commands still apply, the pins and the screen do not.
