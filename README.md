# PiggyMeter × Iskra MT174 — ESPHome / Home Assistant

Reading an **Iskraemeco MT174** electricity meter over its optical (IR) port with a
[PiggyMeter](https://aquaticus.info/meter.html) head and **ESPHome**, exposing full data
plus analytics to Home Assistant — no cloud, fully local.

> **This is a hardware-specific, out-of-the-box config for the PiggyMeter board
> (Lolin C3 mini / ESP32-C3).** The pin map and the RX-tap timing fix are tied to the
> PiggyMeter's wiring — it is *not* a generic ESP32 sketch. The runtime "universal" knobs
> below tune the *meter protocol*, not the board.

*(Polska wersja — patrz sekcja [🇵🇱 Po polsku](#-po-polsku) na końcu.)*

The **MT174** is widely deployed in **Poland** (ENEA, Tauron, Energa areas) and less common
in Western Europe. As far as I can tell this is the first publicly documented PiggyMeter
integration for this specific model, so the notes below may save others a lot of trial and error.

## Hardware
- **PiggyMeter** optical head (IR phototransistor + IR LED), IEC 62056-21 optical probe.
- **Lolin C3 mini** (ESP32-C3). Pin map used here:
  - `RX = GPIO4` (phototransistor, meter → ESP)
  - `TX = GPIO5` (IR LED, ESP → meter)
  - `LED = GPIO7`

## Key findings (the non-obvious bits)

1. **Protocol: mode C.** The MT174 does **not** reliably push data on its own (mode D was
   silent for minutes). It answers the mode-C request `/?!`, identifies as `/ISk5MT174-...`,
   and negotiates **up to 9600 baud** (full readout ~15 s instead of minutes at 300 baud).

2. **The `uart.debug` RX tap is REQUIRED — it is not just for debugging.**
   The optical head is half-duplex, so the `/?!` request echoes back into RX. Without an RX
   tap the component mis-parses that echo and logs `Invalid identification frame` on every
   cycle. Adding a non-consuming `debug:` RX tap changes the UART read timing just enough
   that the echo is handled and mode C works. With the tap: 4/4 clean cycles; without it: 0.
   Keep the tap in the `uart:` block.

3. **UART framing:** start at **300 baud, 7E1**, negotiates to 9600. `rx_buffer_size: 2048`.

4. **Pin order matters.** `rx=GPIO4 / tx=GPIO5`. Swapping them yields silence / invalid frames.

## What you get in Home Assistant
- **Energy:** active import total + T1/T2, active export (0 on a non-PV meter), reactive Q1–Q4.
- **Power:** average power, max demand (+ timestamp).
- **Diagnostics:** meter time/date (formatted), serial, firmware, power-failure count, event log.
- **Analytics (computed on-device):** `tan φ` inductive & capacitive from cumulative registers,
  a **reactive-penalty alarm** (`tan φ > 0.4`) — this surfaced a real **capacitive
  over-compensation** (`tan φ ≈ 0.42`) on the test meter — and a max-demand-vs-contracted-power
  warning (settable contracted power).
- **Controls:** data-polling on/off, alarm enables, raw-log toggle.
- **Universal runtime config (no reflash):** a `profile` selector (Iskra MT174 / Custom) plus
  live `mode / max-baud / update-interval / receive-timeout / retries` selects & numbers, so you
  can integrate other IEC 62056-21 meters by trial and error from the HA UI.
  *Reflash-only (ESPHome compile-time): UART parity/data/stop bits, RX/TX pins, base baud.*

## OBIS codes observed on this MT174
| OBIS | Meaning | Example |
|---|---|---|
| `1-0:1.8.0` / `.1` / `.2` | Active energy import: total / T1 / T2 | `23753.xxx kWh` |
| `1-0:2.8.0..2` | Active energy export | `0.000 kWh` |
| `1-0:5.8.0` / `6.8.0` / `7.8.0` / `8.8.0` | Reactive energy Q1 / Q2 / Q3 / Q4 | `kvarh` |
| `1-0:1.5.0` | Average power | `0.6xx kW` |
| `1-0:1.6.0` | Max demand (+ timestamp, group 2) | `2.880 kW (YYMMDDhhmm)` |
| `1-0:0.9.1` / `0.9.2` | Meter time (HHMMSS) / date (DYYMMDD) | `150408` → 15:04:08 |
| `1-0:0.0.0` | Serial number | `82634130` |
| `1-0:0.2.0` | Firmware/parameter version | `1.06` |
| `0-0:C.7.0` | Power-failure count | `51` |
| `0-0:C.51.1` / `C.51.2*NN` | Event count / event timestamps | historic |

## Setup
1. Install the [aquaticus/esphome-iec62056](https://github.com/aquaticus/esphome-iec62056)
   external component (referenced in the YAML). Tested with **ESPHome 2026.7**.
2. `cp secrets.yaml.example secrets.yaml` and fill in your Wi-Fi + generated keys.
3. Adjust `rx_pin` / `tx_pin` if your board differs.
4. Compile & flash. Watch the ESPHome log: you should see
   `Meter identification: '/ISk5MT174-...'` → `negotiated baud rate 9600` → entities populate.

## Credits
Built on [PiggyMeter](https://aquaticus.info/meter.html) and the
[aquaticus/esphome-iec62056](https://github.com/aquaticus/esphome-iec62056) component.
Integration notes & analytics for the Iskra MT174 by the repo author.

---

## 🇵🇱 Po polsku

Odczyt licznika energii **Iskraemeco MT174** przez port optyczny (podczerwień) głowicą
**PiggyMeter** i **ESPHome**, z pełnymi danymi i analityką w Home Assistant — lokalnie, bez chmury.

> **To gotowa konfiguracja pod sprzęt PiggyMeter (Lolin C3 mini / ESP32-C3).** Mapa pinów
> i sztuczka z tapem RX są związane z okablowaniem PiggyMetera — to *nie* jest generyczny
> szkic na dowolne ESP32. Strojenie „uniwersalne" w locie dotyczy *protokołu licznika*, nie płytki.

MT174 jest **popularna w Polsce** (ENEA, Tauron, Energa), rzadsza na Zachodzie. Z tego co udało
się ustalić, to pierwszy publicznie udokumentowany przypadek integracji tego modelu z PiggyMeterem.

### Sprzęt
- Głowica **PiggyMeter** (fototranzystor IR + dioda IR), sonda optyczna IEC 62056-21.
- **Lolin C3 mini** (ESP32-C3). Piny: `RX = GPIO4` (fototranzystor), `TX = GPIO5` (dioda IR), `LED = GPIO7`.

### Kluczowe odkrycia
1. **Protokół: mode C.** MT174 **nie** wypycha danych sam (mode D milczał minutami). Odpowiada na
   `/?!`, przedstawia się jako `/ISk5MT174-...` i negocjuje **do 9600 baud** (pełny odczyt ~15 s).
2. **Tap `uart.debug` RX jest WYMAGANY — to nie tylko debugowanie.** Głowica jest półdupleksowa,
   więc `/?!` wraca echem do RX. Bez tapu komponent bierze echo za błędną ramkę
   (`Invalid identification frame`) w każdym cyklu. Nieprzechwytujący tap RX zmienia timing
   odczytu UART tak, że echo jest poprawnie odrzucane. Z tapem: 4/4 czyste cykle; bez: 0.
3. **Ramka UART:** start **300 baud, 7E1**, negocjacja do 9600. `rx_buffer_size: 2048`.
4. **Kolejność pinów ma znaczenie:** `rx=GPIO4 / tx=GPIO5`. Zamiana = cisza / błędne ramki.

### Co dostajesz w Home Assistant
Energia czynna (pobór total/T1/T2, oddana), bierna Q1–Q4; moc i moc maksymalna; diagnostyka
(czas/data sformatowane, serial, firmware, licznik zaników, log zdarzeń); **analityka on-device**:
`tan φ` indukcyjny/pojemnościowy + **alarm energii biernej** (`tan φ > 0,4` — na liczniku testowym
wykrył realną **nadkompensację pojemnościową**, `tan φ ≈ 0,42`) oraz ostrzeżenie o mocy maksymalnej
względem umownej; sterowanie (pobieranie on/off, włączniki alarmów, log surówki); **konfiguracja
runtime bez flashowania** (profil Iskra MT174 / Custom + tryb/baud/interwał/timeout/retry w HA).
*Tylko reflash: parzystość/bity/stop, piny RX/TX, bazowy baud.*

### Instalacja
1. Komponent [aquaticus/esphome-iec62056](https://github.com/aquaticus/esphome-iec62056)
   (wskazany w YAML). Testowane z **ESPHome 2026.7**.
2. `cp secrets.yaml.example secrets.yaml` i uzupełnij Wi-Fi + wygenerowane klucze.
3. Skoryguj `rx_pin` / `tx_pin`, jeśli masz inną płytkę.
4. Skompiluj i wgraj. W logu ESPHome zobacz `Meter identification: '/ISk5MT174-...'` →
   `negotiated baud rate 9600` → encje się zapełniają.
