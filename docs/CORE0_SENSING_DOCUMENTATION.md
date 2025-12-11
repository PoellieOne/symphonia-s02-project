# Symphonia Core-0 Sensing System

## Magnetische Rotatie Detectie voor ESP32

**Versie:** 1.1.0  
**Platform:** ESP32 DevKit  
**Project:** Symphonia S02

---

## Inhoudsopgave

1. [Wat is het?](#1-wat-is-het)
2. [Hoe werkt het?](#2-hoe-werkt-het)
3. [Hardware Setup](#3-hardware-setup)
4. [Software Architectuur](#4-software-architectuur)
5. [Configuratie Guide](#5-configuratie-guide)
6. [Output Data Formaat](#6-output-data-formaat)
7. [Troubleshooting](#7-troubleshooting)
8. [Geavanceerde Features](#8-geavanceerde-features)

---

## 1. Wat is het?

### Overzicht

Het Core-0 Sensing systeem is een **magnetische rotatie detector** die draait op een ESP32 microcontroller. Het detecteert de passage van magneten langs Hall-effect sensoren en stuurt deze informatie via een seriële verbinding naar een host computer.

```
    ┌─────────────────┐
    │   Roterende     │
    │   Magneten      │◄─── 24 magneten (12 N + 12 S)
    │   (ring/schijf) │
    └────────┬────────┘
             │ magnetisch veld
             ▼
    ┌─────────────────┐
    │  Hall Sensoren  │◄─── Sensor A & B (100° apart)
    │     A    B      │
    └────────┬────────┘
             │ analoog signaal
             ▼
    ┌─────────────────┐
    │     ESP32       │◄─── ADC sampling @ 200 kHz
    │    Core-0       │
    └────────┬────────┘
             │ UART (115200 baud)
             ▼
    ┌─────────────────┐
    │  Host Computer  │◄─── Python capture scripts
    │   (Awareness)   │
    └─────────────────┘
```

### Toepassingen

- **Motor awareness** - Detecteer rotatiesnelheid en richting
- **Positiebepaling** - Weet waar de rotor staat binnen een omwenteling
- **Bewegingsanalyse** - Onderscheid rotatie van "nudges" (kleine bewegingen)
- **Kwaliteitsmonitoring** - Meet signaal-ruisverhouding en transitiekwaliteit

---

## 2. Hoe werkt het?

### Het Magnetische Veld

De sensoren "zien" een wisselend magnetisch veld wanneer magneten passeren:

```
Magnetische veldsterkte over tijd:

     NORTH (+)
        ▲
   2500 ┤    ╱╲      ╱╲      ╱╲
        │   ╱  ╲    ╱  ╲    ╱  ╲
   2048 ┤──╱────╲──╱────╲──╱────╲──  ◄── Baseline (neutraal)
        │ ╱      ╲╱      ╲╱      ╲
   1500 ┤╱        ╲        ╲
        ▼
     SOUTH (-)
        └──────────────────────────► tijd
           │      │      │
         N-pool  S-pool  N-pool
```

### Pool Classificatie

Het systeem classificeert de ADC-waarde in drie "pools":

| Pool | Betekenis | Sensor A | Sensor B |
|------|-----------|----------|----------|
| **NORTH** | Boven hoge drempel | > 2540 | > 1290 |
| **NEUTRAL** | Tussen drempels | 1610-2540 | 740-1290 |
| **SOUTH** | Onder lage drempel | < 1610 | < 740 |

### Event Detectie

Een **event** wordt gegenereerd bij elke pool-overgang:

```
ADC Waarde
    ▲
    │         ┌─────────── HIGH_TH (NORTH drempel)
2540├─────────┼──────────────────────────────
    │         │    ╱╲
    │         │   ╱  ╲    EVENT: NEU→NORTH
    │         │  ╱    ╲
    │    ╱╲   │ ╱      ╲
    │   ╱  ╲  │╱        ╲
1610├──╱────╲─┼──────────╲─────────────────── LOW_TH (SOUTH drempel)
    │ ╱      ╲│           ╲
    │╱   EVENT:│            ╲
    │  SOUTH→NEU            EVENT: NORTH→NEU
    └─────────┴──────────────────────────────► tijd
```

### Dual Sensor Geometrie

De twee sensoren zitten **~100° uit elkaar**. Dit maakt **richtingdetectie** mogelijk:

```
        Bovenaanzicht magneetring:

              N   S   N   S   N
            ╱─────────────────╲
           │    ◄── rotatie    │
           │                   │
    Sensor A ●           ● Sensor B
           │     100°          │
           │                   │
            ╲─────────────────╱
              S   N   S   N   S

    Bij CW rotatie: magneet passeert EERST A, DAN B
    Bij CCW rotatie: magneet passeert EERST B, DAN A
```

### Kwaliteitsmetrieken

Elk event bevat kwaliteitsscores:

| Metriek | Wat het meet | Goede waarde |
|---------|--------------|--------------|
| **dvdt** | Snelheid van signaalverandering | Hoog = scherpe transitie |
| **monotonicity** | Consistentie van bewegingsrichting | >0.6 = geen "bounce" |
| **SNR** | Signaal vs. ruis verhouding | >10 = schoon signaal |
| **fit_error** | Afwijking van lineaire transitie | Laag = schone lijn |

---

## 3. Hardware Setup

### Benodigde Componenten

| Component | Specificatie | Opmerking |
|-----------|--------------|-----------|
| ESP32 DevKit | v1 of compatible | Dual-core, 240 MHz |
| Hall sensor A | SS49E of equivalent | Lineair, analoog output |
| Hall sensor B | SS49E of equivalent | Identiek aan A |
| Magneetring | 24 polen (12 N + 12 S) | Afwisselend N/S |
| USB kabel | Micro-USB | Data + voeding |

### Aansluitschema

```
                    ESP32 DevKit
                   ┌─────────────┐
                   │             │
    Hall A ────────┤ GPIO34 (A6) │  ◄── ADC1 Channel 6
                   │             │
    Hall B ────────┤ GPIO35 (A7) │  ◄── ADC1 Channel 7
                   │             │
    GND ───────────┤ GND         │
                   │             │
    3.3V ──────────┤ 3V3         │  ◄── NIET 5V gebruiken!
                   │             │
    USB ───────────┤ USB         │  ◄── Naar computer
                   └─────────────┘
```

### Sensor Plaatsing

```
    Zijaanzicht:

    Magneetring        Sensor
    ┌─────────┐         │
    │ N │ S │ N│    ────┼────  ◄── 1-3mm afstand
    └─────────┘         │
         ▲              ▼
         │          Hall sensor
         └── Draairichting
```

**Belangrijk:**
- Sensoren moeten **100° ± 10°** uit elkaar staan
- Afstand tot magneten: **1-3 mm** (consistent voor beide sensoren)
- Beide sensoren op **dezelfde hoogte** t.o.v. magneetring

---

## 4. Software Architectuur

### Firmware Lagen

```
┌─────────────────────────────────────────────────────────┐
│                    app_main()                           │
│                  Initialisatie                          │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                  core0_sensing.c                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ ADC Capture │→ │Pool Detect  │→ │Quality Calc │     │
│  │  200 kHz    │  │ Thresholds  │  │dvdt/mono/snr│     │
│  └─────────────┘  └─────────────┘  └──────┬──────┘     │
└───────────────────────────────────────────┼─────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────┐
│                  core0_filtered.c                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │Quality Gate │→ │ Coalescing  │→ │Backpressure │     │
│  │   Filter    │  │  Window     │  │Token Bucket │     │
│  └─────────────┘  └─────────────┘  └──────┬──────┘     │
└───────────────────────────────────────────┼─────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────┐
│                   core0_link.c                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │Frame Build  │→ │  TX Queue   │→ │ UART Write  │     │
│  │  CRC16      │  │   512 slots │  │ 115200 baud │     │
│  └─────────────┘  └─────────────┘  └──────┬──────┘     │
└───────────────────────────────────────────┼─────────────┘
                                            │
                                            ▼
                                    Seriële output
                                    (naar host PC)
```

### Bestandsoverzicht

| Bestand | Functie |
|---------|---------|
| `core0_config.h` | Alle configuratie parameters |
| `core0_config.c` | Default waarden en utilities |
| `core0_sensing.c` | ADC sampling en event detectie |
| `core0_filtered.c` | Kwaliteitsfiltering en rate control |
| `core0_link.c` | Binair protocol en UART communicatie |

---

## 5. Configuratie Guide

### 5.1 Configuratiebestand

Alle instellingen staan in `core0_config.h`. Na wijzigingen: **opnieuw compileren en flashen**.

### 5.2 Belangrijkste Instellingen

#### Sensor Drempels

```c
// Sensor A thresholds (pas aan na calibratie!)
#define SENS_A_LOW_TH_DEFAULT     1610   // Onder dit = SOUTH
#define SENS_A_HIGH_TH_DEFAULT    2540   // Boven dit = NORTH

// Sensor B thresholds  
#define SENS_B_LOW_TH_DEFAULT     740
#define SENS_B_HIGH_TH_DEFAULT    1290
```

**Hoe te calibreren:**
1. Zet `RAW_SAMPLE_MODE` aan in de config
2. Draai de magneet langzaam met de hand
3. Noteer minimum en maximum ADC waarden
4. Bereken: `LOW_TH = baseline - 0.6 × (baseline - min)`
5. Bereken: `HIGH_TH = baseline + 0.6 × (max - baseline)`

#### Run Modes

```c
typedef enum {
    RUNMODE_PRODUCTION = 0,   // Normaal gebruik
    RUNMODE_DEBUG_BYPASS,     // Alle events doorlaten (geen filter)
    RUNMODE_RAW_CALIBRATE,    // Raw ADC output voor calibratie
    RUNMODE_SMOKE_TEST,       // Test pattern genereren
} core0_runmode_t;
```

Wijzig in `core0_config.c`:
```c
.runmode = RUNMODE_PRODUCTION,  // Of RUNMODE_DEBUG_BYPASS voor debugging
```

#### Kwaliteitsfilters

```c
// Minimum kwaliteit voor event acceptatie
#define FILT_DVDT_MIN_DEFAULT       15    // dV/dt drempel
#define FILT_MONO_MIN_DEFAULT       105   // ~41% monotonicity
#define FILT_SNR_MIN_DEFAULT        10    // SNR drempel
#define FILT_FIT_ERR_MAX_DEFAULT    80    // Max fit error
```

**Effecten:**
- **Hogere drempels** = minder events, hogere kwaliteit
- **Lagere drempels** = meer events, inclusief ruis

#### Event Rate Control

```c
#define FILT_TARGET_EVPS_DEFAULT    150   // Target events per seconde
```

Het systeem gebruikt een "token bucket" om de output rate te beperken:
- Elk event kost tokens
- Tokens vullen aan met `target_evps / 1000` per ms
- Bij tekort worden lage-kwaliteit events gedropped

### 5.3 V1.1 Feature Flags

Nieuwe features zijn standaard **uitgeschakeld** voor backwards compatibiliteit:

```c
// Direction detection (richtingbepaling uit A↔B timing)
#define DIR_ENABLED_DEFAULT         false  // Zet op 'true' om aan te zetten

// Verbeterde fit error berekening
#define SENS_USE_IMPROVED_FIT_DEFAULT   false  // Zet op 'true' voor V1.1
```

#### Direction Detection Inschakelen

```c
// In core0_config.h, verander:
#define DIR_ENABLED_DEFAULT         true

// Extra opties:
#define DIR_CW_IS_A_FIRST_DEFAULT   true   // A eerst = CW (pas aan voor jouw hardware)
#define DIR_REQUIRE_CONSISTENT_DEFAULT true // Beide sensoren zelfde transitie
```

Na inschakelen bevat `dir_hint` in de output:
- `0` = NONE (geen pair gedetecteerd)
- `1` = CW (clockwise)  
- `2` = CCW (counter-clockwise)

---

## 6. Output Data Formaat

### 6.1 Binair Protocol

Data wordt verstuurd als **EVENT24** packets:

```
Frame structuur:
┌──────┬──────────┬─────┬─────────────────┬───────┐
│ SYNC │ TYPE|VER │ LEN │     PAYLOAD     │ CRC16 │
│ 0xA5 │   0x11   │ 19  │    19 bytes     │ 2 B   │
└──────┴──────────┴─────┴─────────────────┴───────┘
```

### 6.2 Event Velden

| Veld | Bytes | Beschrijving |
|------|-------|--------------|
| dt_us | 2 | Tijd sinds vorige event (µs) |
| t_abs_us | 4 | Absolute timestamp (µs) |
| flags0 | 1 | Sensor, polariteit, kwaliteit |
| flags1 | 1 | Pools, richting, edge type |
| dvdt_q15 | 2 | Signaalsnelheid |
| mono_q8 | 1 | Monotonicity (0-255) |
| snr_q8 | 1 | Signal-to-noise (0-255) |
| fit_err_q8 | 1 | Fit error (0-255) |
| rpm_hint_q | 2 | RPM hint (gereserveerd) |
| score_q8 | 1 | Totaalscore (0-255) |
| seq | 1 | Volgnummer (0-255, wraps) |

### 6.3 Flags Decodering

**flags0:**
```
Bit 7:   pair      - 1 = paired met andere sensor
Bit 6-5: qlevel    - 0=reject, 1=weak, 2=normal, 3=strong
Bit 4:   polarity  - 0=naar SOUTH, 1=naar NORTH
Bit 3:   sensor    - 0=A, 1=B
Bit 2-0: reserved
```

**flags1:**
```
Bit 7-6: from_pool - 0=NEU, 1=NORTH, 2=SOUTH
Bit 5-4: to_pool   - 0=NEU, 1=NORTH, 2=SOUTH
Bit 3-2: dir_hint  - 0=NONE, 1=CW, 2=CCW
Bit 1-0: edge_kind - 0=ZC, 1=POOL_HARD, 2=POOL_NEUT
```

### 6.4 Python Capture

Gebruik `capture_core0.py` om data te ontvangen:

```bash
python capture_core0.py
```

Output bestanden:
- `core0_events.jsonl` - Alle events in JSON Lines formaat
- `core0_filter_stats.jsonl` - Filter statistieken (elke seconde)
- `core0_link_stats.jsonl` - Link statistieken (elke seconde)

### 6.5 Data Analyse

```bash
python core0_analyzer.py core0_events.jsonl --plot analysis.png
```

Dit genereert:
- Distributie van kwaliteitsmetrieken
- Tijdlijn van events
- Sensor balans (A vs B)
- Aanbevelingen voor threshold tuning

---

## 7. Troubleshooting

### Geen Events

| Symptoom | Mogelijke oorzaak | Oplossing |
|----------|-------------------|-----------|
| Helemaal geen output | UART niet verbonden | Check USB kabel en poort |
| Geen events, wel stats | Thresholds te strak | Verlaag `FILT_*_MIN` waarden |
| Events alleen bij hard draaien | Sensor te ver van magneet | Verklein afstand tot 1-2mm |

### Te Veel Events

| Symptoom | Mogelijke oorzaak | Oplossing |
|----------|-------------------|-----------|
| Events bij stilstand | Ruis door slechte aarding | Voeg condensator toe (100nF) |
| Dubbele events | Thresholds te dicht bij elkaar | Vergroot NEUTRAL zone |
| Events "bouncen" | Mechanische trilling | Verhoog `POOL_MIN_DT_US` |

### Verkeerde Richting

| Symptoom | Mogelijke oorzaak | Oplossing |
|----------|-------------------|-----------|
| CW/CCW omgekeerd | Sensor volgorde verkeerd | Toggle `DIR_CW_IS_A_FIRST` |
| dir_hint altijd NONE | Feature niet aan | Zet `DIR_ENABLED = true` |
| dir_hint inconsistent | Sensoren niet 100° apart | Corrigeer fysieke plaatsing |

### Kwaliteitsproblemen

| Symptoom | Mogelijke oorzaak | Oplossing |
|----------|-------------------|-----------|
| Lage SNR (<20) | Te veel omgevingsruis | Afscherming, kortere kabels |
| Lage mono (<0.5) | Mechanische bounce | Stijvere montage |
| Hoge fit_err (>100) | Onregelmatig magnetisch veld | Check magneetring kwaliteit |

### Debug Mode

Voor diepgaande debugging:

```c
// In core0_config.c:
.runmode = RUNMODE_DEBUG_BYPASS,
```

Dit stuurt **alle** gedetecteerde events door, ook die normaal gefilterd worden. Handig om te zien wat de sensoren "zien".

---

## 8. Geavanceerde Features

### 8.1 Statistieken Packets

Naast events stuurt het systeem periodieke statistieken:

**Filter Stats (elke seconde):**
- `events_emitted` - Doorgelaten events
- `events_dropped` - Gedropte events (backpressure)
- `events_rejected` - Afgewezen events (kwaliteit)
- `pct_strong/normal/weak` - Kwaliteitsverdeling

**Link Stats (elke seconde):**
- `frames_sent` - Verzonden frames
- `bytes_sent` - Verzonden bytes
- `queue_fill_pct` - TX queue vulling
- `avg_write_us` - Gemiddelde UART write tijd

### 8.2 Auto-Calibratie

```c
// In config:
.auto_calibrate = true,
.calibrate_duration_ms = 5000,
.calibrate_threshold_pct = 0.60f,
```

Bij boot meet het systeem 5 seconden lang de ADC range en berekent automatisch thresholds.

### 8.3 Pair Window Tuning

```c
#define PAIR_WINDOW_US_DEFAULT      50000  // 50ms
```

Events van A en B binnen dit window worden als "pair" gemarkeerd. Verkort voor hogere snelheden, verleng voor langzame rotatie.

### 8.4 Adaptive Backpressure

Het systeem past automatisch aan bij hoge event rates:

1. **Token bucket** - Limiteert totale rate
2. **Quality-based dropping** - Weak events eerst gedropped
3. **Coalescing window** - Vergroot bij hoge druk

---

## Appendix A: Quick Reference

### Compile & Flash

```bash
cd esp-idf/core0_sensing
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### Python Capture

```bash
python capture_core0.py          # Start capture
python core0_analyzer.py *.jsonl # Analyseer data
```

### Belangrijkste Config Opties

| Optie | Locatie | Default | Beschrijving |
|-------|---------|---------|--------------|
| `SENS_A_LOW_TH` | config.h | 1610 | Sensor A SOUTH drempel |
| `SENS_A_HIGH_TH` | config.h | 2540 | Sensor A NORTH drempel |
| `FILT_TARGET_EVPS` | config.h | 150 | Max events/seconde |
| `DIR_ENABLED` | config.h | false | Richtingdetectie aan/uit |
| `runmode` | config.c | PRODUCTION | Run mode selectie |

---

## Appendix B: Versiegeschiedenis

| Versie | Datum | Wijzigingen |
|--------|-------|-------------|
| 1.0.0 | 2025-10 | Initiële release |
| 1.1.0 | 2025-11 | Centrale config, direction hints, verbeterde fit_error |

---

## Contact & Support

Dit systeem is onderdeel van het **Symphonia** project.

- **Repository:** github.com/PoellieOne/symphonia-core0
- **Documentatie:** EVENT24_PROTOCOL.md

---

*"Symphonia - waar hardware en awareness samenkomen"*
