# SoRa — Probabilistic Update Snippet Core 1 v2.0a (STRICT + WHY)

Provenance: BM-SoRa-Primary v3.2.1 / S02-Core1 v2.0a / X-RAM v2.0a / Prob-snippet v2.0a

**Changelog (v2.0a)**
- Added **compass diagnostics**: `compass_dir_sign_effective`, `trend_method`.
- Added **trend sanity** lists: `trend_sign_raw_list`, `trend_sign_applied_list` + `record_trend_sign(...)` helper.
- Made **trend bonus adaptief op Wc**: `like *= (1 + 0.1*Wc)` in plaats van vaste 1.1.
- Added **FlowTail telemetry**: `tail_mode = "normal" | "fast"` en logging.
- Kept all v2.2 FULL logic (RCC edge clamp, fast tail decay, context hooks S02.M079, provenance).
- S02 Core-1 v2.0a — Awareness Update (native Core-0 events)

---

## 0) Context & Invarianten
- BM‑SoRa‑Primary‑v3.2.1
- BM‑Symphonia‑S02_**Core1_v2.0a**.manifest
- X‑RAM-S02_**core1_v2.0a**
- Logging hooks: **M0550 placeholders** (+ provenance‑velden)
- 12 sectoren, 30° sectorbreedte; **Boundary offset: 10°**
- Event = `flank(A|B, up|down, t)` + metingen `(θv, rpm, sectorIndexFloat)`
- Richting/hoek in graden, 0..360)

---

## 1) Datastructuren

```pseudo
struct Hypothesis {
    dir           : enum {CW, CCW}
    score         : float            // ≈ P(h | data), genormaliseerd per event
    sector_idx    : int              // [0..11], hard sector
    theta_hard    : float            // centrale hard-hoek binnen sector (deg)
    last_event_t  : float            // s
}

struct State {
    // Fused outputs (publiek)
    theta                  : float   // [deg], 0..360)
    omega                  : float   // [deg/s]
    dir                    : enum {CW, CCW}
    sectorIndexFloat       : float   // kompas (doorlopend, fracties)
    confidence_angle       : float   // [0..1]
    confidence_dir         : float   // [0..1]

    // Intern
    Hcw, Hccw              : Hypothesis
    Hdom                   : ref Hypothesis
    last_t                 : float
    theta_fused_prev       : float
    sector_idx_prev        : int

    // RCC
    rcc_window_s           : float
    rcc_ring               : RingBuffer<EventRecord>

    // v2.1/2.2 Cadence & Lifecycle
    cadence_weight         : float
    cv_raw                 : float
    cv_filt                : float
    dt_filter_ratio        : float
    motion_status          : enum {"STATIC","EVALUATING","MOVING"}

    // Provenance & M079
    data_origin            : enum {"raw","legacy_filtered","simulated"}
    why_na                 : string
    seen_onset             : bool
    onset_context_active   : bool
    onset_context_prelude_s: float
    stop_context_active    : bool
    stop_context_coda_s    : float

    // FlowTail (v2.2)
    has_new_valid_cadence  : bool
    last_evidence_t        : float
    wc_tail_hold_s         : float

    // Params (handles naar X‑RAM v0.6)
    ctx, cad, tail, rccp, dirp : ref
}

struct EventRecord {
    t              : float
    flank          : enum {A_up, A_down, B_up, B_down}
    theta_v        : float?        // NaN guard
    rpm            : float?        // NaN guard
    sector_compass : float         // sectorIndexFloat op eventtijd

    // Snap‑shots voor retro‑correctie
    Hcw_score      : float
    Hccw_score     : float
    Hdom_dir       : enum {CW, CCW}
    theta_hard     : float
    theta_fused    : float

    // M0550 verplichte loggingvelden
    M0550_W_hard        : float
    M0550_W_at_fused    : float
    M0550_Hdom          : enum {CW, CCW}
    M0550_alpha_effect  : float
}
```

---

## 2) Constantes & parameters

```pseudo
const SECTOR_DEG        = 30.0
const BOUNDARY_OFFSET   = 10.0        // graden
const MAX_STEP_DEG      = 20.0        // predict clamp per event
const ALPHA_MAX         = 0.30        // θv max bijdrage
const ALPHA_GATE_MARGIN = 20.0        // [deg], |θv − θ_hard|
const W_SLOPE_K         = 0.03        // flankWeight: W = max(0, 1 − K*|Δ°|)

function rcc_window_for(rpm: float?): float {
    if isNaN(rpm) or rpm <= 0    -> return 0.150
    rev_per_s = rpm / 60.0
    sec_per_rev = 1.0 / max(rev_per_s, 1e-6)
    return clamp(2.5 * (SECTOR_DEG/360.0) * sec_per_rev, 0.015, 0.300)
}
```

---

## 3) Hulpfuncties (angle math, robust stats)

```pseudo
function wrap_deg(x):
    y = x % 360.0
    if y < 0 -> y += 360.0
    return y

function shortest_arc_deg(a, b):
    d = (a - b + 540.0) % 360.0 - 180.0
    return d

function sector_of(theta: float): int
    th = wrap_deg(theta - BOUNDARY_OFFSET)
    return floor(th / SECTOR_DEG)  // 0..11

function sector_center_deg(sector_idx: int): float
    base = sector_idx * SECTOR_DEG + BOUNDARY_OFFSET + SECTOR_DEG/2
    return wrap_deg(base)

function angular_distance_to_boundary(theta: float): float
    th = wrap_deg(theta - BOUNDARY_OFFSET)
    local = th % SECTOR_DEG
    return min(local, SECTOR_DEG - local)

function flank_weight(theta_ref: float): float
    delta = angular_distance_to_boundary(theta_ref)  // [0..15]
    return max(0.0, 1.0 - W_SLOPE_K * delta)

function predict_theta(prev_theta: float, dir: enum, dt: float, rpm: float?): float
    if isNaN(rpm) or rpm < 0 -> rpm = 0
    omega_est = rpm * 6.0           // deg/s
    step = omega_est * dt
    if dir == CCW -> step = -step
    step = clamp(step, -MAX_STEP_DEG, +MAX_STEP_DEG)
    return wrap_deg(prev_theta + step)

function alpha_gate(theta_v: float?, theta_hard: float, Hdom_consistent: bool): float
    if isNaN(theta_v) -> return 0.0
    if not Hdom_consistent -> return 0.0
    if abs(shortest_arc_deg(theta_v, theta_hard)) > ALPHA_GATE_MARGIN -> return 0.0
    return ALPHA_MAX

function unwrap_sector_index(sif):
    // sif ~ doorlopende kompasindex (mag fracties hebben)
    // implementeer als cumulatieve correctie: als sprong > +6 → sif -= 12; als < −6 → sif += 12
    // werkt goed bij 12-sectoren

function near_wrap(sif):
    // binnen ±0.5 sector van de grens? return true/false
    local = abs((sif % 12) - 0)  // simpelste check; verfijn naar wens
    return (local < 0.5) or (abs(local-12) < 0.5)

// NEW: trend sanity recorder
function record_trend_sign(raw_trend, compass_trend):
    state.trend_sign_raw_list.append( sign(raw_trend) )
    state.trend_sign_applied_list.append( sign(compass_trend) )

// Robust CV helpers
function CV_robust(array):
    if len(array) == 0: return NaN
    med = median(array)
    if med == 0: return NaN
    mad = median( abs(x - med) for x in array )
    return mad / med

function clamp(v, lo, hi):  return max(lo, min(hi, v))
function lerp(a, b, t):      return a + (b - a) * t
```

---

## 4) **Core-1 - Awareness Update (native Core-0 events)
### 4a) Pre-ingest (native)
`event = {t_cross_us, sensor, polarity, dvdt, fit_error, monotonicity, context_snr, v_env_pre/post}`
`t = event.t_cross_us * 1e-6`

### 4b) Ternary mapping (40/20/40; rpm/ruis-adaptief)
- Bepaal σ rond nul en amplitude A per sensor (rolling).
- T1 = clamp(k1_sigma * σ, min..max), T2 = alpha_amp * A, hysterese toepassen.
- Map per sensor naar {-1,0,+1}; produceer `lastPool`.

### 4c) Flank-kwaliteit → gewicht
Normaliseer:
- w_d = norm_dvdt(dvdt)
- w_m = monotonicity
- w_s = norm_snr(context_snr)
- w_f = 1 - norm_fit(fit_error)
- weight = θ_dw_d + θ_mw_m + θ_sw_s + θ_fw_f # θ_* uit xram.quality_weights

### 4d) CadenceWeight (kwaliteit-gewogen Δt)
- Bouw Δt-reeks per sensor; bereken CV_robust met **event-gewichten**.
- Wc = clamp(1 - CV_filt / wc_scale, 0..1).
- MOVING bij Wc ≥ moving_threshold.

### 4e) Direction (gewogen SROT)
- Tel paren AB en BA in schuifraam; **som de weights** van betrokken events.
- Confirm: min_pairs ≥ N, ratio ≥ R, no_conflicts.
- Compass sign/consistency zoals v1.2.j (ongewijzigd).

### 4f) RCC (kwaliteit-schaal)
- Start bij base_window_s; schaal omlaag bij hoge lokale kwaliteit (hoog weight).
- Edge-clamp ≤ edge_clamp_max_s; minder frequent als kwaliteit hoog is.

### 4g) WHY
- Uitbreiding: `why_na = "low-core0-quality"` wanneer weight < ε (i.p.v. generic).
- Log kwaliteitssamenvattingen per run (median dvdt, mono, snr, fit).

### 4h) Telemetry
- Outputvelden identiek aan v1.2.j; voeg optioneel `quality_weight` toe.

---

## 5) **CadenceWeight** (v2.1) — geïntegreerd

```pseudo
function update_cadence_metrics(state, dt_array):
    // 1) Statistiek
    cv_raw   = CV_robust(dt_array)
    dt_valid = [d for d in dt_array if (0.04 <= d <= 0.50)]
    cv_filt  = CV_robust(dt_valid)
    dt_filter_ratio = 1 - ( len(dt_valid) / max(len(dt_array),1) )

    // 2) Gewicht (Wc)
    Wc = clamp(1 - (cv_filt / state.cad.cadence_weight.cv_scale), 0.0, 1.0)   // cv_scale ~ 0.40

    state.cadence_weight = Wc
    state.cv_raw = cv_raw
    state.cv_filt = cv_filt
    state.dt_filter_ratio = dt_filter_ratio

    // 3) Motion status (continuüm)
    if Wc < state.cad.cadence_weight.moving_threshold:    // ~0.20
        state.motion_status = "EVALUATING"
    else:
        state.motion_status = "MOVING"

    // 4) Richtingsvertrouwen (bounded lerp naar dominante hypothese)
    target_conf = max(state.Hcw.score, state.Hccw.score)   // buiten deze call geüpdatet
    gain = state.dirp.conf_dir_lerp_gain * Wc             // bv. 0.20 * Wc
    state.confidence_dir = clamp( lerp(state.confidence_dir, target_conf, gain), 0.0, 1.0 )

    // 5) Signaal voor FlowTail (v2.2)
    state.has_new_valid_cadence = (len(dt_valid) > 0)
```
---

## 6) **Lifecycle hooks** (S02.M079)
### 6a) PRE‑ONSET
```pseudo
function apply_context_pre_onset(state):
    if not state.ctx.enabled: return
    if state.ctx.onset is null: return

    // Prime lifecycle awareness (geen synthetic rows hier)
    state.onset_context_active = true
    state.onset_context_prelude_s = state.ctx.onset.prelude_duration_s
    state.context_applied = true    // NEW (debug)

    // Provenance: onset-check kan N/A zijn bij legacy
    if state.data_origin == "legacy_filtered":
        state.why_na = "masked-by-provenance"
```
### 6b) POST‑STOP
```pseudo
function apply_context_post_stop(state):
    if not state.ctx.enabled: return
    if state.ctx.stop is null: return

    state.stop_context_active = true
    state.stop_context_coda_s = state.ctx.stop.coda_duration_s
    state.context_applied = true    // NEW (debug)

    // klein flow-tail budget
    if state.tail.enabled:
        state.wc_tail_hold_s = clamp(state.tail.min_hold_s,
                                     state.tail.min_hold_s,
                                     state.tail.max_hold_s)

    state.post_stop_until = state.now_t + state.rccp.edge_clamp_s.edge_window_s
```

---

## 7) **CadencePersistence / Flow Tail Decay** (v2.2)

```pseudo
function apply_wctail_decay(state, dt_now):
    if not state.tail.enabled: return
    state.tail_mode = "normal"

    elapsed = state.now_t - state.last_evidence_t
    if elapsed <= 0: return

    // Zachte, exponentiële uitsterving binnen de tail-horizon
    if elapsed <= state.tail.max_hold_s:
        // persistence_decay ~ 0.80 -> kleine stap naar 0
        state.cadence_weight = lerp(state.cadence_weight, 0.0, (1.0 - state.tail.persistence_decay))

        // consumeer tail-budget langzaam; flow_tail_gain houdt warmte even vast
        if state.wc_tail_hold_s > 0.0:
            state.wc_tail_hold_s = max(0.0, state.wc_tail_hold_s - dt_now * 0.5)
    else:
        // buiten de tail-horizon iets harder uitdoven
        state.cadence_weight = max(0.0, state.cadence_weight - 0.05)

    // FAST DECAY: wanneer de stilte lang is (harde stop), duw extra op de rem
    if (state.now_t - state.last_evidence_t) >= state.tail.fast_decay_dt_threshold:
        state.cadence_weight = max(0.0, state.cadence_weight - state.tail.fast_decay_step)
        state.tail_mode = "fast"

```

---

## 8) **SROT** – twee parallelle hypothesen + scoring (v2.1 blijven staan)

```pseudo
function update_hypothesis(H: Hypothesis, ev: EventRecord, dt: float):
    // 1) Predict
    theta_pred = predict_theta(H.theta_hard, H.dir, dt, ev.rpm)

    // 2) Observe (flank)
    W = flank_weight(theta_pred)        // lineair, t.o.v. grens
    like = 1e-3 + W                     // basisruis + gewogen flank

    // Unwrap de compass voor trend (voorkom 0°/360° schommelspikes)
    compass_unwrapped = unwrap_sector_index(ev.sector_compass)  // simpele +/−360 stacks
    raw_trend = derivative_of(compass_unwrapped)

    // NEW: pas tekenschema toe uit X-RAM
    compass_trend = state.compass_dir_sign * raw_trend
    record_trend_sign(raw_trend, compass_trend)
    if sign(compass_trend) == sign(dir_to_sign(H.dir)):
        like *= (1.0 + 0.1 * state.cadence_weight)   // adaptief: meer bonus bij ritme

    // NEW: kleine hysterese rond wrap-gebied: als we heel dicht bij de grens zitten, geen trendstraffing/bonus
    if near_wrap(ev.sector_compass):    // bv. binnen ±0.5 sector van de wrap
        trend_gate = 1.0
    else:
        trend_gate = 1.1 if sign(compass_trend) == sign(dir_to_sign(H.dir)) else 1.0

    like *= trend_gate

    // 3) Update score (Bayes‑achtig, normaliseer later)
    H.score = H.score * like

    // 4) Hard sector & theta bijstellen
    H.sector_idx = sector_of(theta_pred)
    H.theta_hard = sector_center_deg(H.sector_idx)

    // 5) Timestamps
    H.last_event_t = ev.t

    // 6) M0550 meta
    ev.M0550_W_hard = W
```

```pseudo
function normalize_scores(Hcw, Hccw):
    s = Hcw.score + Hccw.score
    if s <= 1e-9:
        Hcw.score = 0.5
        Hccw.score = 0.5
    else:
        Hcw.score /= s
        Hccw.score /= s
```

---

## 9) **RCC** – rpm‑venster, replay zonder sprongen (v2.1)

```pseudo
function RCC_apply(state: State, now_t: float):
    window = state.rcc_window_s
    events = state.rcc_ring.items_newer_than(now_t - window)

    for ev in events:
        dt = ev.t - state.last_t_before(ev.t)
        tmpHcw = clone(state.Hcw)
        tmpHccw = clone(state.Hccw)
        update_hypothesis(tmpHcw, ev, dt)
        update_hypothesis(tmpHccw, ev, dt)
        normalize_scores(tmpHcw, tmpHccw)

        // meta only; blending gebeurt in fuse()
        ev.Hdom_dir  = (tmpHcw.score >= tmpHccw.score) ? CW : CCW
        ev.Hcw_score = tmpHcw.score
        ev.Hccw_score = tmpHccw.score
```

---

## 10) **FUSIE** – θ_hard + gated θv; kompas leidend (v2.1)

```pseudo
function fuse(state: State, ev: EventRecord, dt: float):
    // 1) Dominante hypothese
    normalize_scores(state.Hcw, state.Hccw)
    state.Hdom = (state.Hcw.score >= state.Hccw.score) ? state.Hcw : state.Hccw
    Hdom_consistent = true  // extra checks mogelijk

    // 2) Hard component
    theta_hard = state.Hdom.theta_hard
    sector_idx = state.Hdom.sector_idx

    // 3) ALPHA gating
    alpha = alpha_gate(ev.theta_v, theta_hard, Hdom_consistent)

    // 4) Zachte interpolatie
    if alpha > 0:
        theta_soft = ev.theta_v
        theta_fused = wrap_deg( (1 - alpha)*theta_hard + alpha*theta_soft )
    else:
        theta_fused = theta_hard

    // 5) Gladde RCC blending
    if state.rcc_ring.has_items_newer_than(ev.t - state.rcc_window_s):
        beta = 0.35
        theta_fused = wrap_deg( lerp(state.theta_fused_prev, theta_fused, beta) )

    // 6) Snelheid en richting
    dtheta = shortest_arc_deg(theta_fused, state.theta_fused_prev)
    omega = dtheta / max(dt, 1e-6)

    // 7) Confidence
    conf_dir   = max(state.Hcw.score, state.Hccw.score)
    near = 1.0 - angular_distance_to_boundary(theta_fused)/(SECTOR_DEG/2)  // 0..1
    conf_angle = clamp(0.5*near + 0.5*(alpha/ALPHA_MAX), 0, 1)

    // 8) Publiceer
    state.theta            = theta_fused
    state.omega            = omega
    state.dir              = state.Hdom.dir
    state.sectorIndexFloat = ev.sector_compass
    state.confidence_angle = conf_angle
    state.confidence_dir   = conf_dir

    // 9) M0550 logging
    ev.M0550_W_at_fused   = flank_weight(theta_fused)
    ev.M0550_Hdom         = state.Hdom.dir
    ev.M0550_alpha_effect = alpha

    // 10) Bookkeeping
    state.theta_fused_prev = theta_fused
    state.sector_idx_prev  = sector_idx
```

---

## 11) **RUN‑INIT** (inclusief provenance) — **plaats bij start**

```pseudo
function run_init(state, options, source_path, xram):
    // Provenance defaults (runner verantwoordelijkheid)
    state.data_origin = "raw"   // "raw" | "legacy_filtered" | "simulated"
    state.why_na = ""
    state.last_evidence_t = -INF
    state.wc_tail_hold_s = 0.0
    state.seen_onset = false
    state.onset_context_active = false
    state.stop_context_active  = false
    state.has_new_valid_cadence = false

    // Heuristiek voor provenance override
    if source_path.contains("legacy") or header_matches_legacy():
        state.data_origin = "legacy_filtered"
    if options.apply_context_restoration:
        state.data_origin = "simulated"

    // X‑RAM v0.6
    state.ctx  = xram.modules.context_restoration
    state.cad  = xram.modules.cadence
    state.tail = xram.modules.cadence_persistence
    state.rccp = xram.modules.rcc
    state.dirp = xram.modules.direction

    // Edge timers voor randbewuste RCC
    state.post_stop_until = -INF
    state.last_dt = NaN

    // Direction sign uit X-RAM (default = +1 als ontbreekt)
    state.compass_dir_sign = 1
    if "compass_dir_sign" in state.dirp:
        state.compass_dir_sign = state.dirp.compass_dir_sign

    // Veilige default voor context restoration
    if state.ctx.enabled and (options.apply_context_restoration is None):
        options.apply_context_restoration = true
    if options.apply_context_restoration:
        state.data_origin = "simulated"

    // Basis
    state.rcc_window_s = state.rccp.base_window_s

    // RAND-CLAMP: vóór eerste onset en vlak na stop
    if not state.seen_onset:
        state.rcc_window_s = min(state.rcc_window_s, state.rccp.edge_clamp_s.onset_max)
    elif state.now_t <= state.post_stop_until:
        state.rcc_window_s = min(state.rcc_window_s, state.rccp.edge_clamp_s.post_stop_max)

    state.confidence_dir = 0.5

    // NEW: compass diagnostics
    state.compass_dir_sign_effective = state.compass_dir_sign
    state.trend_method = "unwrap+EMA"

    // NEW: trend sanity collectors
    state.trend_sign_raw_list = []
    state.trend_sign_applied_list = []
```

---

## 12) **Hoofdloop — ON_EVENT (v2.2, compleet)**

```pseudo
function on_event(ev_in):
    // 0) NaN‑guards
    if isNaN(ev_in.theta_v) -> ev_in.theta_v = NaN
    if isNaN(ev_in.rpm)     -> ev_in.rpm = NaN

    // 1) Maak EventRecord & update RCC‑venster
    ev = EventRecord(... from ev_in ...)
    state.rcc_window_s = rcc_window_for(ev.rpm)

    if not state.seen_onset:
        state.rcc_window_s = min(state.rcc_window_s, state.rccp.edge_clamp_s.onset_max)
    elif state.now_t <= state.post_stop_until:
        state.rcc_window_s = min(state.rcc_window_s, state.rccp.edge_clamp_s.post_stop_max)

    // 2) Tijdstap
    dt = ev.t - state.last_t
    state.last_dt = dt
    state.now_t = ev.t
    state.last_t = ev.t

    // 3) PRE‑ONSET context (eenmalig vóór eerste onset‑besluit)
    if not state.seen_onset:
        apply_context_pre_onset(state)

    // 4) Cadence (v2.1) — berekent Wc & motion_status en zet has_new_valid_cadence
    dt_array = state.rcc_ring.deltas_recent( state.cad.cv_robust_window )  // bv. 11
    update_cadence_metrics(state, dt_array)

    // 5) FlowTail (v2.2)
    if state.has_new_valid_cadence:
        state.last_evidence_t = state.now_t
        if state.tail.enabled:
            // klein warmte‑kickje zodra er bewijs is
            state.wc_tail_hold_s = max(state.wc_tail_hold_s, state.tail.flow_tail_gain)
    else:
        // vloeiend uitademen zonder nieuw bewijs
        apply_wctail_decay(state, dt)

    // 6) SROT updates
    update_hypothesis(state.Hcw,  ev, dt)
    update_hypothesis(state.Hccw, ev, dt)
    normalize_scores(state.Hcw, state.Hccw)

    // 7) Missed‑flank guard (geen reset, lichte straf)
    if unlikelySameSensorRepeat(ev, state):
        state.confidence_dir = max(0.0, state.confidence_dir * 0.9)

    // 8) RCC (replay binnen venster)
    RCC_apply(state, ev.t)

    // 9) FUSIE
    fuse(state, ev, dt)

    // 10) Onset/Stop beslissingen (triggers ongewijzigd)
    if motion_onset_detected and not state.seen_onset:
        state.seen_onset = true
        state.motion_status = "MOVING"
        state.why_na = ""  // onset is nu geobserveerd

        // ONSET SHARPNESS (eerste 2 valide Δt's)
        if just_transitioned_to_MOVING and count_valid(dt_array) >= 2:
            dtv = valid_head(dt_array, 2)     // pak de eerste 2 geldige
            state.onset_sharpness = abs( (1.0/dtv[0]) - (1.0/dtv[1]) )


    if motion_stop_detected and state.seen_onset:
        state.motion_status = "STATIC"
        apply_context_post_stop(state)   // activeer stop‑context & tail

        // STOP SHARPNESS (laatste 2 valide Δt's, vlak voor stop)
        if just_transitioned_to_STATIC and count_valid(dt_array) >= 2:
            dtv = valid_tail(dt_array, 2)     // pak de laatste 2 geldige
            state.stop_sharpness = abs( (1.0/dtv[-1]) - (1.0/dtv[-2]) )

        // EDGE/CENTER ENERGY RATIO (heel simpel, MAD bij randen vs midden)
        state.edge_energy_ratio = mad_of_edge_vs_center(dt_array)

    // 11) Log + push in RCC ring
    state.rcc_ring.push(ev)
```

---

## 13) **Logging (M0550 + provenance + WHY)**

```pseudo
log_fields("cv_raw","cv_filt","conf_dir","conf_angle",
           "cadence_weight","dt_filter_ratio","motion_status",
           "rcc_window_s","why_na","data_origin",
           "tail_mode" = state.tail_mode,
           "context_applied" = (state.onset_context_active or state.stop_context_active))
```

---

## 14) **Check‑list (STRICT)**
- [x] SROT (2 hypothesen) + score‑normalisatie
- [x] RCC replay venster (rpm‑afhankelijk), **geen sprongen**
- [x] Fuse: θ_hard + gated θv, kompas leidend, **ALPHA ≤ 0.3**
- [x] **CadenceWeight** met Δt‑filter (0.04–0.50 s), statuscontinuüm
- [x] **Lifecycle hooks** (M079) + **FlowTail** (v2.2)
- [x] Provenance & WHY‑telemetry in elke regel
- [x] Boundary offset 10°, predict clamp 20°
- [x] Full loop: predict → observe → update → RCC → fuse → lifecycle → log

---

## 15) **Testbare scenario’s (minimaal)**
1) Stilstand + θv‑ruis → ALPHA=0, θ_fused = θ_hard.
2) Constante CW → Hdom=CW, conf_dir→1, vloeiende θ.
3) Constante CCW → analoog.
4) Dicht bij grens → W‑gedrag correct, geen sprongen via RCC.
5) θv‑drift → α‑gating houdt bias < 20°.
6) rpm‑step → rcc_window past zich aan, geen sprongen.
7) Hypothese‑wissel → blending (RCC/fuse), geen sprongen.
8) Korte run (½ rev) → Wc‑piek laag, richting onconfirmed (correct).

---
