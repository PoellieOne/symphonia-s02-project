# Architect Notes — Post-G4/H Observations

**Serie:** S02
**Rol:** Stam / Lead Architect (Ralph + Sophia)
**Scope:** Observability & fenomenologie (geen control, geen semantiek)
**Status:** Canoniek voor *onderzoekshouding*; geen implementatie-instructies

## 0. Doel van dit document

Vastleggen wat we als mensen (Stam) hebben geleerd uit het G-H traject, zónder het systeem semantisch te belasten.
Dit document dient als:

* **anker** tegen “te vroeg labelen”
* **routekaart** voor volgende experimenten
* **contract** dat observability uitbreidingen “additief en neutraal” blijven

## 1. Afbakening

Dit document:

* introduceert **geen** nieuwe TRUTH-vormen
* introduceert **geen** control-doelen
* introduceert **geen** numerieke thresholds
* gebruikt geen labels voor “proto states” behalve **technische configuratie-IDs**

## 2. Feiten (alleen wat we hebben vastgesteld)

### 2.1 G1 — Boundary Stress

* Detectie aanwezig.
* Binding bleef uit.
* Consistent over intents en beweging.

### 2.2 G2 — Dual-Frame Overlap

* **Frame A (live/sliding window):** overlap alleen zichtbaar bij ALL-signals; CORE-overlap niet live zichtbaar.
* **Frame B (post-hoc):** CORE-overlap zichtbaar achteraf.
* Frames strikt gescheiden gehouden.

### 2.3 G3 — Proto Motion States (eerste poging)

* Post-hoc episode reconstructie op CORE niet mogelijk met toenmalige logstructuur.
* Informatief afgesloten zonder interpretatie.

### 2.4 H — Temporal Presence

* CORE-signalen loggen nu **PRESENCE_START / PRESENCE_END / PRESENCE_HEARTBEAT**.
* Additief, geen control-implicaties, geen semantiek.
* Read-only terminal dashboard toegevoegd voor live menselijke observatie (logging blijft leidend).

### 2.5 G4 — Proto Motion States (re-observatie met Presence)

* Episodes post-hoc reconstrueerbaar uit PRESENCE-events.
* 4 runs → 13 episodes.
* Episodes technisch gegroepeerd in **5 configuraties** (geen labels/semantiek).
* `ORIGIN_CANDIDATE` presence in 3/4 logs; `ORIGIN_COMMIT` niet aanwezig in deze dataset.
* Live/post-hoc niet samengevoegd; rapportage bleef observatief.

## 3. Wat dit ons “als Stam” nu toestaat (zonder betekenis)

### 3.1 Nieuwe capability is geopend

We kunnen nu **tijd-aanwezige fenomenen** op CORE post-hoc reconstrueren.
Dat maakt het eerlijk mogelijk om te spreken over:

* episodes
* duur
* overgangen
* herhaling

Zonder te doen alsof het systeem het “weet”.

### 3.2 We hebben een ‘repertoire’ zonder naam

De 5 configuraties zijn een **technische index**:

* ze mogen bestaan als *ID’s*, niet als betekenis.
* ze zijn bruikbaar als *zoekhaak* voor herhaling/vergelijking.

## 4. Kernbewustzijn: wanneer mag iets “kern” worden?

**Kernbewustzijn** (in onze betekenis) is toegestaan wanneer iets:

* herhaalbaar is over runs
* niet afhankelijk is van specifieke handbewegingen of timing
* stabiel genoeg is om niet direct te verdwijnen bij kleine variatie
* zichtbaar blijft in zowel live observatie (als aanwezigheid) als post-hoc (als episode)

Belangrijk: “kern” betekent hier **structureel observeerbaar**, niet “waar”.

## 5. ORIGIN-lijn: huidige positie

* `ORIGIN_CANDIDATE` verschijnt regelmatig als presence-fenomeen.
* `ORIGIN_COMMIT` is in deze dataset **afwezig** (feit).
* Conclusie (alleen procesmatig): we bevinden ons in een regime van **nadering zonder overgang**.
  (Geen claim waarom.)

## 6. Risico’s die we actief willen vermijden

1. **Hindsight-semantiek:** post-hoc patronen als live “binding” gaan behandelen.
2. **Label-drift:** configuratie-IDs langzaam als betekenis gaan gebruiken.
3. **Control-lek:** episodes direct naar actions mappen (te vroeg).
4. **Dashboard-leiding:** live view belangrijker maken dan de ruwe logs.

## 7. Directe volgende onderzoeksvraag (voor vervolgprompt)

> Bestaan er **overgangen** tussen de 5 configuraties die herhaalbaar zijn?
> Niet “wat betekenen ze”, maar:

* duur
* volgorde (alleen technisch)
* stabiliteit / breukmomenten
* relatie tot presence van ORIGIN_CANDIDATE (alleen co-occurrence, geen duiding)

Dit wordt de basis voor **G5** (als jij dat wilt).

## 8. Wat we (nog) níet doen

* Geen naming.
* Geen classificatie.
* Geen realtime terugkoppeling.
* Geen drempels.
* Geen “commit-forceren”.

---
