# Symphonia Architecture — Cycle‑Backbone Edition v1.0

## 1. Inleiding
Deze blauwdruk beschrijft de nieuwe, cyclische architectuur voor Core‑0 sensing en Motor Awareness binnen Symphonia. Het model baseert zich op fysieke rotor‑cycli (S → O → N of N → O → S) en projecteert alle events, pure steps en awareness‑metingen tegen een tijd‑geordende cyclische "backbone". Dit levert een robuust, fysiek‑consistent motorbewustzijn op.

Doelstellingen:
- Volledig gebruik maken van 3‑punts cycles (fysieke rotor-truth).
- Interne coherentie herstellen door sequentie‑wetten af te dwingen.
- Stereo‑timing (A vs B) integreren voor absolute plaatsbepaling.
- Direction, motion en state vaststellen vanuit structurele consistentie.
- Alle afgeleide lagen (awareness, pure, bench) verbeteren via deze backbone.

## 2. Historische context
### 2.1 Core‑0 Event24
- Hoge betrouwbaarheid van flankdetectie.
- Ruisreductie via dvdt, snr, mono, fit_err.
- Atomaire events zonder structureel verband.

### 2.2 Pure Steps
- Pair-detectie (A→B of B→A) en nudges.
- Richting afgeleid uit eventvolgorde.
- Geen cyclische context of fysieke constraints.

### 2.3 Awareness v1.0
- Berekening van direction, dir_conf, rpm.
- Motion state: STATIC / SLOW / MOVING.
- Geen kennis van cycle-structuur, geen sequentie‑validatie.

### 2.4 Huidige grenzen
- Direction werkt goed bij nette runs, maar degradeert bij mixed/pauzes.
- Events worden niet vergeleken met fysieke rotorwetten.
- Cycle‑informatie wordt niet operationeel benut.
- Geen tijdsconsistent model om ruis te corrigeren.

## 3. Nieuwe conceptuele fundering
### 3.1 Rotor-cycli als fysieke DNA‑codons
Een 3‑punts sequence (bijv. S→O→N) vormt een minimale rotor‑waarheidseenheid (“cycle”). Deze structuur is hardware‑gedicteerd en altijd geldig.

Eigenschappen:
- Elke cycle heeft één richting: **cycle_up** (N→O→S) of **cycle_down** (S→O→N).
- De volgorde is onveranderlijk: SN, NS, SS, NN, OO zijn fysiek onmogelijk.
- Elke cycle heeft een centrum‑tijd t_center_us.
- Cycle‑duur dt_us is stabiel binnen spreiding.
- Stereo‑ouders A en B delen dezelfde cyclische waarheid met een meetbaar Δt.

### 3.2 De Cycle‑Backbone
- Een sequentiële keten van gesorteerde cycles.
- Verbindt elke cycle met predecessor/successor.
- Legt verwachte timing, verwachte richting en stereo‑lag vast.
- Vormt het structurele referentiekader van alle andere lagen.

### 3.3 Event‑projectie
Alle pure steps en awareness‑events krijgen:
- cycle‑match score: past het event in de lokale cycle?
- stereo‑score: klopt Δt met verwacht model?
- direction‑score: pure prediction vs backbone.
- timing‑score: ligt het event binnen vertrouwde dt‑range?

### 3.4 Global Direction Solver
Richting wordt niet langer puur statistisch bepaald, maar via:
- backbone‑consistency (primair)
- pure‑steps weighted
- awareness-weighted
- stereo‑dominantie
- sequentie‑validiteit

## 4. Architectuur Overzicht
Lagen:
1. **L0: Raw Events** (Event24)
2. **L1: Pure Steps** (paar/nudge + baseline richting)
3. **L2: Cycle Builder** (detecteert cycles per sensor)
4. **L3: Cycle Backbone** (tijd‑geordende rotorstructuur)
5. **L4: Event Projection** (map pure+aware → backbone)
6. **L5: Global Direction Solver** (fysisch + statistisch)
7. **L6: BenchDirection v2** (nieuwe evaluatielaag)

## 4.3  TileSpan-Resonantie (S02.M082)

### S02.M082 — TileSpan-Resonantie (0.6-cycles tiles)

**Context**

* Rotor: 24 magneten → 12 N–S paren → **12 EM-cycles per omwenteling**.
* 1 EM-cycle = 360° / 12 = **30° mechanisch**.
* Twee Hall-sensoren (A/B) met fysieke offset ≈ **10° mechanisch**.
* Motor awareness stack (S02) werkt via:

  * **L1**: 3-punts cycles (NEU/N/S) per sensor
  * **L2**: PhaseTiles (tile_duration_us, dt_ab_us)
  * **L2.5**: InertialCompass (window over tiles)
  * **L3**: MovementBody v3 (rotaties, θ, rpm, flow, motion)

**Kernbeslissing**

De canonische tile-span voor S02 is:

> `tile_span_cycles = 0.6`

met:

* `cycles_per_rot = 12.0`
* → **tile_deg ≈ 0.6 × 30° = 18° mechanisch**
* → **tiles_per_rot ≈ 360 / 18 ≈ 20 tiles per omwenteling**

In combinatie met:

* `compass_window_tiles = 20`
* `compass_alpha = 0.95`
* `compass_threshold_high = 0.6`
* `compass_threshold_low = 0.25`

**Hardware-afhankelijk inzicht**

* Sensor-offset ≈ 10° → we willen tiles die **breder zijn dan de offset**, zodat A en B binnen dezelfde tile-regio kunnen vallen.
* Met `tile_span_cycles = 0.6`:

  * tile ≈ 18° → **A en B overlappen in dezelfde tile**, maar:
  * de offset is groot genoeg om een duidelijke **signed dt_ab_us** te meten.
* Met `tile_span_cycles = 1.0`:

  * tile = 30° → A en B passen ook, maar dt_ab wordt relatief “klein” in verhouding tot de tile, en de CW/CCW-statistiek wordt minder uitgesproken → inertial compass bouwt een zwakker global_score op.

**Resonantie in de stack**

`tile_span_cycles = 0.6` zorgt voor een natuurlijke resonantie:

* **≈ 20 tiles per omwenteling**
* **kompas-window = 20 tiles**
  → één compass-window ≈ één volledige rotoromwenteling.
* De offline en realtime engines vallen hiermee samen:

  * PhaseTiles v3 (offline)
  * Realtime tiles (0.6)
  * InertialCompass v3
  * MovementBody v3

Dit is empirisch bevestigd:

* Bij `tile_span_cycles = 0.6` + bovenstaande compass-settings:

  * `direction_global_effective = CW`
  * `direction_global_conf ≈ 0.896`
  * `rotations ≈ 3.0833`
  * `theta_deg = 30.0`
  * `rpm_est ≈ 58`
  * `flow_state = FLOW`
* Deze waarden komen overeen met de **offline benchmark-resultaten** voor dezelfde EVENT24 dataset.

**Praktische regels**

1. **S02-canon (RealtimePipeline v1.1)**

   * Gebruik standaard:

     * `tile_span_cycles = 0.6`
     * `cycles_per_rot = 12.0`
     * `compass_window_tiles = 20`
     * `compass_alpha = 0.95`
     * `compass_threshold_high = 0.6`
     * `compass_threshold_low = 0.25`
2. **Intuïtief anker**

   * Denk aan tiles als “plaatjes” van ≈ 18° rotorhoek:

     * breed genoeg om beide sensoren tegelijk “te zien”,
     * smal genoeg om hun tijdverschil (dt_ab) scherp te voelen.
3. **Experimenten**

   * Afwijken van `tile_span_cycles = 0.6` mag, maar:

     * altijd expliciet labelen als experiment,
     * verwachten dat dt_ab-signatuur, global_score en lock-gedrag kunnen verschuiven,
     * offline ↔ realtime-vergelijking opnieuw valideren.

## 5. Datastromen
### 5.1 L0 → L1
- Event24 → pure step.
- Output: CW, CCW, NONE + quality + phase_tag.

### 5.2 L1 → L2 Cycle Builder
- Combineert events binnen dezelfde sensor.
- Detecteert sequenties S→O→N of N→O→S.
- Markeert cycle type, dt_us, t_center_us.

JSON per cycle:
```
{
  "sensor": 0,
  "cycle_type": "cycle_up",   // of cycle_down, cycle_mixed
  "t_center_us": 22123456,
  "dt_us": 74872,
  "phase_pos": 2,
  "edges": [ {…}, {…}, {…} ]
}
```

### 5.3 L2 → L3 Backbone
- Sorteer op t_center.
- Bepaal expected_next_type.
- Bepaal dt_range via histogram.
- Bepaal stereo Δt_range.

Backbone node:
```
{
  "cycle_id": 42,
  "sensor": "A/B",
  "cycle_type": "cycle_down",
  "t_center": 21710959,
  "expected_next": "cycle_up",
  "dt_range": {"min": 50000, "max": 110000},
  "stereo": {"dt_min": -3000, "dt_max": 4000}
}
```

### 5.4 L3 → L4 Event Projection
Event‑mapping:
- time‑nearest cycle
- direction match? (pure vs cycle)
- stereo match? (A vs B cycle timings)
- timing tolerance match?

Output per projected event:
```
{
  "step_id": 12,
  "t_us": 21758226,
  "proj_cycle": 42,
  "score": {
      "cycle_phase": 0.8,
      "stereo": 0.9,
      "direction": 1.0,
      "timing": 0.7,
      "total": 0.85
  }
}
```

### 5.5 L4 → L5 Global Direction
Gebruik alle projected events:
- Weeg op basis van total-scores.
- Combineer met awareness (dir_conf).
- Combineer met pure (quality).
- Backbone zelf heeft direction_sign.

Resultaat:
```
{
  "global_direction": "CW",
  "strength": "very_strong",
  "confidence": 0.94
}
```

### 5.6 L5 → L6 Bench v2
- Genereert genormaliseerde metrics: z‑scores, dominance-ratio, consistency.
- Produceert een duidelijk verdict.

## 6. Implementatie Roadmap
### Fase 1 – Fundament
- Cycle Builder v1.0
- Cycle Backbone v1.0
- Pure Step integratie (cycle context)

### Fase 2 – Event Projection & Solver
- Event Projection engine
- Global Direction Solver (scores + fusion)

### Fase 3 – Bench v2
- Nieuwe evaluatie metrics
- Visualisatie van backbone en projections

### Fase 4 – Awareness v2.0
- Awareness wordt cycle-informed
- Nieuwe states gebaseerd op backbone-consistency

## 7. Conclusie
Deze Cycle‑Backbone Architectuur vormt de nieuwe ruggengraat van Symphonia’s motorbewustzijn. Voor het eerst verbinden we alle meetniveaus (raw → pure → cycle → backbone → awareness → bench) in één fysisch‑consistent model. Dit vergroot precisie, robuustheid en interpretatievermogen aanzienlijk.

De volgende stap is uitvoering van Fase 1: implementatie van de Cycle Builder en Backbone modules en integratie hiervan met pure_steps.

