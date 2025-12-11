# S02.Core0-Pure v0.2
## Rol
Core0-Pure is de proprioceptieve tussenlaag tussen ruwe sensorflanken (Core-0 Sensing)
en bewuste motor state (Core-1 Awareness).

Core0-Pure:
- beoordeelt individuele flanken op kwaliteit,
- probeert per flank al een lokale richtingshint af te leiden,
- matcht flanken tussen A en B tot rotatiestappen,
- en levert gestabiliseerde "PureStep"-objecten aan Awareness.

Dit document beschrijft de besluitlogica en onzekerheidsbehandeling.

(In deze versie (v0.2b) zijn de addenda S02.M082 en S02.M083 geïntegreerd. Deze forward spec beschrijft de volgende evolutie van Core0-Pure waarin lokale fase-informatie wordt geëxporteerd via theta_local_deg.)

---

## 1. Eventkwalificatie (Quality Gate)
Voor elk Core-0 RAW event wordt beslist of dit flankmoment fysiek betekenisvol is.

Reject een event als:
- `monotonicity < 0.6` (te weinig eenrichtingskarakter).
- `context_snr < 2.0` (te weinig signaal t.o.v. lokale ruis).
- `from_pool -> to_pool` is fysiek onrealistisch (|Δpool| > 1, bv. SOUTH direct naar NORTH)

Aanname: `from_pool` / `to_pool` ∈ {SOUTH, NEUTRAL, NORTH}.

Events die slagen worden `recent_valid_flanks`.

Rationale:
- Core-1 Awareness mag geen ruis eten.
- "False flank rate < 0.5%" uit de S02-acceptatiecriteria wordt hier afgedwongen.

---

## 2. Single-sensor Directional Hint (S02.M082)
Voor elk gevalideerd flank-event wordt een lokale richtingshint (`dir_hint`) bepaald.

Heuristiek:
- Gebruik `dvdt` teken en/of (from_pool -> to_pool) richting.
- SOUTH -> NEUTRAL -> NORTH met stijgende veldsterkte ⇒ `dir_hint = "up"`.
- NORTH -> NEUTRAL -> SOUTH met dalende veldsterkte ⇒ `dir_hint = "down"`.
- Onbeslist ⇒ `dir_hint = "none"`.

We registreren per sensor (A / B):
`last_dir_hint[sensor] = { t_us, dir_hint }`

Interpretatie:
Dit is de eerste vorm van richtingservaring ("ik voel spanning in deze richting"),
nog vóór we twee sensoren hebben samengelegd.
Dit geldt ook wanneer slechts één sensor actief is of de rotor extreem traag beweegt.

## 3. Pairing tot rotatiestappen
We houden een pairing_window aan van ~30 ms (30000 µs).
Doel: langzaam met de hand draaien moet nog steeds als één mechanische stap tellen.

Regel:
- Als er een geldige flank van A komt en er is een recente flank van B binnen pair_window,
vorm een pair-step (A_then_B of B_then_A).
- Als er een geldige flank van B komt en er is een recente flank van A binnen pair_window,
idem.
- Anders markeren we de flank als pending_A of pending_B en wachten.

Indien de pending ouder wordt dan pair_window zonder partner:

- We vormen een "single_X_only"-stap (stability_hint = nudge).
- We voegen de bijbehorende single-sensor dir_hint in die stap.

---

## 4. Richtingsbepaling van een stap

### 4.1 Intersensorisch (klassiek)
Bij een pair:
- A_then_B ⇒ step_dir = "CW" (kalibreerbaar; kan gespiegeld zijn afhankelijk van fysieke montage)
- B_then_A ⇒ step_dir = "CCW"

We nemen ook delta_t_AB_us op en bepalen quality uit monotonicity/snr van beide flanken.
Als `quality >= 0.7 ⇒ stability_hint = "rot_candidate"`,
anders `stability_hint = "nudge"`.

### 4.2 Intrasensorisch (nieuw: S02.M082)
Bij een single-sensor stap:
- step_dir = "NONE" omdat we nog geen geometrische bevestiging hebben.
Maar we bewaren de dir_hint van die sensor in de stap.
- stability_hint = "nudge".

Zo draagt de stap wel richting-energie, zelfs als hij geen volwaardige rotatiekandidaat is.

---

## 5. Export van PureSteps
Elke keer dat een stap wordt gevormd (pair of single), exporteert Core0-Pure een PureStep:
We produceren een `Core0.PureStep.v1` object:

```json
{
  "t_us": <int>,
  "step_dir": "CW" | "CCW" | "NONE",
  "dir_hint": "up" | "down" | "none",
  "quality": <float 0..1>,
  "delta_t_AB_us": <int|null>,
  "phase_tag": "A_then_B" | "B_then_A" | "single_A_only" | "single_B_only",
  "stability_hint": "rot_candidate" | "nudge" | "ambiguous"
}
```
Deze PureSteps zijn de enige input voor Core-1 Awareness.

Core-1 Awareness:
- kan een reeks rot_candidate stappen met stabiele step_dir herkennen als rotating_[dir],
- kan een incidentele nudge met dir_hint herkennen als nudged_*,
- kan rpm schatten alleen op basis van rot_candidate stappen in dezelfde step_dir.

## 6. VirtualAngleSpace Integration

### 6.1 Rotorfysica (canon)
- De rotor heeft 24 magneten totaal.
- Dat zijn 12 noord/zuid-poolparen.
- Elke poolpaar-cyclus (ZUID → NEUTRAAL → NOORD → NEUTRAAL → ZUID) is wat de Hall-sensoren zien als één “elektrische cyclus”.

Gevolg:
- Terwijl de rotor 360° mechanisch draait, doorlopen de velden 12 volledige elektromagnetische cycli.
- Dus: 1 mechanische omwenteling ≈ 12 elektrische omwentelingen.

We segmenteren daarom de mechanische omwenteling in 12 vaste sectoren van ongeveer 30° mechanisch per sector.

Dit verklaart waarom ruwe sensorfasen (A,B) sneller “rondgaan” dan de fysieke as:
de virtuele veldhoek draait 12x sneller dan de mechanische hoek.

### 6.2 Virtuele hoekruimte (elliptical angle space)
We definiëren een virtuele hoek:
`θ_virtual = atan2(B′, A′)`
waar A′ en B′ genormaliseerde / gecorrigeerde waardes zijn van Hall A en Hall B.

In een ideale wereld met perfecte sensoren:
- A′ en B′ zouden ideale sinussen zijn, 90° in fase verschoven, gelijke amplitude, nul offset ⇒ perfecte cirkel in (A′,B′)-ruimte.
- θ_virtual zou lineair oplopen binnen elke elektrische cyclus.
- De rotor zou dus 12 × 0→360° doorlopen in θ_virtual per 360° mechanisch.

In de echte wereld (onze hardware):
- A en B hebben verschillende amplitudes, offsets en geen perfecte 90° fase.
- Daardoor vormt (A,B) geen cirkel maar een verschoven, afgeplatte ellips.
- θ_virtual bestaat nog steeds (atan2 werkt altijd), maar hij versnelt en vertraagt afhankelijk van waar we op de ellips zitten.
- Met andere woorden: θ_virtual is een continue “veld-fase”, niet een uniforme hoek in mechanische tijd.

Belangrijk punt:
θ_virtual is wél betekenisvol als “waar zitten we binnen dit magnetische segment?”, ook al is hij niet direct 0–360° mechanisch.

### 6.3 Hoe θ_virtual thuishoort in de lagen
We onderscheiden twee niveaus:

1. Core-0 PURE (v0.2 → v0.3 scope)
Verantwoordelijkheid van PURE t.o.v. θ_virtual:
- PURE = fase binnen een segment, AWARENESS = samenvoegen tot globale asbeweging.
- PURE mag θ_virtual lokaal evalueren om te weten “waar in de huidige poolovergang zitten we nu?”
  bv. “we zitten nog in de opgaande flank van sensor A”, “we naderen neutraal”, “we zijn net voorbij het veldmaximum”.
- PURE mag deze waarde annoteren bij een stap als extra context (interne telemetrie / debug).
- PURE hoeft θ_virtual niet te lineariseren over 0–360° mechanisch, en ook niet te integreren over meerdere sectoren.
  → Met andere woorden: PURE mag θ_virtual zien als lokale binnen-sector fase, niet als absolute rotorhoek.

2. Core-1 Awareness
Verantwoordelijkheid van Awareness t.o.v. θ_virtual:
- Awareness mag opeenvolgende θ_virtual-metingen (plus sector-index) over tijd vergelijken om:
-- richtingconsistentie te bevestigen (“θ_virtual loopt vooruit in dezelfde polariteit over opeenvolgende stappen”),
-- snelheid/rpm betrouwbaarder te schatten (differentiëren van fase i.p.v. alleen step timing),
-- mechanische voortgang te reconstrueren:
  `mech_angle ≈ sector_index * (360° / 12) + f(θ_virtual)`
  waarbij f(θ_virtual) de offset/positie binnen de actieve 30° sector is.
- Awareness mag dus zeggen:
-- “Ik ben 3.2 sectoren verder sinds 500 ms geleden”
-- en dat vertalen naar mechanische rotatie, globale richting, en intentie.

Kortom:
PURE = fase binnen een segment.
AWARENESS = samenvoegen van segmenten tot globale asbeweging.

### 6.4 Relatie tot M082 (dir_hint)
We hebben nu twee primaire manieren om richting te voelen:
1. SingleSensorDirectionalHint (M082):
- Kijk naar één sensor.
- Volg de lokale veldhelling (dv/dt, van SOUTH→NEUTRAL→NORTH of omgekeerd).
- Dit geeft een “dir_hint”: "up", "down", "none".
- Dit werkt zelfs bij extreem langzame of heel korte tikjes (nudge).
2. θ_virtual drift in tijd (M083):
- Kijk naar de evolutie van θ_virtual.
- Als θ_virtual consequent oploopt, weten we “ik ga in mijn positieve veldrichting verder.”
- Als θ_virtual consequent afloopt, weten we “ik ga de andere kant op.”
- Dit is een continue indicator, geen discrete stap.

Beide mechanismen geven richting-informatie vóór of zelfs zonder een perfecte A↔B pair-confirmatie.
Daarmee ontstaat een hiërarchie van richting:
- “ik voel een duw in dit veld” (M082 / dir_hint),
- “ik zie mijn fase zich voortbewegen” (M083 / θ_virtual drift),
- “ik zie ook dat A en B in vaste volgorde happen” (klassieke pair → CW/CCW),
- “ik blijf dat doen over meerdere sectoren” (Awareness → echte intentie).

### 6.5 Waar we dit voor gebruiken
1. Core-0 PURE v0.2 (en v0.3 vooruitkijkend) mag dir_hint (M082) en lokale θ_virtual-fase (M083) samenvoegen in de PureStep export.
- Bijvoorbeeld door extra velden als dir_hint en theta_local mee te geven.
2. Core-1 Awareness neemt die PureSteps over tijd en kan eindelijk met vertrouwen zeggen:
- “rotating_CW”
- “nudged_CCW”
- “stabilizing”
- inclusief rpm en confidence,
  en uiteindelijk ook een reconstructie van globale mechanische voortgang, niet alleen lokale duwtjes.

### 6.6 Canonische update
- De rotor in deze S02 build heeft 12 poolparen (24 magneten totaal).
- Eén mechanische omwenteling = 12 elektromagnetische cycli.
- θ_virtual = atan2(B′, A′) registreert positie binnen één elektromagnetische cyclus.
- Die binnen-cyclus-positie plus sector-index kan op Awareness-niveau een globale mechanische hoek schatten.

Dit betekent dat onze “virtuele hoek” nooit fout was.
Ze was alleen nooit bedoeld als directe absolute 0–360° as-encoder.
Ze is een lokale faseklok die, in combinatie met sectorregeling, toch een lichaamsbesef van plaats kan geven.

## 7. Forward spec: Core0-Pure v0.3 (theta_local_deg export)
### 7.1 Doel
Elk PureStep object moet niet alleen zeggen “ik bewoog CW of CCW”, maar ook “ik bevond me rond x % van mijn veldcyclus toen dat gebeurde”.
- Core0-Pure v0.3 breidt de bestaande PureStep-structuur uit met een veld theta_local_deg, dat de positie van de rotor binnen het huidige magnetische segment vastlegt.
- Waar M082 richting voelde (dir_hint) en M083 fase beschreef (θ_virtual), vormt v0.3 de praktische implementatie ervan.

### 7.2 Nieuwe variabele
| Veld              | Beschrijving                                                    | Bereik | Bron                                                                                     |
| ----------------- | --------------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------- |
| `theta_local_deg` | Lokale fasehoek binnen het actieve magnetische segment (0–360°) | float  | afgeleid uit `from_pool` / `to_pool` (v0.3-C optie) of later uit `atan2(B′, A′)` (v0.4+) |

### 7.3 Implementatie (Optie C — lokale poolfase)
Voor v0.3 wordt de hoek niet via een echte atan2(A,B) bepaald (geen gelijktijdige samples), maar via een ruwe fase-mapping per sensor:
- SOUTH →  0°
- NEUTRAL → 180°
- NORTH  → 360°
(Hoeken worden intern in graden weergegeven; 0 ≤ θ < 360.)

De gemiddelde overgang tussen from_pool en to_pool geeft de lokale fase van het flankmoment.
Voorbeeld: SOUTH→NEUTRAL ≈ 90°, NEUTRAL→NORTH ≈ 270°.

Deze benadering is stabiel, werkt voor single-sensor flanken, en vereist geen simultane A + B meting.
Ze mag in latere versies vervangen worden door een echte θ_virtual-berekening zodra gecalibreerde A′ en B′ beschikbaar zijn.

### 7.4 Export schema (PureStep v0.3)
```json
{
  "t_us": <int>,
  "step_dir": "CW" | "CCW" | "NONE",
  "dir_hint": "up" | "down" | "none",
  "theta_local_deg": <float>,
  "quality": <float>,
  "delta_t_AB_us": <int|null>,
  "phase_tag": "A_then_B" | "B_then_A" | "single_A_only" | "single_B_only",
  "stability_hint": "rot_candidate" | "nudge" | "ambiguous"
}
```
theta_local_deg wordt geëxporteerd voor zowel pair als single-sensor stappen.
Bij pair-stappen wordt de waarde genomen van de laatste (flank met hoogste t_us).

### 7.5 Verwachte gedragingen
- Awareness kan de monotone trend van theta_local_deg over tijd gebruiken om rotatie en stabiliteit te detecteren.
- Bij step_dir = "NONE" en dir_hint ≠ "none" kan Awareness nu een richting afleiden (nudged_CW/nudged_CCW).
- De fase-evolutie (θ toenemend of afnemend) wordt zo een extra vector voor consistency checks.

### 7.6 Toekomst (v0.4 vooruitblik)
Wanneer Core-0 Sensing synchroon A en B kan streamen, wordt theta_local_deg verfijnd tot:
`θ_virtual = atan2(B′, A′)`
waarbij A′ en B′ de offset- en amplitude-gecorrigeerde sensorwaarden zijn.
Dan kan theta_local_deg een echte lokale veldhoek worden en zal Awareness deze gebruiken voor rpm / consistency-analyse.

### 7.7 Status
| Attribuut           | Waarde                             |
| ------------------- | ---------------------------------- |
| Versie              | Core0-Pure v0.3 (draft)            |
| Manifest / XRAM     | blijven v0.2 (geldig)              |
| Implementatie       | Python Core0PureBuilder_v03 schets |
| Addenda referenties | S02.M082 · S02.M083                |

Deze forward spec maakt Core-0 PURE fasebewust zonder zijn nuchtere karakter te verliezen:
hij weet nu niet alleen dat hij bewoog, maar ook waar in zijn cyclus hij geraakt werd.

## 8. Samenvatting van verantwoordelijkheid
Core0-Pure v0.2 is niet meer “alleen maar pairing”.
Ze:

1. Schermt ruis af van de hogere lagen.
2. Creëert betekenisvolle mechanische stappen.
3. Geeft ook bij incomplete informatie toch een gevoelsrichting mee.
4. Preserveert tijdstempels en kwaliteitsmetadata op microseconde-schaal.
5. Vormt samen met Core-1 Awareness een volledige “field-to-motion” bewustzijnslus (M079 → M083).

Zo hoeft Core-1 Awareness niet meer te worstelen met onvolledige rauwe flanken,
maar kan ze praten in termen van “ik draai echt CW” of
“ik werd net een tikje CCW aangestoten”.

Belangrijk hier:
- S02.M082 is nu onderdeel van de officiële beslislogica.
- S02.M083 VirtualAngleSpace Integration vastgelegd
-- 12 poolparen / 24 magneten
-- rol van θ_virtual als binnen-segment fase,
-- en de verdeling PURE vs Awareness.
- `dir_hint` is niet alleen “nice to have”, maar bewust een veld dat doorstroomt tot Awareness.
