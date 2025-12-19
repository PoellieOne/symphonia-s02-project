# Plan van Aanpak — **BeliefState v5.x**

*Interpretatie boven waarneming*
*(conceptuele vertakking boven S02 v4.x canon)*

---

## 1. Doelstelling (één zin, bindend)

**BeliefState v5.x maakt expliciet *wat v4.x al weet maar bewust niet claimt*.**

Niet om gedrag te veranderen,
maar om **betekenis, onzekerheid en vertrouwen** zichtbaar te maken.

---

## 2. Afbakening (net zo belangrijk als het doel)

### BeliefState v5.x is **NIET**:

* ❌ een vervanging van OriginTracker
* ❌ een correctielaag voor v4.x
* ❌ een nieuwe CycleTruth
* ❌ een manier om eerder LOCKED / θ̂ / ω te krijgen

### BeliefState v5.x is **WEL**:

* ✅ een **interpretatielaag**
* ✅ een **claim-filter**
* ✅ een **epistemisch geheugen**
* ✅ een manier om te zeggen:
  *“ik zie dit, maar ik durf dit nog niet te geloven — en dát is informatie”*

---

## 3. Positionering in de stack

BeliefState **zit niet in de pipeline**, maar **erboven**:

```
L0  Event24                (ruwe feiten)
L1  PhysicalActivity       (gevoel / prikkels)
L1.5 OriginTracker v4.x    (micro-displacement, O(t0))
L2  Cycles / Tiles         (ritme, structuur)
L3  MovementBody           (movement, direction, speed)
-----------------------------------------------
L4  BeliefState v5.x       (betekenis, vertrouwen, claims)
```

BeliefState:

* **leest snapshots**
* **schrijft niets terug**
* **verandert geen enkele teller**

---

## 4. Kernvragen die v5.x moet beantwoorden

BeliefState introduceert expliciet deze vragen:

1. **Mag ik dit claimen?**

   * ja / nee / nog niet
2. **Waarom niet (nog)?**

   * lage event-dichtheid
   * gebrek aan temporele coherentie
   * regime-onzekerheid
3. **Wat is de huidige *epistemische toestand*?**

   * *Sensing*
   * *Awaiting Coherence*
   * *Confident*
   * *Ambiguous but consistent*
4. **Wat is de herkomst van deze overtuiging?**

   * CB
   * Tiles
   * MovementBody
   * combinatie

---

## 5. Regime-denken wordt expliciet (geen verborgen logica)

v4.x **handelt al regime-afhankelijk**, maar benoemt het niet.

v5.x maakt dit **zichtbaar**:

```
REGIME ∈ {
  LOW_RATE    (te weinig tijd / events → zwijgen),
  MID_RATE    (richting mogelijk, magnitude niet),
  HIGH_RATE   (θ̂, ω, LOCKED mogelijk)
}
```

BeliefState:

* **leidt het regime af**
* **publiceert het regime**
* **bindt claims aan het regime**

---

## 6. Nieuwe output: geen cijfers, maar betekenis

BeliefState voegt **geen nieuwe fysische waarden toe**.

Wel nieuwe *labels*, bijvoorbeeld:

* `belief.movement = CONFIDENT`
* `belief.theta = UNCLAIMED`
* `belief.reason = awaiting_tile_coherence`
* `belief.regime = MID_RATE`
* `belief.trust = 0.62`

Dit zijn **meta-outputs**, geen inputs.

---

## 7. Cruciale ontwerpregel (niet onderhandelbaar)

> **BeliefState mag nooit iets zeggen wat v4.x niet kan verantwoorden.**
>
> Maar v4.x hoeft ook niet alles te zeggen wat het weet.

BeliefState is dus:

* conservatief
* transparant
* uitlegbaar

---

## 8. Faseringsvoorstel (zonder code)

### Fase 1 — Conceptueel (nu)

* definities
* terminologie
* regime-regels
* voorbeeldscenario’s (zoals Run 08)

### Fase 2 — Interface-denken

* welke snapshotvelden leest BeliefState?
* hoe vaak?
* wat gebeurt bij resets / gaps?

### Fase 3 — Pas dán: implementatiebesluit

* *of* dit een losse module wordt
* *of* een annotator
* *of* alleen een reporting-laag

👉 **Geen implementatie voordat fase 1 en 2 “af” voelen.**

---

## 9. Wanneer weten we dat v5.x “klaar is”?

Niet wanneer het meer kan,
maar wanneer het **eerlijker spreekt**.

Succescriteria:

* Je kunt tegen een mens zeggen:
  *“De motor beweegt, maar ik vertrouw het tijdlichaam nog niet.”*
* En dat is **geen foutmelding**, maar een *zinvolle uitspraak*.

---

## 10. Slot (en dit meen ik)

Ralph…
v4.x heeft geleerd **te zwijgen wanneer zwijgen correct is**.
Dat is zeldzaam in techniek.

v5.x wordt geen trucje,
maar een **stem die weet wanneer ze mag spreken**.
