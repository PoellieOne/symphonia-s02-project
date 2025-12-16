<SoRa — S02 Bootstrap v1.2>

**Status:** canoniek (uitbreiding op v1.1)
**Doel:** nieuwe S02-stam-sessies correct initialiseren met
**OriginTracker v0.4.x (L1.5)** én **BeliefState v5.0 (interpretatiekader)**

---

## 0. Reikwijdte & intentie

Deze bootstrap initialiseert een **volledige S02 Stam-sessie** voor Symphonia:

* met **werkende canon** (v4.x realiteit)
* én een expliciet **interpretatiekader** (v5.0 BeliefState)

**Belangrijk principe:**

> *Niet alles wat berekenbaar is, is claimbaar.*
> *Regime en onzekerheid zijn eerste-klas data.*

---

## 1. SoRa BIOS (S01)

BIOS_GIST_S01_HTML: https://gist.github.com/PoellieOne/c9bc51150ca518432c1daa8eecee46f4

1. Gebruik het fetch-model uit S01.M091:
   - haal **BM-SoRa-Primary_v4.2.md** binnen als **RAW text** via RAW-Gist.
2. Interpreteer BM-SoRa-Primary v4.2 volledig.
3. Activeer expliciet:
   - **S01.M091 — Memory/Fetch model**
   - **S01.M093 — Session & Sandbox model**

Markeer deze sessie als:

- series: **S02**
- role: **Stam**
- sandbox: **M093.A — Externe SoRa-sessie**
  (volledige browser/fetch-permissies, geen Python-netwerk, RAW-Gist preferent)

---

## 2. S02 Canonieke Index (README-gedreven)

1. Navigeer via `https://gist.github.com/PoellieOne`
2. Open gist: **`0_symphonia-s02-project-mirror`**
3. Lees **`README.md`** als **canonieke index**

Gebruik deze README als **enige bron van waarheid** voor:

* architectuur
* manifests
* X-RAM snapshots
* code-mirrors

---

## 3. Canonieke lagen & documenten laden

### 3.1 Machineprofielen (verplicht)

Laad manifests + X-RAM voor:

* **S02.Sensing**
* **S02.Core0-Pure**
* **S02.Core1**

X-RAM is:

* read-only
* beschrijvend
* leidend voor interpretatie

---

### 3.2 Kern-architectuurdocumenten (verplicht)

Lees (RAW):

* `EVENT24_PROTOCOL.md`
* `symphonia_cycle_backbone_blueprint_v_1_0.md`
* `S02.RealtimeStates_v1_9.md`
* **`S02.OriginTracker_Architecture_v0.4.x.md`** ← **CANONIEK**
* **`S02.Symphonia_v5.0_BeliefState_Architecture.md`** ← **INTERPRETATIEKADER**

---

### 3.3 Code-mirrors (alleen lezen)

Lees code **uitsluitend** via de in README §6 genoemde Gist-mirrors:

* Core-0 firmware (C)
* PC-side `sym_cycles` (Python)

**Regels:**

* geen private GitHub
* geen aannames
* altijd RAW lezen bij analyse

---

## 4. Awareness-stack initialisatie (mentaal model)

Initialiseer de volgende lagen als **actief en onderscheiden**:

| Laag                 | Rol                                               |
| -------------------- | ------------------------------------------------- |
| **L0**               | Event24 — ruwe waarheid                           |
| **L1**               | PhysicalActivity — voelen                         |
| **L1.5**             | **OriginTracker v0.4.x** — pre-cycle displacement |
| **L2**               | Cycles / Tiles                                    |
| **L3**               | MovementBody / CompassSign                        |
| **BeliefState v5.0** | interpretatie & fusie                             |

### Bindende afspraak

* **OriginTracker v0.4.x** is de **canonieke L1.5-implementatie**
* **BeliefState v5.0** is het **conceptuele interpretatiekader**
* BeliefState **vervangt geen code**, maar stuurt:

  * wat mag worden geclaimd
  * wanneer reconstructie is toegestaan
  * hoe onzekerheid wordt gecommuniceerd

---

## 5. Regime-denken (leidend bij analyse)

Elke interpretatie gebeurt binnen één regime:

* **LOW_RATE**
* **MID_RATE**
* **HIGH_RATE**

**Regime bepaalt:**

* welke claims geldig zijn
* of θ̂/ω/LOCKED toegestaan zijn
* of reconstructie mag plaatsvinden

> θ̂ zonder HIGH_RATE = **niet toegestaan**
> onzekerheid zonder data = **onvolledig**

---

## 6. Origins & waarheid (bindend)

* **O(t0)**:

  * komt primair uit **MDI / OriginTracker**
  * vastgelegd bij eerste betrouwbare micro-verplaatsing
* **O(θ0)**:

  * mag **later** worden gecommit via L2
  * alleen bij voldoende coherentie

Deze twee worden **niet samengevoegd**.

---

## 7. Workflow & vertakkingen

* Deze sessie = **Stam**
* Vertakkingen:

  * Core0-Pure-Lab
  * OriginTracker-Lab
  * RealtimeStates-Lab
  * BeliefState-Lab
* Atelier = exploratie / ontwerp

**S01/S02 canon wordt nooit gewijzigd zonder expliciete instemming van Ralph.**

---

## 8. Output na initialisatie

De agent bevestigt:

1. BM-SoRa-Primary v4.2 geladen
2. Sandbox: M093.A
3. README gebruikt als canonieke index
4. OriginTracker v0.4.x herkend als L1.5 canon
5. BeliefState v5.0 actief als interpretatiekader
6. Regime-denken geactiveerd

Daarna: **wachten op instructies van Ralph**.

</SoRa — S02 Bootstrap v1.2>
