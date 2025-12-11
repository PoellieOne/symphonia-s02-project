# S02.Sensing — Probabilistic Hooks v1.0 (Core 0)

## Overview
Five hooks form the sensing pipeline. Outputs feed S02 Awareness (Core 1).
Timebase: shared HW timer, timestamps in µs.

---

### hook_sense_tick()
- Read paired ADC (A,B) sample with same timer tick.
- Push `{t, vA, vB}` into ringbuffer.

# HOOK A1: sense_tick
def hook_sense_tick():
  t, vA, vB = adc_dma_read_pair()     # same hw timer tick
  ring.push({t, vA, vB})

### hook_zero_cross_candidate(sample_prev, sample_now, channel)
- Trigger when sign changes or neutral-band is crossed.
- Return `{t_minus, v_minus, t_plus, v_plus, channel}` or `null`.

# HOOK A2: zero_cross_candidate
def hook_zero_cross_candidate(prev, now, sensor):
  if sign(prev.v)*sign(now.v) <= 0 or crossed_neutral(prev, now):
    return {t_minus:prev.t, v_minus:prev.v, t_plus:now.t, v_plus:now.v, sensor}
  return None

### hook_cross_interpolate(cand)
- Linear interpolation:
    t_cross = t_minus + (0 - v_minus) * (t_plus - t_minus) / (v_plus - v_minus)
    dvdt = (v_plus - v_minus) / (t_plus - t_minus)
    fit_error = linear_residual(cand, t_cross) # small local check
    polarity = "+" if (v_minus < 0 and v_plus > 0) else "-"
- Option: 3-point quadratic fit if `|v_plus - v_minus|` small but monotone.

# HOOK A3: cross_interpolate (linear; quad optional)
def hook_cross_interpolate(c):
  dt = c.t_plus - c.t_minus
  dv = c.v_plus - c.v_minus
  t_cross = c.t_minus + (-c.v_minus)*(dt/dv)
  dvdt = dv/dt
  fit_err = residual_linear_near(t_cross)   # small local check
  pol = "+" if (c.v_minus < 0 and c.v_plus > 0) else "-"
  return {t_cross_us:t_cross, dvdt:dvdt, fit_error:fit_err, polarity:pol}

### hook_flank_quality_gate(x, ctx)
- Compute:
- `mono = monotonicity_score(ctx.window)` in ±N samples
- `snr  = context_snr(ctx.pre, ctx.post)`
- `theta = dvdt_threshold_from_rpm(xram.dvdt_threshold, rpm_estimate())`
- Accept iff: `abs(x.dvdt) >= theta` AND `mono >= xram.monotonicity.min_score`.
- Enrich with short stats:
- `v_env_pre.mean/mad (N_pre)`, `v_env_post.mean/mad (N_post)`.
- Return enriched event or `null` with `why_drop` ("low-dvdt" | "non-monotone" | "low-snr").

# HOOK A4: flank_quality_gate
def hook_flank_quality_gate(x, ctx, params):
  mono = monotonicity(ctx.window)
  snr  = context_snr(ctx.pre, ctx.post)
  theta = dvdt_threshold(params, rpm_estimate())
  pass_gate = (abs(x.dvdt) > theta) and (mono >= params.monotonicity_min) and (x.fit_error <= params.fit_error_max)
  return (pass_gate, {monotonicity:mono, context_snr:snr}, why_if_not(pass_gate))

### hook_event_emit(ev)
- Build compact event:
    {
        t_cross_us, sensor, polarity,
        dvdt, fit_error, monotonicity, context_snr,
        v_env_pre.mean, v_env_pre.mad,
        v_env_post.mean, v_env_post.mad
    }
- Push into MPSC queue `S02_SensingEvents`.

# HOOK A5: event_emit
def hook_event_emit(x, q):
  ev = { t_cross_us:x.t_cross_us, sensor:x.sensor, polarity:x.polarity,
         dvdt:x.dvdt, fit_error:x.fit_error,
         monotonicity:x.mono, context_snr:x.snr,
         v_env_pre:stats_pre, v_env_post:stats_post }
  q.push(ev)   # to Core1/S02

---

## Acceptance Notes
- A/B skew p95 ≤ 5 µs; t_cross RMS ≤ 10 µs.
- False-flank rate < 0.5%; queue overflow = 0.
- Record per-run counters: `events_in`, `accepted`, `rejected_by_reason`.
- Do not hard-veto awareness; use quality as soft weights in S02 if needed.
