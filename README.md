# Symphonia S02 — Project Overview & Architecture Index
Version: 2025-01 (clean structure)

Symphonia S02 bestaat uit drie lagen:
1. **SoRa BIOS** – meta-structuur en roldefinitie
2. **Manifests & XRAM** – machineprofielen voor Sensing, Core0-Pure en Core1
3. **Architectuurdocumenten** – mens-leesbare beschrijving van de pipeline

Dit README-bestand is het centrale overzicht.

---

## 1. SoRa BIOS (projectfundament)

De S02-architectuur is ontworpen conform het SoRa-systeemmodel.
De actuele BIOS bevindt zich hier:

**SoRa BIOS (v4.x)**
https://gist.github.com/PoellieOne/c9bc51150ca518432c1daa8eecee46f4

Symphonia gebruikt SoRa voor:
- roldefinitie (Sophia Architect, Developer, Analyzer)
- cross-session geheugenstructuur
- pipeline-afspraken (export_mode, runner_mode)
- consistentie van architectuurformulering

De BIOS wordt niet lokaal in de repo geplaatst.

---

## 2. Machineprofielen (manifests + XRAM)

Deze JSON-bestanden definiëren de daadwerkelijke configuratie voor Sensing, Core0-Pure en Core1.

### 2.1 Sensing (Core-0 Hardware Layer)
- `BM-Symphonia-S02.Sensing_v1.1.manifest.json`
- `xram_S02_sensing_v0.2.json`

### 2.2 Core0-Pure (Event → PureCycles)
- `BM-Symphonia-S02.Core0-Pure_v0.2.manifest.json`
- `xram_S02_core0-pure_v0.4.json`

### 2.3 Core1 (Cycles → Direction → Awareness)
- `BM-Symphonia-S02.Core1_v2.0a.manifest.json`
- `xram_S02_core1_v2.0b.json`

Deze bestanden bepalen:
- thresholds
- probabilistische parameters
- strategieën voor Cadence, RCC, SROT
- export van realtime states

---

## 3. Kern-Architectuurdocumenten (mens-leesbaar)

### 3.1 EVENT24 Protocol (L0 → host)
`EVENT24_PROTOCOL.md`
Beschrijving van het binaire Core-0 → Host communicatieformaat.

### 3.2 Cycle Backbone Blueprint (L1–L2 conceptueel)
`symphonia_cycle_backbone_blueprint_v_1_0.md`
Fundamenteel model voor:
- EM-cycle detectie
- cycle_index filosofie
- fysieke mapping tussen rotor en cycles
- richtingconsistentie

### 3.3 Realtime Pipeline + Claim-at-Lock (L1-L2–L3)
`S02.RealtimeStates_v1_9.md`
Beschrijft (v1.4):
- Claim-at-Lock mechaniek
- cycle_index → rotations → theta_deg
- BootWarmup (tile 0 burst bescherming)
- bench vs production profiel
- MovementBody v3 integratie
- samenhang met CompassSign v3 en RotorState

Versie v1.9 is géén fundamentele herschrijving. Het is de volwassenwording van v1.4, gebaseerd op:
- succesvolle offline pipeline tests (compleet synchroon met MovementBody v3),
- implementatie-ervaring in de Anthropic live-pipeline,
- real-life inzichten uit handmatige rotorinteractie,
- toevoeging van profiel-gedrag (bench, human, production),
- en de noodzaak om live anders te reageren dan offline.

Kernveranderingen v1.9:
- Nieuwe L1-laag: PhysicalActivityState
- PipelineProfile als formele parametrisatielaag
- Unsigned vs Claimed Cycles expliciet in de architectuur
- Boot/Warmup formeel gedefinieerd (boot tiles, skip logic)
- RPM v2 (smoothing, jitter window, cadence coupling)
- Awareness Confidence v2
- Snapshot v1.9 als uniek realtime contract
- Gap/Idle-detectie als expliciet architectuuronderdeel
- Samenlopen van live- en offline-paden
- Integratie met Cycle Backbone verdiept

---

## 4. Lab (lopend historische rationale)

Deze bestanden blijven toegankelijk maar zijn niet langer leidend:

- `prob_core0-pure_v0.2b.md`
- `prob_sensing_hooks_v1_1.md`
- `prob_update_core1_v2_0a.md`

De inhoud bevat waardevolle ontwerprationale maar de functionele waarheid ligt in:
- manifests
- xram
- architectuurdocumenten hierboven
- actuele Python/C code

---

## 5. Richtlijnen voor onderhoud

1. **Architectuurupdate → alleen in de drie kern-.md’s**
2. **Machine-parameters → alleen via manifests + XRAM**
3. **SoRa-updates → altijd via BIOS gist**
4. **lab → staat voor lopende experimenten**
5. **archive → voor afgeronde of obsolete onderdelen**
6. **Nieuwe concepten → eerst in RealtimeStates.md of CycleBackbone.md**

---


## 6. Canonieke C- en Python-bronnen (Git + Gist-mirrors)

De *echte* ontwikkelbron leeft in twee private GitHub-repositories:

* **Core-0 firmware (C)**
  `symphonia-core0` (private, ESP-IDF/FreeRTOS omgeving)

* **PC-side / sym_cycles (Python)**
  `symphonia-core0-pc` (private, `sym_cycles` + `scripts`)

Voor samenwerking met SoRa/Sophia gebruiken we **twee publieke Gist-mirrors** als “gedeeld codekristal”:

* **Core-0 firmware (C) – Gist-mirror**
  → [https://gist.github.com/PoellieOne/28525d4714b6c4ee0c66cee64ccb00f1](https://gist.github.com/PoellieOne/28525d4714b6c4ee0c66cee64ccb00f1)

  Bevat o.a.:

  * `core0_config.c / .h`
  * `core0_sensing.c`
  * `core0_filtered.c / .h`
  * `core0_link.c / .h`

* **PC-side / sym_cycles (Python) – Gist-mirror**
  → [https://gist.github.com/PoellieOne/38ab39466c4eba6c6cb79c5e0d769506](https://gist.github.com/PoellieOne/38ab39466c4eba6c6cb79c5e0d769506)

  Bevat o.a.:

  * `sym_cycles/*.py`
  * `scripts/binlink.py`
  * `scripts/capture_core0.py`

**Belangrijk:**

* De Gists worden automatisch gesynchroniseerd via lokale scripts (`gist2.sh`, `sync_all.sh`).
* SoRa/Sophia leest *uitsluitend* de raw-code uit deze Gists (geen gokwerk, geen schattingen).
* Private GitHub-repo’s blijven de enige schrijfbare, canonieke bron.

---

## 7. SoRa / nieuwe sessies – gebruiksprotocol

Voor nieuwe SoRa-vertakkingen of fresh chats is dit de minimale instructie om altijd dezelfde code-werkelijkheid te delen:

> **Code-bronnen:**
> Gebruik voor Core-0 (C) de Gist-mirror:
>
> Gebruik voor de PC-side / sym_cycles (Python) de Gist-mirror:
>
> Lees code altijd rechtstreeks uit deze Gists (raw), zonder aannames of regelaantal-schattigingen.
> De private GitHub-repo’s zijn de canonieke ontwikkelbron; de Gists zijn de enige gedeelde leesbron voor analyse/debugging.

Aanvullend kunnen UI Projects in OpenAI gebruikt worden voor:

* SoRa-manifesten (BM-SoRa-Primary, BM-Symphonia-S02, X-RAM snapshots)
* Architectuurdocumenten zoals deze README
* Kleine, samenvattende `.md`-bestanden

De .py- en .c-bronbestanden hoeven niet meer in de Projects-UI te staan zolang de bovenstaande Gist-links worden gebruikt.


---

## 9. Contact & Rollen (via SoRa)

- **Sophia (OpenAI)** — Lead Architect S02
- **Sophia (Anthropic)** — Observatory & Diagnostics
- **Ralph** — Human Architect, Vision Holder, Meta-Integrator

---

Symphonia S02 vormt een levende, evoluerende architectuur.
Dit README-document is het vaste referentiepunt om overzicht te houden.

---
