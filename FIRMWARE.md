# Cardputer Linux — dokumentacja firmware

Terminal w stylu Ubuntu Server dla **M5Stack Cardputer ADV** (ESP32-S3, ekran 240×135, klawiatura, microSD, bez PSRAM).
Ten dokument opisuje, co potrafi firmware, jak go używać, jak pisać aplikacje i sterowniki oraz jak działa pod spodem.

> **Wersja opisana:** `v0.1.0.25` (zawiera `apt`, `extended-display`, rozruch w stylu Ubuntu i nowe polecenia). Starsze wydania nie mają tych funkcji —
> sprawdź `UpdateVersionInfo`.
>
> **Status.** Wszystko poniżej jest sprawdzone w symulatorze na komputerze (testy `./build.sh test`) i kompiluje się dla
> Cardputera ADV. Rzeczy zależne od sprzętu (zewnętrzny wyświetlacz, piny, dźwięk, WiFi) **nie były jeszcze uruchamiane na
> urządzeniu** — przy takich miejscach jest to napisane wprost. Zgłoś, co zobaczysz, a poprawię.

## Spis treści

1. [Co to jest](#1-co-to-jest)
2. [Sprzęt, pamięć i ograniczenia](#2-sprzęt-pamięć-i-ograniczenia)
3. [Pierwsze uruchomienie i rozruch](#3-pierwsze-uruchomienie-i-rozruch)
4. [Terminal i klawiatura](#4-terminal-i-klawiatura)
5. [System plików](#5-system-plików)
6. [Powłoka (shell)](#6-powłoka-shell)
7. [Polecenia — referencja](#7-polecenia--referencja)
8. [Pakiety: `apt` i `brew`](#8-pakiety-apt-i-brew)
9. [WiFi i sieć](#9-wifi-i-sieć)
10. [Piny i magistrale](#10-piny-i-magistrale)
11. [Zewnętrzny wyświetlacz: `extended-display`](#11-zewnętrzny-wyświetlacz-extended-display)
12. [Aktualizacje firmware](#12-aktualizacje-firmware)
13. [Pisanie aplikacji w Lua](#13-pisanie-aplikacji-w-lua)
14. [API Lua — referencja](#14-api-lua--referencja)
15. [Środowisko graficzne: `gfx` i `ui`](#15-środowisko-graficzne-gfx-i-ui)
16. [Pisanie sterowników](#16-pisanie-sterowników)
17. [Tworzenie i publikowanie pakietów](#17-tworzenie-i-publikowanie-pakietów)
18. [Wydajność i pamięć](#18-wydajność-i-pamięć)
19. [Budowanie, testy i wydawanie firmware](#19-budowanie-testy-i-wydawanie-firmware)
20. [Budowa kodu](#20-budowa-kodu)
21. [Rozwiązywanie problemów](#21-rozwiązywanie-problemów)

---

## 1. Co to jest

Firmware zamienia Cardputera w mały „serwer z terminalem”:

- **Powłoka i system** z potokami, przekierowaniami, zmiennymi, `if`/`for`/`while`/`case`, funkcjami i aliasami.
- **Ponad 170 poleceń** w stylu Ubuntu (`ls -lah`, `ps aux`, `free -h`, `df -h`, `top`, `awk`, `systemctl`, `apt`...).
- **System plików na karcie microSD** (FAT32) widoczny jako `/`, plus wirtualne `/dev` i `/proc`.
- **Menedżer pakietów `apt`** (nakładka na `brew`): aplikacje, gry, animacje, narzędzia i sterowniki są programami w Lua pobieranymi z
  repozytorium GitHub (`evilesp/cardputer-formulae`).
- **Lua 5.4** z API do ekranu, grafiki, klawiatury, sieci, plików, pinów, UART/SPI/I²C, dźwięku i wideo.
- **Zewnętrzny wyświetlacz** SPI (ILI9341, ILI9488, ST7789...) jako drugi ekran z kopią terminala i grafiki.
- Aktualizacje firmware przez WiFi (`SystemUpdate`).

**Uczciwie o tym, czym to *nie* jest.** To nie jest jądro Linux: ESP32-S3 nie ma MMU, więc nie ma procesów w tle, `fork`,
praw dostępu do plików ani prawdziwych binariów ELF. Polecenia zachowują się i drukują jak w Ubuntu, a „procesy” to zadania FreeRTOS.
`ln -s`, `chmod`, `chown` i `kill` są akceptowane tam, gdzie to ma sens, albo odpowiadają tak, jak odpowiedziałby system,
któremu na to nie pozwolono (np. `Operation not permitted`). Aplikacje w `apt` to skrypty Lua, nie programy z Ubuntu.

## 2. Sprzęt, pamięć i ograniczenia

| Element | Wartość |
|---|---|
| Procesor | ESP32-S3, 2 rdzenie, 240 MHz |
| RAM | ok. 260 KB wolne po starcie (bez PSRAM); z włączonym WiFi ok. 136 KB wolne dla programów |
| Flash | 8 MB (partycje `default_8MB`) |
| Ekran | 240×135, kolor, sterownik ST7789 (wbudowany) |
| Klawiatura | matryca (klasyczny Cardputer) albo I²C TCA8418 (ADV) |
| Karta | microSD, **FAT32** (exFAT nie działa) |
| Dźwięk | głośnik I²S, mikrofon ES8311 (ADV) |
| Inne | IMU, IR (nadajnik), dioda RGB, bateria |

**Pamięć jest najważniejszym ograniczeniem.** Lua dostaje ok. 90 KB (przy WiFi), płótno graficzne zjada kolejne 32 KB (8 bitów)
lub 64 KB (16 bitów), blok 50 KB jest trzymany w rezerwie dla TLS (HTTPS). Zasady pisania programów oszczędnych w pamięć są w
rozdziale [18](#18-wydajność-i-pamięć). Polecenie `free` pokazuje stan, `/proc/meminfo` szczegóły.

Inne ograniczenia:

- Brak procesów w tle (`&` i `jobs` nic nie robią); jedno polecenie naraz. Okna terminala (do 3) pozwalają przełączać się między sesjami.
- HTTPS **nie weryfikuje certyfikatów**. Nie wysyłaj wrażliwych danych.
- Hasło WiFi leży jawnie w `/etc/wifi.conf` na karcie.
- Terminal pokazuje ASCII (czcionka 6×8): polskie litery zamieniają się na `?` albo są transliterowane przez aplikację.
- Zegar jest poprawny dopiero po połączeniu z WiFi (NTP); wcześniej `date` pokazuje rok 1970.

## 3. Pierwsze uruchomienie i rozruch

1. Włóż kartę microSD (FAT32). Przy pierwszym starcie powstaje układ katalogów i pliki startowe.
2. Włącz urządzenie. Rozruch wygląda jak serwer Ubuntu:

```
Cardputer Linux v0.1.0.25 (ESP-IDF v5...)
cpu: ESP32-S3 rev 0, 240 MHz, 2 core(s)
[  OK  ] Reached target Memory (136 KB free).
[  OK  ] Started TLS reserve (50 KB for HTTPS).
[  OK  ] Mounted / (sd0 FAT, 14.5G, 14.4G free).
[  OK  ] Started Step Counter.
[  OK  ] Started Network Manager: wifi: ...
[  OK  ] Reached target Multi-User System.

Cardputer Linux 24.04 LTS cardputer tty1

cardputer login: user (automatic login)

Welcome to Cardputer Linux v0.1.0.25
 (GNU/Linux 6.8.0-cardputer xtensa)

  System information
  Memory usage:   48% of 336 KB
  Processes:      12
  Usage of /:     0.5% of 14.5G
  Temperature:    38.2 C
  IPv4 (wlan0):   192.168.1.20

user@cardputer:~$
```

   Linie `[  OK  ]` pokazują naprawdę wykonane kroki (karta, rezerwa TLS, licznik kroków, sieć); `[FAILED]` pojawia się, gdy któryś
   się nie udał (np. brak karty).
3. Połącz się z WiFi: `wifi connect SIEĆ HASŁO` (`wifi save SIEĆ HASŁO` zapamięta ją na starcie).
4. Pobierz listę pakietów i zainstaluj coś: `sudo apt update`, `sudo apt install fastfetch`.

Po rozruchu uruchamia się `~/.profile` (`/home/user/.profile`) — tu możesz wpisać polecenia startowe, aliasy, a także
`extended-display start`. Plik `/etc/motd` jest drukowany po ekranie powitalnym.

## 4. Terminal i klawiatura

Siatka tekstu ma **40 kolumn × 15 wierszy** pod paskiem stanu (nazwa hosta, WiFi, bateria, godzina). Historia przewijania: 160 linii.

| Skrót | Działanie |
|---|---|
| `Fn` + `;` `.` `,` `/` | strzałki góra / dół / lewo / prawo |
| `Opt` + `;` / `.` | przewijanie wstecz / dalej (PgUp / PgDn) |
| `Ctrl+C` | przerywa polecenie albo kasuje linię |
| `Ctrl+L` | czyści ekran |
| `Ctrl+U` / `Ctrl+K` / `Ctrl+W` | kasuje do początku / do końca / słowo |
| `TAB` | uzupełnia nazwy poleceń, plików i katalogów |
| `↑` / `↓` | historia poleceń (zapisywana w `/home/user/.history`) |
| `Ctrl+T` / `Ctrl+N` / `Ctrl+P` / `Ctrl+X` | nowe okno / następne / poprzednie / zamknij okno (do 3) |

Aplikacje pełnoekranowe używają samych liter i znaków (bez `Fn`): `w a s d`, strzałki, `q`, spacja, `Enter`.

## 5. System plików

Karta jest `/`. Układ tworzony przy pierwszym starcie:

| Ścieżka | Zawartość |
|---|---|
| `/bin`, `/usr/bin` | skrypty powłoki |
| `/usr/local/bin` | programy zainstalowane przez `apt`/`brew` (pliki Lua) |
| `/usr/local/lib/drivers` | sterowniki Lua (`driver.load`) |
| `/etc` | `hostname`, `os-release`, `lsb-release`, `motd`, `wifi.conf`, `settings.conf`, `brew/taps`, `extended-display.json` |
| `/home/user` | katalog domowy (`~`), `.profile`, `.history` |
| `/tmp`, `/var/lib/brew`, `/var/cache/lua` | tymczasowe, lista pakietów, pamięć podręczna bajtkodu Lua |
| `/dev/gpioN` | `echo 1 > /dev/gpio2`, `cat /dev/gpio2` |
| `/dev/null` | jak w Linuksie; `dd if=/dev/zero` i `dd if=/dev/urandom` też działają (tylko w `dd`) |
| `/proc/uptime`, `meminfo`, `cpuinfo`, `version`, `battery`, `wifi` | informacje o systemie |

FAT nie ma praw dostępu ani dowiązań. `ls -l` pokazuje typowe `drwxr-xr-x` / `-rw-r--r--` i właściciela `user`
(`root` dla `/etc`, `/bin`, `/usr`, `/var`).

## 6. Powłoka (shell)

Składnia jak w `sh`/`bash`:

```bash
ls -lah /etc | grep conf | wc -l          # potoki (bufor 16 KB)
echo "hej" > plik.txt ; cat < plik.txt     # przekierowania > >> < 2>
make && echo ok || echo blad               # && || ;
NAME=value ; echo "$NAME ${NAME}x $?"      # zmienne, $?, $1..$9, $#, $@
echo $(date +%F) $((2 + 3 * 4))           # podstawianie poleceń i arytmetyka
for f in *.txt; do wc -l "$f"; done        # pętle, wildcardy * ? [..]
if [ -f x ]; then echo jest; else echo brak; fi
case "$x" in a*) echo A;; *) echo inne;; esac
greet() { echo "hej $1"; }                 # funkcje
alias ll='ls -lah'                          # aliasy
sudo apt update                             # sudo: polecenie jako root (bez hasła)
```

Skrypty: `bash plik.sh`, `sh plik.sh`, `source plik.sh`, pierwsza linia `#!/bin/sh`.
Brak zadań w tle (`&`, `jobs`, `bg` nic nie robią). `time POLECENIE` i `timeout 5 POLECENIE` działają.

## 7. Polecenia — referencja

Poniższa lista jest wygenerowana z tabeli poleceń w `shell.cpp` (`help` pokazuje to samo na urządzeniu; `help NAZWA` opisuje jedno).

**Pliki**

| Polecenie | Opis |
|---|---|
| `ls` | list directory contents |
| `cd` | change directory |
| `pwd` | print working directory |
| `cat` | print files |
| `mkdir` | make directories (-p) |
| `rmdir` | remove empty directories |
| `rm` | remove files (-r, -f) |
| `cp` | copy files |
| `mv` | move or rename |
| `touch` | create empty files |
| `tree` | show directory tree |
| `find` | find files by name (-name PATTERN) |
| `du` | disk usage (-s, -h) |
| `df` | free space on the SD card |
| `stat` | size and type of a file |
| `file` | guess the type of files |
| `chmod` | accepted; the SD card has no permissions |

**Tekst**

| Polecenie | Opis |
|---|---|
| `echo` | print arguments (-n, -e) |
| `printf` | formatted print |
| `grep` | find lines: grep [-E -i -v -n -c -l -r] PATTERN [FILE...] |
| `awk` | pattern scanning: awk -F: '{print $1}' FILE, awk 'BEGIN{...}' |
| `gawk` | same as awk |
| `mawk` | same as awk |
| `sed` | edit lines: sed [-n -E -i] 's/A/B/g' [FILE...] |
| `cut` | cut fields or characters: cut -d: -f2 or -c1-5 |
| `tr` | translate or delete characters: tr a-z A-Z |
| `xargs` | run a command with the words from its input |
| `diff` | differences between two files |
| `cmp` | first byte where two files differ |
| `sha256sum` | SHA-256 of files or text |
| `md5sum` | MD5 of files (-c checks a list) |
| `sha1sum` | SHA-1 of files (-c checks a list) |
| `sha512sum` | SHA-512 of files (-c checks a list) |
| `cksum` | CRC checksum and size |
| `tac` | print lines in reverse order |
| `rev` | reverse every line |
| `nl` | number lines: nl -ba -w3 -s: FILE |
| `paste` | join files side by side: paste -d, a b |
| `comm` | compare two sorted files: comm -12 a b |
| `fold` | wrap long lines: fold -w 20 -s |
| `column` | columns: column -t -s: FILE |
| `expand` | tabs to spaces |
| `strings` | printable text of a file: strings -n 6 |
| `shuf` | shuffle lines: shuf -n 3 FILE, shuf -i 1-10 |
| `yes` | repeat a line until stopped |
| `factor` | prime factors: factor 360 |
| `bc` | calculator: echo 'scale=2; 10/3' \| bc |
| `cal` | calendar: cal, cal 10 2026, cal -y, cal -3 |
| `base64` | base64 encode or decode (-d) |
| `xxd` | hex dump (-r reverse, -p plain, -c cols, -l length) |
| `hexdump` | hex dump: hexdump -C FILE |
| `truncate` | set a file size: truncate -s 1K FILE |
| `ln` | copy a file (the card has no links) |
| `readlink` | full path: readlink -f NAME |
| `realpath` | full path of a name |
| `mktemp` | make a temporary file (-d: directory) |
| `tput` | terminal: tput cols, lines, clear |
| `stty` | terminal settings: stty size |
| `reset` | reset the terminal |
| `sync` | flush the card (nothing to do) |
| `umount` | unmount (the root card stays) |
| `chown` | change owner (the card has no owners) |
| `chgrp` | change group (the card has no groups) |
| `su` | switch user (use sudo) |
| `passwd` | change the password (there is none) |
| `dd` | copy data: dd if=A of=B bs=1k count=4 |
| `head` | first lines (-n N) |
| `tail` | last lines (-n N) |
| `wc` | count lines, words, bytes |
| `sort` | sort lines (-r) |
| `uniq` | drop repeated lines |
| `seq` | print a number sequence |
| `tee` | copy stdin to stdout and a file (-a) |
| `basename` | strip directory from a path |
| `dirname` | strip last component of a path |
| `true` | exit with status 0 |
| `false` | exit with status 1 |
| `less` | read a file one screen at a time (space, b, q) |
| `more` | alias of less |

**Archiwa**

| Polecenie | Opis |
|---|---|
| `tar` | archives: tar -cf A.tar FILES \| -tf A.tar \| -xf A.tar [-C DIR] \| -z for gzip |
| `gzip` | compress a file to FILE.gz (stored, not compressed) |
| `gunzip` | decompress FILE.gz |

**Powłoka i system**

| Polecenie | Opis |
|---|---|
| `test` | conditions: test -f FILE, -d, -z, -n, a = b, a -eq b |
| `[` | same as test, ends with ] |
| `expr` | arithmetic and text: expr 2 + 3 |
| `read` | read a line into a variable: read NAME |
| `shift` | move the $1 .. parameters down by one |
| `set` | set -- ARGS sets $1 ..; prints variables without arguments |
| `printenv` | print variables |
| `type` | what a name is: builtin, alias, function or program |
| `alias` | name a command: alias ll='ls -l' |
| `unalias` | remove an alias |
| `pushd` | change directory and remember the old one |
| `popd` | go back to the directory from pushd |
| `dirs` | show the directory stack |
| `jobs` | list background jobs (none: the Cardputer runs one command at a time) |
| `bash` | run a script: bash script.sh [ARGS] or bash -c 'commands' |
| `timeout` | run a command for at most N seconds: timeout 5 COMMAND |
| `watch` | run a command again every N seconds: watch -n 2 COMMAND (Ctrl+C stops) |
| `help` | list commands or show one |
| `man` | alias of help |
| `clear` | clear the screen |
| `history` | show command history |
| `env` | show environment variables |
| `export` | set variable: export NAME=value |
| `unset` | remove variable |
| `which` | locate a command |
| `sleep` | wait N seconds |
| `date` | show date and time |
| `uptime` | time since boot |
| `uname` | system information (-a, -r, -m) |
| `hostname` | show or set the hostname |
| `whoami` | current user |
| `id` | user and group ids |
| `free` | memory usage |
| `ps` | running processes |
| `top` | tasks and their processor use (q quits; -bn1 prints once) |
| `kill` | send a signal to a process (the tasks are not killable) |
| `pgrep` | find tasks by name |
| `who` | who is logged in |
| `w` | who is logged in and what they do |
| `users` | names of the logged in users |
| `last` | logins and boots |
| `groups` | groups of the user |
| `logname` | name of the login user |
| `tty` | name of the terminal |
| `nproc` | number of processor cores |
| `arch` | machine architecture |
| `lsb_release` | release information: lsb_release -a |
| `hostnamectl` | host name and system information |
| `timedatectl` | date, time zone and clock state |
| `locale` | language settings |
| `systemctl` | services: systemctl status\|list-units\|start\|stop |
| `journalctl` | the log (the same as dmesg): journalctl -n 20 |
| `sysctl` | kernel values: sysctl -a |
| `lsmod` | loaded modules |
| `lsusb` | USB devices |
| `lspci` | PCI devices |
| `nohup` | run a command (there are no hangups here) |
| `time` | run a command and show how long it took |
| `lscpu` | CPU information |
| `lsblk` | block devices (SD card) |
| `mount` | mounted filesystems |
| `dmesg` | kernel log |
| `reboot` | restart the device |
| `poweroff` | deep sleep, BOOT button wakes |
| `exit` | leave a script with status N |
| `source` | run a script in this shell |
| `sh` | run a shell script |

**Sieć**

| Polecenie | Opis |
|---|---|
| `wifi` | scan, connect, off, status, save |
| `ip` | network interface information |
| `ifconfig` | alias of ip |
| `host` | resolve a host name (DNS) |
| `nslookup` | alias of host |
| `ping` | round trip time to a host (TCP connection, port 80 by default) |
| `telnet` | open a terminal to a server: telnet HOST [PORT] |
| `curl` | transfer data over HTTP/HTTPS |
| `wget` | download a file over HTTP/HTTPS |
| `apt` | package manager: sudo apt install\|remove\|update\|upgrade\|search\|show\|list |
| `apt-get` | package manager (same as apt) |
| `apt-cache` | search and show packages: apt-cache search WORD |
| `dpkg` | installed packages: dpkg -l, -s NAME, -L NAME |
| `add-apt-repository` | add a package source (a tap): sudo add-apt-repository URL |
| `brew` | package manager: install apps from your tap |
| `SystemUpdate` | install the newest firmware over WiFi (asks before restart) |
| `SystemUpdateBeta` | like SystemUpdate, but also beta versions |
| `SystemBackUpdate` | list every version (betas too) and install the one you choose |
| `UpdateVersionInfo` | what changed, release date and size of a version |
| `lua` | run a Lua script (lua FILE \| lua -e CODE) |

**Sprzęt**

| Polecenie | Opis |
|---|---|
| `gpio` | read and drive GPIO pins |
| `beep` | play a tone on the speaker |
| `battery` | battery level and voltage |
| `brightness` | screen brightness 0-255 |
| `imu` | accelerometer and gyroscope readings |
| `sendIr` | send an infrared signal: sendIr nec ADDRESS COMMAND |
| `osascript` | macOS volume scripts: osascript -e 'set volume output volume 50' |
| `system_profiler` | audio hardware report: system_profiler SPAudioDataType |


### Opcje najczęściej używanych poleceń

| Polecenie | Obsługiwane opcje |
|---|---|
| `ls` | `-a -A -l -h -r -t -S -X -U -F -p -1 -s -n -o -g -m -x -C -R -d -i --si --group-directories-first --sort=` — nazwy ze spacjami w cudzysłowie jak w Ubuntu 24.04 |
| `date` | `+FORMAT` (`%Y %m %d %H %M %S %a %A %b %B %e %j %y %Z %z %s %p %I %u %F %T %D %R %-d`), `-u`, `-d "@epoch"`, `-d "YYYY-MM-DD HH:MM"`, `-d yesterday/tomorrow/"3 days ago"`, `-I`, `-R` |
| `free` | `-b -k -m -g -h --si -t -s N -c N` (kolumny `total used free shared buff/cache available`) |
| `df` | `-h -H -T -k -m -i -a` |
| `ps` | `ps`, `ps aux`, `ps -ef`, `ps -e`, `ps -o pid,user,comm,%cpu,...` |
| `uptime` | `-p`, `-s` |
| `uname` | `-a -s -n -r -v -m -p -i -o` |
| `top` | `-b -n N -d SEC`; interaktywnie `q` kończy |
| `awk` | `-F -v -f`, wzorce, `BEGIN`/`END`, zakresy, tablice, funkcje użytkownika, `printf`, `sub gsub match split substr index sprintf tolower toupper sin cos atan2 exp log sqrt int rand srand`, `getline`, `print > "plik"` |
| `xxd` | `-r -p -c -l -s -u` |
| `base64` | `-d -w -i` |
| `md5sum` `sha1sum` `sha256sum` `sha512sum` | pliki lub stdin, `-c LISTA` |
| `cal` | `cal`, `cal 10 2026`, `cal 2026`, `-y -m -3` |
| `bc` | `-l`, `scale=N`, `+ - * / % ^`, nawiasy, zmienne `a`–`z`, `sqrt s c a l e` |
| `stat` | pełny format Linuksa, `-c '%n %s %F %a %y'` |
| `systemctl` | `status`, `list-units`, `is-active`, `start/stop/restart` (sieć), `reboot`, `poweroff` |
| `gpio` | `list readall info mode read write toggle blink pwm adc wait watch pulse reserved` |

## 8. Pakiety: `apt` i `brew`

`apt` to polecenie z Ubuntu. Pod spodem pracuje `brew` — ten sam menedżer, te same pakiety, ten sam indeks — ale komunikaty i
kolejność kroków są takie jak w `apt`. Polecenia, które zmieniają system, wymagają `sudo` (jak w Ubuntu):

```bash
sudo apt update                       # pobiera listy pakietów z repozytoriów
sudo apt install tanks cmatrix        # instaluje (pyta Do you want to continue? [Y/n]; -y pomija pytanie)
sudo apt install -y extended-display ili9341
sudo apt upgrade                      # aktualizuje zainstalowane pakiety
sudo apt remove tanks                 # usuwa (purge, autoremove też są akceptowane)
sudo apt install all                  # wszystkie pakiety z listy (zainstalowane są aktualizowane)
apt search wąż                        # szukanie w nazwach i opisach
apt show tanks                        # Package/Version/Depends/Description
apt list --installed                  # lub --upgradable
apt policy tanks                      # Installed / Candidate
dpkg -l                               # zainstalowane pakiety
dpkg -s tanks                         # status;  dpkg -L tanks — pliki
apt-cache search ui                   # apt-cache show/policy/depends/pkgnames
add-apt-repository URL                # dodaje repozytorium (tap); -r usuwa
apt edit-sources                      # pokazuje źródła (/etc/brew/taps)
```

Przykładowy wynik:

```
$ sudo apt install tanks
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  ui
The following NEW packages will be installed:
  tanks ui
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Do you want to continue? [Y/n] y
Get:1 https://raw.githubusercontent.com/evilesp/cardputer-formulae/main/Driver/ui.lua
Setting up ui (1.0.0.1) ...
Get:2 https://raw.githubusercontent.com/evilesp/cardputer-formulae/main/Formula/tanks.lua
Setting up tanks (1.0.0.3) ...
```

Bez `sudo` dostaniesz — jak w Ubuntu —
`E: Could not open lock file /var/lib/dpkg/lock-frontend - open (13: Permission denied)`.

**Pakiety zależne.** Linia `needs:` w indeksie (np. `needs:ui json`) mówi, co zainstalować wraz z pakietem; `apt` pokazuje to jako
„additional packages”. **Sterowniki** (`Driver/...`) trafiają do `/usr/local/lib/drivers`, programy do `/usr/local/bin`.

**Skrót dla sterowników wyświetlacza.** `apt install extended-display ili9341` instaluje pakiet `extended-display` i sterownik
`xd-ili9341` (słowo, które nie jest pakietem, a ma sterownik `xd-SŁOWO`, jest brane jako sterownik).

`brew` nadal działa (`brew install`, `brew upgrade`, `brew outdated`, `brew tap` ...) — to ten sam mechanizm.
`brew upgrade` aktualizuje też, gdy zmieniła się sama zawartość pliku, nie tylko numer wersji.

## 9. WiFi i sieć

| Polecenie | Opis |
|---|---|
| `wifi scan` | sieci w zasięgu |
| `wifi connect SSID HASŁO` | łączy |
| `wifi save SSID HASŁO` | zapamiętuje sieć i łączy ją przy starcie |
| `wifi status` / `wifi off` | stan / wyłącza radio (zwalnia ok. 45 KB RAM) |
| `ip a`, `ifconfig` | adres, maska, brama |
| `host NAZWA`, `nslookup` | DNS |
| `ping HOST [PORT] [N]` | czas połączenia TCP (port 80 domyślnie; to nie ICMP) |
| `curl`, `wget` | HTTP i HTTPS (`-o`, `-s`, `-L`, `-I`, `-X`, `-d`, `-H`) |
| `telnet HOST [PORT]` | terminal do innego komputera (`Ctrl+]` kończy) |
| `server PORT [NAZWA]` | pakiet: strona w przeglądarce z terminalem, plikami i stanem urządzenia (bez hasła, tylko w sieci domowej) |

## 10. Piny i magistrale

### Które piny są wolne

| Cardputer ADV (domyślnie) | |
|---|---|
| **Wolne** | G1, G2 (złącze Grove), G3, G4, G5, G6, G7, G13, G15, G16, G17, G18, G44 (IR TX), G47, G48 |
| Zajęte | G0 BOOT · G8/G9 I²C (klawiatura, IMU) · G10 bateria · G11 przerwanie klawiatury · G12/G14/G39/G40 karta SD · G33–G38 LCD · G41–G43/G46 dźwięk · G19/G20 USB · G21 dioda RGB · G26–G32/G45 flash |

Klasyczny Cardputer: G3–G9, G11, G13, G15 to macierz klawiatury; wolne są G1, G2, G16–18, G44, G47, G48.
Próba użycia zajętego pinu kończy się komunikatem z powodem (`gpio write 13 1` → `reserved (keyboard matrix)`).

### Polecenie `gpio`

```bash
gpio readall                  # wszystkie piny: tryb, poziom, kto go używa (free / uart / spi / i2c / extended-display / powód rezerwacji)
gpio info 5                   # ADC? PWM? dotyk? kto trzyma?
gpio mode 5 out|in|pullup|pulldown
gpio write 5 1                # też:  echo 1 > /dev/gpio5
gpio read 5                   # też:  cat /dev/gpio5
gpio toggle 5
gpio blink 5 10 250           # 10 razy co 250 ms
gpio pwm 5 128 2000           # wypełnienie 0–255, 2 kHz
gpio adc 1                    # napięcie w mV (piny 1–10)
gpio wait 5 rising 10         # czeka na zbocze (rising/falling) albo poziom (high/low), kod 124 po czasie
gpio watch 5 30               # drukuje każdą zmianę poziomu z czasem, 30 s
gpio pulse 6 100              # impuls 100 µs (np. wyzwalanie)
```

### UART, SPI, I²C z Lua

Piny dla magistrali programu sprawdza rejestr (`periph_claims`): musi być wolny, nie może być użyty dwa razy.
Magistrale otwarte przez program są zamykane, gdy program się kończy.

| Magistrala | Otwarcie | Sprzęt |
|---|---|---|
| UART | `uart.open{rx=PIN, tx=PIN, baud=9600}` | Serial1/Serial2 (do 2 naraz) |
| SPI | `spi.open{sck=, mosi=, miso=, cs=, mode=0, clock=1000000}` | SPI3 (HSPI), **jedna** magistrala naraz |
| I²C | `i2c.open{sda=, scl=, clock=100000}` | Wire1 (jedna); klawiatura ADV używa Wire na G8/G9 |

Jeśli działa `extended-display`, zajmuje magistralę SPI3 i swoje piny — `spi.open` dostanie błąd, dopóki go nie zatrzymasz.

Pakiety do sprzętu: `gps`, `gpsview`, `i2cscan`, `softspi` (SPI programowe na dowolnych pinach), `mcp3008`, `adcview`, `rtc` (DS3231), `remote` (IR).

## 11. Zewnętrzny wyświetlacz: `extended-display`

Drugi ekran na wolnych pinach: kopia terminala i grafiki aplikacji na panelu SPI. Obsługiwane układy są **danymi** (pakiety
`xd-NAZWA`) — nowy panel to nowy plik Lua, nie nowy firmware. Firmware zawiera ogólny sterownik paneli LCD (LovyanGFX) i modułu
`xdisp`.

> **Status:** napisane i skompilowane, sterowniki przepisane z dokumentacji układów; **nie sprawdzone na prawdziwym panelu**.
> Jeśli ekran pozostaje czarny albo ma zły kolor, patrz [diagnostyka](#diagnostyka-wyświetlacza).

### Instalacja

```bash
sudo apt install extended-display ili9341     # polecenie + sterownik
extended-display list                          # sterowniki: ili9341 ili9342 ili9488 st7789 st7789-240 st7789-135 st7735 st7796
```

Sterownik innego układu: `sudo apt install xd-ili9488`. Gdy sterownika brakuje, `extended-display set` i `start` pobiorą go same
(potrzebne WiFi i `sudo apt update` wykonane choć raz).

### Podłączenie

Domyślnie program wybiera pierwsze wolne piny płytki (ADV: G1…G6). Zmienisz je w pliku konfiguracyjnym. Potrzebne sygnały:

| Sygnał | Opis |
|---|---|
| `sclk`, `mosi` | zegar i dane SPI |
| `dc` | dane/komenda |
| `cs` | wybór układu (`-1`, jeśli panel ma CS na stałe do masy) |
| `rst` | reset (`-1`, jeśli niepodłączony) |
| `bl` | podświetlenie (PWM, jasność `bright`); `-1`, jeśli na stałe |
| `miso` | zwykle `-1` (odczyt z panelu niepotrzebny) |

Zasilanie 3,3 V i masa wspólna. Kable krótkie (zegar 40 MHz!).

### Polecenia

| Polecenie | Działanie |
|---|---|
| `extended-display set [STEROWNIK]` | z nazwą sterownika: zapisuje konfigurację domyślną dla tego sterownika i **otwiera plik w `nano`**; bez nazwy: otwiera istniejący plik |
| `extended-display set KLUCZ WARTOŚĆ` | zmienia jedną wartość: `set rotation 1`, `set brightness 40`, `set pins.cs 3`, `set freq 20000000`, `set invert true` |
| `extended-display start [STEROWNIK]` | uruchamia wyświetlacz; ze sterownikiem — przełącza się na niego (pobierany, jeśli go nie ma) i zapisuje w konfiguracji |
| `extended-display stop` | wygasza ekran, zwalnia piny i SPI |
| `extended-display restart` | zatrzymuje i uruchamia od nowa z konfiguracji |
| `extended-display bright PROCENT` | jasność 0–100, zapisana w konfiguracji |
| `extended-display status` | co działa: sterownik, rozmiar, obrót, skala, zegar, piny, liczba klatek |
| `extended-display test` | paski kolorów, biały kwadrat w lewym górnym rogu (widać obrót), napis z rozmiarem |
| `extended-display enable` / `disable` | dopisuje / usuwa `extended-display start` w `~/.profile` (start przy każdym rozruchu) |
| `extended-display --config PLIK ...` | inny plik konfiguracyjny |

### Plik konfiguracyjny `/etc/extended-display.json`

```json
{
  "driver": "ili9341",
  "width": 240,
  "height": 320,
  "depth": 16,
  "rotation": 0,
  "scale": 0,
  "freq": 40000000,
  "brightness": 100,
  "invert": false,
  "bgr": false,
  "pins": { "sclk": 5, "mosi": 6, "miso": -1, "cs": 3, "dc": 4, "rst": 2, "bl": 1 }
}
```

| Pole | Znaczenie |
|---|---|
| `driver` | nazwa sterownika (`ili9341` → pakiet `xd-ili9341`) |
| `width`, `height` | rozdzielczość panelu **bez obrotu**, w pikselach |
| `memory_width`, `memory_height` | pamięć sterownika, jeśli większa niż szkło (np. ST7789 240×320 dla szkła 135×240) |
| `offset_x`, `offset_y` | przesunięcie okna w pamięci (np. 52/40 dla 135×240) |
| `depth` | 16 (RGB565) albo 18 (3 bajty na piksel, ILI9488) |
| `rotation` | **0** brak, **1** o 90°, **2** o 180°, **3** o 270° |
| `scale` | 0 = automatycznie (największa całkowita skala, na małych panelach ułamek); inaczej wymuszona skala |
| `freq` | zegar SPI w Hz (1–80 MHz); przy kłopotach zmniejsz do 10–20 MHz |
| `brightness` | 0–100 % |
| `invert` | odwrócone kolory (typowe dla ST7789 IPS) |
| `bgr` | kolejność składowych koloru BGR zamiast RGB (gdy czerwony i niebieski są zamienione) |
| `pins` | numery GPIO, `-1` = nieużywany; `sclk`, `mosi`, `dc` są wymagane |
| `init` | (opcjonalnie) własna lista komend startowych — [własny sterownik w jednym pliku](#własny-sterownik-w-pliku-konfiguracyjnym) |

Wartości z pliku mają pierwszeństwo przed wartościami sterownika.

### Co jest wyświetlane

Terminal i aplikacje graficzne (`gfx`) są kopiowane na zewnętrzny ekran. Obraz 240×135 jest skalowany całkowitą skalą
(`scale`) i wyśrodkowany: panel 320×240 pokazuje go w skali 1, panel 480×320 w skali 2. Kopia grafiki jest wysyłana najwyżej co ok. 40 ms,
żeby wolny panel nie spowalniał gry.

### Sterowniki w pakietach

| Pakiet | Układ | Rozmiar | Uwagi |
|---|---|---|---|
| `xd-ili9341` | ILI9341 | 240×320 | 2,4–2,8″, 16 bit |
| `xd-ili9342` | ILI9342C | 320×240 | panel M5Stack Core |
| `xd-ili9488` | ILI9488 | 320×480 | 18 bit (wolniejszy), 3,5″ |
| `xd-st7789` | ST7789 | 240×320 | IPS, `invert` |
| `xd-st7789-240` | ST7789 | 240×240 | kwadratowy moduł IPS |
| `xd-st7789-135` | ST7789 | 135×240 | małe szkło (offset 52/40) |
| `xd-st7735` | ST7735R | 128×160 | 1,8″; zielone/czerwone „tab” mogą wymagać `offset_x/y` |
| `xd-st7796` | ST7796S | 320×480 | 16 bit |

### Diagnostyka wyświetlacza

- **Czarny ekran.** `extended-display status` — jeśli „not running”, `start` wypisze przyczynę (np. `pin 5 is used twice`,
  `controller did not start`). Sprawdź zasilanie, `bl`, `rst` i `freq` (spróbuj 10 MHz).
- **Złe kolory** (czerwony↔niebieski): `extended-display set bgr true`. **Negatyw:** `set invert true`.
- **Przesunięty obraz / kolorowa krawędź:** `offset_x`, `offset_y`.
- **Obrócony obraz:** `set rotation 1`/`2`/`3`; `test` pokazuje biały kwadrat w lewym górnym rogu.
- **`spi.open` nie działa:** SPI3 zajmuje wyświetlacz; `extended-display stop`.
- **Brak pamięci:** `wifi off` zwalnia ok. 45 KB.

### Własny sterownik w pliku konfiguracyjnym

Panel bez gotowego pakietu możesz opisać w samej konfiguracji: ustaw `"driver": "custom"` i dodaj `"init"` — listę komend startowych.
Każda komenda to lista bajtów (`[komenda, argument, argument ...]`) albo obiekt z opóźnieniem po komendzie:

```json
{
  "driver": "custom", "width": 128, "height": 128, "depth": 16, "freq": 20000000,
  "pins": { "sclk": 5, "mosi": 6, "miso": -1, "cs": 3, "dc": 4, "rst": 2, "bl": 1 },
  "init": [ [1], {"cmd": [17], "delay": 120}, [58, 85], [41] ]
}
```

(`[1]` = SWRESET, `0x11` = SLPOUT z opóźnieniem 120 ms, `0x3A 0x55` = 16 bit, `0x29` = DISPON.) Jak pisać pakiet ze sterownikiem —
[rozdział 16](#16-pisanie-sterowników).

## 12. Aktualizacje firmware

Wbudowane polecenia (nie pakiety):

| Polecenie | Działanie |
|---|---|
| `SystemUpdate` | pobiera najnowszą **stabilną** wersję, sprawdza rozmiar i SHA-256, pyta `[y/N]`, wgrywa i restartuje |
| `SystemUpdateBeta` | jak wyżej, ale uwzględnia wersje beta |
| `SystemBackUpdate` | lista wszystkich wersji, wybierasz, którą wgrać (także starszą) |
| `UpdateVersionInfo [wersja]` | data, rozmiar i zmiany wersji |

Firmware jest publikowany w GitHub Releases repozytorium `evilesp/cardputer-firmware` wraz z `index.txt`
(`etykieta|sha256|rozmiar|adres`) i `changelog.txt` (`etykieta|data|opis`). Brak weryfikacji certyfikatu HTTPS (sprawdzany jest SHA-256 z indeksu).
Wgranie przez USB: `./build.sh upload` (porty `/dev/cu.usbmodem*`; jeśli brak, przytrzymaj BOOT/G0 przy podłączaniu).

## 13. Pisanie aplikacji w Lua

Aplikacja to plik Lua z pierwszą linią `#!lua`. Po `sudo apt install` leży w `/usr/local/bin/NAZWA` i uruchamiasz ją jak polecenie.
Argumenty są w tablicy `arg` (`arg[1]`...). Dostępne są tylko moduły z rozdziału 14 oraz biblioteki `string`, `table`, `math`, `utf8`;
nie ma `io`, `os`, `require`, `dofile` — pliki czytasz przez `fs`.

### Najmniejsza aplikacja (tekst)

```lua
#!lua
-- hello: pokazuje licznik, q kończy
local n = 0
term.fullscreen(true)
while true do
  term.clear()
  term.text(2, 2, "Licznik: " .. n, "green")
  term.flush()
  local k = key.poll()
  if k == "q" or k == "ctrl-q" then break end
  n = n + 1
  sys.sleep(100)
end
term.fullscreen(false)
```

### Aplikacja graficzna (piksele)

```lua
#!lua
local w, h = gfx.open()
if not w then print("gfx: " .. tostring(h)) return end
local x = 0
while true do
  gfx.clear(0x000000)
  gfx.disc(x % w, h // 2, 8, 0xffcc00)
  gfx.text(4, 4, "x=" .. x, 0xffffff)
  gfx.flush()
  x = x + 3
  local k = key.poll()
  if k == "q" or k == "ctrl-q" then break end
  sys.sleep(25)
end
gfx.close()
```

### Dobre nawyki

- **Wyjście:** zawsze `q` albo `ctrl-q`; `Ctrl+C` przerywa program błędem „interrupted” (nie łap go bez potrzeby).
- **Tryb testowy:** `if arg[1] == "--selftest" then ... print("selftest ok") return end` — sprawdza reguły bez ekranu.
- **Gra bez trzymania klawiszy:** firmware nie zawsze zgłasza puszczenie klawisza; ruch ma działać „po naciśnięciu” (`key.down()` jest dodatkiem).
- **Pamięć:** nie twórz tabel i domknięć w pętli klatki; używaj płaskich list liczb; duże dane czytaj z karty na żądanie.
- **Tylko ASCII** na ekranie; nazwy i komentarze w kodzie po angielsku.
- **Sieć:** tylko API bez klucza (zapisuj przykładową odpowiedź w `tests/fixtures/` do testów).

## 14. API Lua — referencja

Moduły ładują się przy pierwszym użyciu. Błędy zwykle zwracają `nil, "opis"` (nie rzucają wyjątku).

### `term` — tryb tekstowy 40×16

| Funkcja | Opis |
|---|---|
| `term.size()` | `kolumny, wiersze` |
| `term.fullscreen(on)` | przełącza pełny ekran (wyłączenie przywraca terminal) |
| `term.clear()` | czyści pełny ekran aplikacji |
| `term.put(x, y, znak, kolor)` | jeden znak |
| `term.text(x, y, tekst, kolor)` | napis od pozycji |
| `term.flush()` | pokazuje klatkę (przepuszcza `Ctrl+C`) |
| `term.print(tekst, kolor)` / `term.write(tekst, kolor)` | wydruk w terminalu (`write` bez nowej linii) |
| `term.image(adres_lub_ścieżka, x, y [, skala])` | JPEG/PNG; ścieżka bez `://` to plik z karty |
| `term.imageSize(ścieżka)` | `szerokość, wysokość` |

Kolory: `text`, `white`, `green`, `greendim`, `prompt`, `path`, `red`, `yellow`, `warn`, `cyan`, `blue`, `dir`, `magenta`, `dim`.

### `key`

| Funkcja | Opis |
|---|---|
| `key.poll()` | nazwa wciśniętego klawisza albo `nil` (nieblokująco): `"q"`, `" "`, `"enter"`, `"backspace"`, `"up"`, `"ctrl-q"` ... |
| `key.down()` | lista klawiszy trzymanych teraz (firmware v0.1.0.18+; może być `nil` na starszym) |

### `sys`

`sleep(ms)` · `millis()` · `micros()` · `delayUs(n)` · `uptime()` · `hostname()` · `user()` · `version()` · `board()` · `chip()` ·
`heap()` (wolne KB, całość KB, PSRAM KB) · `battery()` (%, mV, ładowanie) · `sd()` · `date([format])` · `time()` · `tasks()` ·
`imu()` (`ax ay az gx gy gz`) · `setting(klucz [, wartość])` · `lowPower([on])` · `deepSleep()` · `powerOff()` · `reboot()` ·
`performance([tryb])` (`off`/`on`/`auto`) · `update(url, sha256, rozmiar)`.

### `net`

`connected()` · `ip()` · `ssid()` · `scan()` · `connect(ssid, hasło [, sekundy])` · `disconnect()` · `saved()` · `save(ssid, hasło)` ·
`setAuto(on)` · `get(url [, maxBajtów])` → `status, treść` · `fetch(url [, maxBajtów [, userAgent]])` → `status, treść, obcięto` ·
`download(url)` · `upload(url, bajty)`.

### `fs`

`read(ścieżka)` · `write(ścieżka, dane)` · `append(ścieżka, dane)` · `list(katalog)` · `info(katalog)` · `exists(ścieżka)` · `stat(ścieżka)` ·
`hash(ścieżka)` (SHA-256) · `remove(ścieżka)`. Zapisy w `/etc`, `/bin`, `/usr` są chronione (poza ścieżkami, które pakiet może zmieniać).

### `gpio`

`mode(pin, "in"/"out"/"pullup"/"pulldown")` · `read(pin)` · `write(pin, 0/1)` · `adc(pin)` (mV, piny 1–10) · `pins()` (wolne piny) ·
`tone(pin, hz)` / `noTone(pin)` · `scope(pin, n, us)` (próbki ADC w mV).

### `uart`, `spi`, `i2c`, `ir`

```lua
local port = uart.open{ rx = 1, tx = 2, baud = 9600 }   -- port:read(max) :readline() :write(t) :available() :close()
local bus  = spi.open{ sck = 3, mosi = 4, miso = 5, cs = 6, mode = 0, clock = 1000000 }   -- bus:transfer(t) :write(t) :close()
local i2c  = i2c.open{ sda = 1, scl = 2, clock = 100000 }  -- :write(adr, t) :read(adr, n) :writeRead(adr, t, n) :scan() :close()
ir.send(pin, timings)  ;  ir.receive(pin, timeout_ms, gap_ms)
```

### `audio`, `video`

`audio.play(plik [, maxSek])` · `stat` · `record("file"|"null", plik, sekundy, rate)` · `synth(tryb, plik, fala, hz, sekundy, rate)` ·
`volume([%])` · `inputVolume([%])` · `beep(hz, ms)` (w tle) · `listen()` (wysokość dźwięku i poziomy z mikrofonu) ·
`video.info(plik)` · `video.play(plik)` (AVI: MJPEG + PCM).

### `steps`, `search`, `browser`, `httpd`

`steps.today() total() days(n) enabled([on]) reset()` · `search.web(zapytanie)` · `search.pictures(zapytanie)` ·
`browser.open(url)` / `lines(strona, od, ile)` / `link(strona, n)` · `httpd.start(port, nazwa, strona)` / `poll()` / `stop()` / `requests()`.

### `brew`, `driver`

| Funkcja | Opis |
|---|---|
| `brew.install(nazwa)` | `true, wyjście` albo `false, wyjście` — jedyny „proces”, który program może uruchomić |
| `brew.update()` | odświeża listy pakietów |
| `driver.load(nazwa [, strip])` | tabela ze sterownika `/usr/local/lib/drivers/NAZWA.lua`; `nil, błąd`, gdy brak |

### `xdisp` — zewnętrzny wyświetlacz

Używane przez `extended-display`; aplikacje zwykle go nie wołają.

| Funkcja | Opis |
|---|---|
| `xdisp.start{...}` | uruchamia panel; `true, szerokość, wysokość` albo `nil, błąd` (pola jak w konfiguracji, `pins = {...}`, `init = {{komenda, args..., delay = ms}, ...}`) |
| `xdisp.stop()` | zatrzymuje; zwraca, czy działał |
| `xdisp.restart()` | uruchamia z tą samą konfiguracją |
| `xdisp.bright(procent)` | jasność |
| `xdisp.info()` | tabela (`driver width height rotation scale freq bright frames pins`) albo `nil` |
| `xdisp.test()` | paski kolorów |

## 15. Środowisko graficzne: `gfx` i `ui`

### `gfx` — piksele

`gfx.open()` zajmuje ekran: terminal przestaje rysować, rysujesz w płótnie (w paskach po 15 wierszy), a `gfx.flush()` pokazuje gotową klatkę
(bez migotania). `gfx.close()` albo koniec programu oddaje ekran.

| Funkcja | Opis |
|---|---|
| `gfx.open()` | `szerokość, wysokość, głębia` (240, 135, 16 albo 8 bitów przy braku pamięci) lub `nil, błąd` |
| `gfx.close()` | oddaje ekran |
| `gfx.clear([kolor])` | wypełnia płótno |
| `gfx.pixel(x, y, k)` | piksel |
| `gfx.line(x0, y0, x1, y1, k)` | linia |
| `gfx.rect(x, y, w, h, k)` / `gfx.fill(x, y, w, h, k)` | ramka / prostokąt wypełniony |
| `gfx.hline(x0, x1, y, k)` / `gfx.vline(x, y0, y1, k)` | linie poziome i pionowe |
| `gfx.circle(x, y, r, k)` / `gfx.disc(x, y, r, k)` | okrąg / koło |
| `gfx.triangle(x0, y0, x1, y1, x2, y2, k)` | trójkąt wypełniony |
| `gfx.text(x, y, tekst, k [, rozmiar])` | tekst 6×8 px (rozmiar 1–8) |
| `gfx.textWidth(tekst [, rozmiar])` | szerokość |
| `gfx.flush()` | pokazuje klatkę |
| `gfx.rgb(r, g, b)` | kolor `0xRRGGBB` |
| `gfx.depth()` | głębia kolorów płótna |

Kolory to liczby `0xRRGGBB`. Przy 8 bitach (mało pamięci) kolory są bardziej „gruboziarniste”.
Przy działającym `extended-display` każda klatka jest kopiowana też na zewnętrzny ekran.

### `ui` — biblioteka okien i przycisków

`ui` jest sterownikiem: `sudo apt install ui`, w programie `local ui = driver.load("ui")` (z `true` jako drugim argumentem — wersja odchudzona).

```lua
local ui = driver.load("ui", true)
ui.open()                                   -- zajmuje ekran
gfx.clear(ui.theme.bg)
ui.panel(10, 10, 120, 60, "Ustawienia")     -- okno z paskiem tytułu
ui.label(16, 28, "Glosnosc", ui.theme.text)
ui.bar(16, 40, 100, 10, 7, 10)              -- pasek postępu 7 z 10
ui.button(16, 54, 50, 12, "OK", "focus")    -- "normal", "focus" albo "disabled"
gfx.flush()
```

| Funkcja | Opis |
|---|---|
| `ui.open()` / `ui.close()` | zajmuje / oddaje ekran |
| `ui.panel(x, y, w, h, tytuł)` | okno |
| `ui.label(x, y, tekst, kolor [, rozmiar])` / `ui.center(y, tekst, kolor, rozmiar [, x0, x1])` | tekst |
| `ui.button(x, y, w, h, tekst, stan)` | przycisk |
| `ui.bar(x, y, w, h, wartość, max [, kolor, tekst])` | pasek |
| `ui.list(x, y, w, pozycje, zaznaczona [, wysokość_wiersza])` | lista |
| `ui.dialog(tytuł, linie, przyciski, zaznaczony)` | okno z przyciskami |
| `ui.choose(tytuł, pozycje)` | menu modalne — numer wybranej pozycji albo `nil` |
| `ui.message(tytuł, linie)` | komunikat czekający na klawisz |
| `ui.prompt(tytuł, początkowy, max)` | pole tekstowe |
| `ui.rgb(r,g,b)`, `ui.shade(kolor, f)`, `ui.mix(a, b, t)` | kolory |
| `ui.theme` | `bg panel light border title text dim accent warn danger` |

Klawisze w oknach: `w`/`s` albo strzałki, `Enter`/spacja zatwierdza, `Backspace`/`Ctrl+Q` wychodzi.

## 16. Pisanie sterowników

**Sterownik** to plik Lua, który zwraca tabelę. Instaluje się go jak pakiet (`Driver/NAZWA.lua` w repozytorium → `/usr/local/lib/drivers/NAZWA.lua`),
a program ładuje przez `driver.load("NAZWA")`. Sterownik działa w stanie Lua aplikacji.

```lua
#!lua
-- moj: sterownik czujnika (apt install moj)
local moj = {}

function moj.open(bus, addr)
  local dev = { bus = bus, addr = addr or 0x40 }
  function dev:read()
    local raw = self.bus:writeRead(self.addr, "\x00", 2)
    return (raw:byte(1) << 8 | raw:byte(2)) * 0.01
  end
  return dev
end

return moj
```

Użycie: `local moj = driver.load("moj"); local i2c = i2c.open{sda=1, scl=2}; print(moj.open(i2c):read())`.

Wzorce w repozytorium: `Driver/gps.lua` (NMEA z sumami kontrolnymi), `Driver/softspi.lua` (SPI na dowolnych pinach), `Driver/mcp3008.lua` (działa na dowolnej
magistrali z `transfer`), `Driver/json.lua`, `Driver/eng.lua`, `Driver/ui.lua`. Testy sterowników na komputerze: `tests/lua_host/fakes.lua`
(atrapy `uart`, `spi`, `i2c`, `driver`).

### Sterownik wyświetlacza (`xd-NAZWA`)

Sterownik panelu dla `extended-display` to tabela z danymi:

```lua
#!lua
return {
  name = "ili9341", width = 240, height = 320,
  depth = 16,                 -- 16 (RGB565) albo 18 (ILI9488)
  freq = 40000000,            -- zegar SPI, który panel lubi
  invert = false, bgr = false,
  -- memory_width, memory_height, offset_x, offset_y: gdy szkło jest mniejsze niż pamięć układu
  info = "ILI9341 240x320, 16 bit colour",
  init = {                    -- lista komend startowych: { komenda, argumenty..., delay = ms }
    { 0xCF, 0x00, 0xC1, 0x30 },
    { 0x3A, 0x55 },
    { 0x11, delay = 120 },    -- SLPOUT, po nim 120 ms
    { 0x29, delay = 20 },     -- DISPON
  },
}
```

Jak powstaje nowy sterownik:

1. Weź z karty katalogowej układu (albo z biblioteki TFT_eSPI/LovyanGFX) **sekwencję inicjalizacji**: zapisy `komenda, argumenty`.
   Komendy `MADCTL` (0x36) i `COLMOD` (0x3A) ustawia sam firmware (obrót, głębia) — w liście zwykle wystarczą komendy producenta
   (zasilanie, gamma), `SLPOUT` (0x11) i `DISPON` (0x29).
2. Zapisz je w `Driver/xd-NAZWA.lua`, dopisz wiersz do `index.txt`:
   `xd-nazwa|1.0|extended-display driver: opis (a driver)|Driver/xd-nazwa.lua`.
3. Sprawdź na panelu: `sudo apt install xd-nazwa`, `extended-display set nazwa`, `extended-display start`, `extended-display test`.
4. Koloryt/orientacja: `bgr`, `invert`, `rotation`; ramka/przesunięcie: `offset_x`, `offset_y`.

Opóźnienie `delay` to 1–500 ms po komendzie (maks. 127 argumentów na komendę). Czysty kod pomocniczy (kodowanie listy, układ obrazu) ma testy:
`tests/test_xdisp.cpp`.

## 17. Tworzenie i publikowanie pakietów

Repozytorium pakietów (`formulae/`, GitHub `evilesp/cardputer-formulae`):

```
index.txt          nazwa|wersja|opis|[needs:a b|]ścieżka   (jedna formuła w linii; # to komentarz)
Formula/NAZWA.lua  program (#!lua), instalowany w /usr/local/bin/NAZWA
Driver/NAZWA.lua   sterownik, instalowany w /usr/local/lib/drivers/NAZWA.lua
README.md          tabela pakietów (po polsku)
```

Kroki dla nowej aplikacji:

1. Napisz `Formula/mojapp.lua` (z `--selftest`, wyjściem `q`/`ctrl-q`, komentarzem na górze).
2. Dopisz wiersz do `index.txt` (wersja **1.0**; opis bez znaku `|`; `needs:ui` przed ścieżką, jeśli trzeba biblioteki).
3. Dopisz wiersz do `README.md` repozytorium.
4. Dodaj testy do `tests/lua_host/run_formulae.sh` (przed linią `exit $fail`):
   ```bash
   check mojapp-selftest "$F/mojapp.lua" "selftest ok" --selftest
   KEYS_SEQ="3/enter,6/ctrl-q" check mojapp-run "$F/mojapp.lua" "tekst na ekranie"
   HEAP_KB=136 QUIT_FRAMES=600 KEYS_SEQ="1/ " check mojapp-memory "$F/mojapp.lua" "gfx"
   ```
5. `./build.sh test` — wszystko ma przejść (host uruchamia aplikację prawdziwym Lua z atrapą API; `MEM_REPORT=1` pokazuje zużycie pamięci,
   `GFX_DUMP=plik.ppm` zapisuje ostatnią klatkę).
6. `git add`, `git commit`, `git push`. Na urządzeniu: `sudo apt update && sudo apt install mojapp`.

Każda późniejsza zmiana formuły dodaje **0.0.0.1** do wersji w `index.txt` (1.0 → 1.0.0.1 → 1.0.0.2); `apt upgrade` aktualizuje także po zmianie samej treści.
Repozytorium musi być publiczne (urządzenie nie ma tokena), a pakiet to kod wykonywany na urządzeniu — instaluj tylko z zaufanych źródeł.
Własne repozytorium dodasz poleceniem `sudo add-apt-repository https://raw.githubusercontent.com/UŻYTKOWNIK/repo/main`.

## 18. Wydajność i pamięć

Zasady dla aplikacji (każda wynika z realnych awarii „not enough memory”):

- **Bez nowych tabel i domknięć w pętli klatki.** Śmieci Lua rosną szybciej, niż zbiera je GC przy ~130 KB RAM.
- **Płaskie listy liczb** zamiast tablic tabel; wartości 0–255 pakuj po 8 w jedną liczbę całkowitą.
- **Obliczaj w locie** to, czego nie trzeba trzymać (tło z funkcji szumu, nie z tablicy).
- **Duże dane czytaj z karty na żądanie** (`fs.read` części, `browser.*`), nie całe do RAM.
- **Nie ładuj bibliotek, których nie używasz** (`ui` to ok. 15 KB po odchudzeniu).
- Program powyżej 4 KB jest przechowywany bez informacji debugowych, a jego bajtkod jest buforowany w `/var/cache/lua/` — kolejne uruchomienia są szybsze i mniej fragmentują pamięć.

### GamePerformance

Wyłączenie radia WiFi zwalnia ok. 40–50 KB. `sys.performance(tryb)` / aplikacja `GamePerformance`:

| Tryb | Działanie |
|---|---|
| `off` | WiFi zawsze włączone (domyślnie) |
| `on` | radio wyłączone do odwołania (`on` nie jest zapamiętywane po restarcie) |
| `auto` | radio gaśnie, gdy aplikacja otwiera płótno (`gfx.open`), i wraca po jej zakończeniu |

Awaryjnie: gdy po otwarciu płótna zostaje mniej niż ok. 48 KB wolnej pamięci, albo gdy alokacja Lua się nie powiedzie, firmware sam wyłącza radio na czas gry
(status `wifi:game` w pasku), a po grze łączy ponownie w tle. Sieć (`net.*`) wywołana w grze budzi radio.

### Obciążenie

`top`, `uptime` (load average z zajętości rdzeni), `free`, `/proc/meminfo`. `sys.tasks()` zwraca zadania FreeRTOS z czasem pracy.

## 19. Budowanie, testy i wydawanie firmware

Wymagania: Arduino CLI, core `esp32:esp32` **3.3.12**, biblioteka **M5Cardputer 1.1.1**; Lua 5.4.7 jest w `libraries/LuaESP`.

```bash
./build.sh build                  # kompilacja (FQBN esp32:esp32:m5stack_cardputer:FlashSize=8M,PSRAM=disabled,PartitionScheme=default_8MB,CDCOnBoot=cdc)
./build.sh upload [PORT]          # kompilacja + wgranie przez USB
./build.sh monitor [PORT]         # log z USB (115200)
./build.sh test                   # testy: parser, regex, awk, linuxfmt, apt, xdisp ... + każda formuła Lua
./build.sh repo 8000              # serwer HTTP z formulae/ do testów brew bez GitHuba
```

Co sprawdzają testy: czyste moduły C++ na komputerze (`tests/test_*.cpp`: powłoka, regex, awk względem `awk` z macOS, formatowanie Ubuntu w `linuxfmt`,
tekst `apt`, kodowanie listy startowej wyświetlacza...) oraz każda formuła Lua w prawdziwym interpreterze z atrapą API (`tests/lua_host/`).

Wydanie firmware (autor): zbuduj, policz `sha256` i rozmiar `cardputer_terminal_linux.ino.bin`, utwórz Release w `evilesp/cardputer-firmware`
(`gh release create vX --prerelease` dla bety), sprawdź pobrany plik, dopisz linie do `index.txt` i `changelog.txt` (najnowsza na górze), wypchnij.

## 20. Budowa kodu

```
cardputer_terminal_linux/          szkic Arduino
  *.ino                            setup() / loop()
  sys.*                            rozruch (linie [ OK ]), pasek stanu, /proc, load average, zasilanie
  screen.*                         terminal 40×15 + płótno aplikacji; kopiuje wiersze na zewnętrzny ekran
  keys.* console.*                 klawiatura, edycja linii, historia, TAB
  shell.* shell_parse.* script.*   parser i wykonawca powłoki (potoki, pętle, funkcje, sudo)
  cmd_fs/text/text2/text3/sys/net/hw/misc/shell/archive.cpp   polecenia
  cmd_linux.cpp                    ls, date, uptime, uname, free, df, ps, top, who, systemctl ... (wyjście jak Ubuntu)
  linuxfmt.*                       czysty kod formatowania tekstu narzędzi Linuksa (testowany)
  cmd_awk.cpp awk.*                awk
  cmd_apt.cpp apt_text.*           apt, apt-get, apt-cache, dpkg, add-apt-repository (nakładka na brew)
  cmd_brew.cpp brew_api.h          menedżer pakietów
  lua_runtime.cpp                  Lua i moduły API; lua_xdisp.cpp — moduł xdisp
  gfx.*                            płótno graficzne w paskach, kopia na zewnętrzny ekran
  xdisp.* xdisp_protocol.*         zewnętrzny wyświetlacz (LovyanGFX Panel_LCD z danymi sterownika)
  perf.*                           GamePerformance, awaryjne wyłączenie radia
  net.* vfs.* hw.* periph.*        sieć, karta, GPIO, magistrale UART/SPI/I2C
libraries/LuaESP/                  Lua 5.4.7 (MIT)
formulae/                          repozytorium pakietów (osobne repozytorium git)
firmware-release/                  indeks i wydania firmware (osobne repozytorium git)
tests/                             testy C++ i host Lua
```

## 21. Rozwiązywanie problemów

| Objaw | Co zrobić |
|---|---|
| `not enough memory` w aplikacji | `wifi off` albo `sys.performance("auto")`; `free`; aplikacja bez nowych tabel w pętli |
| Urządzenie restartuje się w grze | zwykle brak pamięci po stronie C++; firmware od v0.1.0.25 wyłącza radio z góry, gdy pamięci jest mało — zaktualizuj i zgłoś numer wersji |
| `E: Unable to locate package` | `sudo apt update`; sprawdź WiFi i `apt edit-sources` |
| `Permission denied` przy `apt` | dopisz `sudo` |
| `TLS failed (block NK, free MK)` | za mało ciągłej pamięci na HTTPS — `wifi off` w grze, albo zamknij aplikacje graficzne przed pobieraniem |
| `brew`/`apt install` nie aktualizuje | `sudo apt update` najpierw; `apt upgrade` porównuje wersje i zawartość |
| Czarny drugi ekran | rozdział 11, „Diagnostyka wyświetlacza” |
| Karta nie działa | musi być FAT32; `lsblk`, `dmesg` |
| Zegar pokazuje 1970 | połącz się z WiFi (NTP) |
| `spi.open`: „every spi bus is in use” | zatrzymaj `extended-display` albo zamknij aplikację z SPI |
