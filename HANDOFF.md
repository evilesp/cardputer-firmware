# Zadanie dla Claude Code: ESP Linux na ESP32-P4

Wklej całość jako pierwszą wiadomość w Claude Code, uruchomionym w katalogu ze źródłami firmware P4
(tym z `cardputer_terminal_linux/` i `llm_engine.cpp`). Pracuj bez pytań, po kolei, i raportuj po polsku.

## Kontekst

- Urządzenie: Guition JC4880P443C, ESP32-P4, ekran dotykowy 480x800, PSRAM, `build.sh` celuje w `esp32:esp32:esp32p4`.
- Użytkownik pisze po polsku. Odpowiedzi krótkie i konkretne. Komendy do skopiowania w osobnych blokach `bash`.
- Repozytoria na GitHubie (użytkownik `evilesp`):
  - `cardputer-firmware`: tylko `index.txt`, `changelog.txt`, `FIRMWARE.md`; obrazy `.bin` są w Releases.
  - `cardputer-formulae`: pakiety Lua dla `brew`/`apt` (aplikacje, sterowniki, `Models/`).
- Aktualizacje: `SystemUpdate` (stabilna), `SystemUpdateBeta` (z betami), `SystemBackUpdate`. Urządzenie czyta `index.txt`
  (4 pola: `etykieta|sha256|rozmiar|url`, nic więcej) i sprawdza SHA-256 oraz rozmiar przed pytaniem `[y/N]`.
- Aktualna wersja w indeksie: `v0.1` (P4). Plik z kodem Cardputera `backup_original_v0.1.tar.gz` jest STARSZY niż wersja P4
  (to v0.1.0.25 pod M5Cardputer) i służy tylko jako wzór.

## Co jest już zrobione (repo `cardputer-formulae`, gałąź `claude/gifted-goldberg-wkusco`)

- `tiltracer 1.0.0.4`: `w`/`↑` gaz, `s`/`↓` hamulec, `a` `d` i strzałki skręcają, przechył z `sys.imu()` jako dodatek.
- `tiltmaze 1.0.0.2`: strzałki obok `a` `d` `w` `s`.
- `waves 1.0.0.3`: fale toczą się w perspektywie, połyski podążają za falami.
- `desktop 1.0.0.1` (beta): pulpit w Lua (okna, pasek zadań, menu, panel szybki, gesty, motywy i tapety jako pliki tekstowe,
  `desktop enable|disable|install|remove|list|status`). Dotyk jest czytany w jednym miejscu, funkcja `readPointer` w
  `Formula/desktop.lua`, przez `touch.read()`. Bez niego sterują strzałki. Przetestowane tylko w symulatorze na komputerze.
- Przegląd błędów w formułach wykonało 4 agentów; ich poprawki są w tej samej gałęzi (patrz historia commitów).

## Do zrobienia w firmware (kolejność)

1. **Moduł `touch` dla Lua** (GT911 na P4): `touch.read()` zwraca `x, y` pierwszego palca albo `nil`, bez blokowania.
   Zwracaj współrzędne w pikselach ekranu `gfx`. Dopisz do `FIRMWARE.md` (rozdział 14) i do atrapy API w `tests/lua_host`.
2. **`gfx` na 480x800**: `gfx.open()` ma zwracać prawdziwą szerokość i wysokość. Sprawdź, czy płótno mieści się w pamięci
   (PSRAM) i czy `gfx.text(x, y, tekst, kolor, rozmiar)` skaluje czcionkę 6x8. Pulpit liczy skalę z rozmiaru ekranu.
3. **Uruchamianie aplikacji z pulpitu**: pulpit wczytuje plik z `fs.read` i uruchamia go przez `load()`, po `gfx.close()`, a potem
   znów `gfx.open()`. Sprawdź, że `load` jest dostępne w piaskownicy i że `Ctrl+C` wraca do pulpitu, a nie do terminala.
4. **Start po uruchomieniu**: `desktop enable` dopisuje linię `desktop` do `~/.profile`. Zrób to też jako `systemctl set-default
   graphical.target` / `multi-user.target` w powłoce (czyta ten sam plik), plus `sudo apt install desktop` (formuła już jest w indeksie).
5. **Akcelerometr**: P4 nie ma IMU. Na razie dane z Cardputera: najprostsze to mały serwer UDP na Cardputerze (formuła `imu-server`,
   odczyt `sys.imu()`, wysyłanie `ax ay az` kilkadziesiąt razy na sekundę) i na P4 `sys.imu()` czytające z sieci, gdy brak czujnika.
   Później podmień źródło na prawdziwy czujnik (np. QMI8658 przez I2C) bez zmian w aplikacjach.
6. **Wersja**: podbij `APP_VERSION`, zbuduj `./build.sh build`, uruchom `./build.sh test`, wgraj na urządzenie, sprawdź na ekranie:
   pulpit, dotyk (tap, przeciąganie okna, przesunięcie z góry i z dołu), `tiltracer`, `waves`.
7. **Wydanie bety** (najpierw beta, potem stabilna po testach):
   - `shasum -a 256 firmware.bin` i rozmiar w bajtach,
   - GitHub Release w `evilesp/cardputer-firmware` z tagiem `v0.1.0.1-beta` i plikiem `firmware-v0.1.0.1-beta.bin`,
   - dopisz u góry `index.txt`: `v0.1.0.1(beta)|SHA|ROZMIAR|https://github.com/evilesp/cardputer-firmware/releases/download/v0.1.0.1-beta/firmware-v0.1.0.1-beta.bin`
     i `changelog.txt`: `v0.1.0.1(beta)|DATA|co się zmieniło; punkty po średniku`,
   - sprawdź na urządzeniu: `SystemUpdateBeta`, potem `UpdateVersionInfo`.

## Modele językowe (`cardputer-formulae/Models/`)

Użytkownik chce kilka modeli do uruchamiania na płytce (około 100M, 300M, 0.8B, 1.5B, w tym Qwen Coder i Llama).
Sprawdź licencje przed publikacją (dołącz plik `.LICENSE.txt` jak przy `minueza-32m-chat`). Propozycje:

- SmolLM2-135M-Instruct (Apache 2.0) i SmolLM2-360M-Instruct (Apache 2.0),
- Qwen2.5-0.5B-Instruct i Qwen2.5-Coder-0.5B / 1.5B (Apache 2.0 dla 0.5B i 1.5B; sprawdź kartę modelu),
- Llama-3.2-1B-Instruct (licencja Llama 3.2, wymaga zgody i dopisku w pliku licencji).

Uwaga o pamięci: 4-bitowe wagi to w przybliżeniu 70 MB (135M), 190 MB (360M), 270 MB (0.5B), 800 MB (1.5B). Silnik trzyma model
w PSRAM (32 MB), więc 135M może się zmieścić tylko częściowo. Większe modele wymagają czytania wag z karty SD kawałkami
(bez trzymania całości w RAM). Zmierz szybkość `ollama bench` i opisz uczciwie w README. Pobieranie z Hugging Face wymaga
konwertera do formatu firmware (`tools/` w źródłach); pliki wag trzymaj w Git LFS albo w GitHub Releases, bo zwykły plik
w repo ma limit 100 MB.

## Zasady

- Nie wymyślaj SHA-256 ani rozmiarów, tylko je policz z gotowego pliku.
- Jeśli czegoś nie da się sprawdzić na urządzeniu, napisz to wprost (jak w `FIRMWARE.md`).
- Nie zmieniaj formatu `index.txt` (cztery pola), bo starsze firmware go odrzuci.
