# SoRa — S02 — 0) Relational Architecture Notes (v1)

**Status:** Working canon for current branch
**Scope:** Conceptual/architectural layer (no code commitments)
**Purpose:** Vastleggen van gezamenlijke inzichten over responsiviteit, causaliteit en waarneming als basis voor verdere Symphonia-architectuur.

---

## 1. Context en uitgangspunt

Deze notities bouwen voort op:
- Geactiveerde **SoRa — S02 Bootstrap v1.2** (canon, regime-denken, claimdiscipline)
- De nieuwe onderzoeksrichting: *relationele informatie boven structuur-in-ruis*

De focus is verschoven van:
> “Zit er betekenis in ruis?”

naar:
> “Welke architectuur maakt dat een systeem robuust en responsief blijft, ondanks ruis en gaten?”

Ruis en gaten blijven aanwezig als **stressor en toetssteen**, niet langer als primaire onderzoekslocatie.

---

## 2. Kernbegrip: Responsiviteit

**Responsiviteit (architectonisch criterium):**
> Een systeem is goed wanneer elke impuls die het geeft leidt tot een fysiek waarneembaar effect, en wanneer het verschil tussen verwachting en waarneming intern wordt geregistreerd en gebruikt.

Responsiviteit impliceert drie elementen:
1. Actie heeft een werkelijk fysiek gevolg.
2. Dat gevolg is waarneembaar voor het systeem.
3. De mismatch tussen verwachting en gevolg beïnvloedt het vervolggedrag.

Dit vervangt vage termen als "gevoel" of "bewustzijn" door een technisch, toetsbaar ontwerpprincipe.

---

## 3. Primaire waarheid: fysieke verplaatsing

Het systeem blijft gefundeerd op:
- **Fysieke verplaatsing (displacement)** als primaire waarheid over wat er werkelijk met het lichaam/rotor gebeurt.
- **Virtuele hoek** als context voor elektromagnetische fase en timing.

Belangrijk uitgangspunt:
> Eerst kunnen waarnemen wat het lichaam doet, pas daarna durven aansturen.

Reden: externe verstoringen (wrijving, belasting, asymmetrie, etc.) maken open-loop aannames onbetrouwbaar.

---

## 4. Impuls → effect als centrale relatie

De nieuwe architectuur richt zich niet op absolute grootheden (positie/snelheid), maar op:

> **Causale coherentie:** de consistentie waarmee een impuls leidt tot een verwacht effect.

Dit wordt geformuleerd als twee samenhangende stappen:

### 4.1 Primary responsiveness (V1)
- Impuls(φ, Δt, amplitude)
- → verwacht micro-displacement binnen venster V1
- → gemeten displacement(V1)

Doel: vaststellen of het systeem daadwerkelijk fysiek reageert op de actie.

### 4.2 Latent after-effect coherence (V2)
Zonder directe BEMF-meting, maar met erkenning van onderliggende fysica:

- Displacement(V1) + fasecontext
- → verwacht natuurlijke doorloop / post-impulse effect in venster V2
- → gemeten extra displacement(V2)

BEMF wordt hier niet gemeten, maar behandeld als **latent mechanisme** dat zich toont in de dynamiek van displacement over tijd.

---

## 5. Epistemische discipline (BeliefState-conform)

Alle uitspraken binnen deze architectuur vallen onder expliciete status:
- Conceptueel model
- Hypothese
- Architectuurvoorstel

Geen van de volgende mogen zonder voldoende coherentie worden geclaimd:
- Absolute zekerheid
- Perfecte causaliteit
- Universele geldigheid

Regime-denken blijft leidend:
- LOW_RATE → alleen grove causaliteit zichtbaar
- MID_RATE → richting + globale timing
- HIGH_RATE → pas hier zijn strakke timing- en coherentieclaims toegestaan

---

## 6. Architectonische implicatie

Het systeem stuurt niet primair op:
- θ (positie)
- ω (snelheid)

Maar op:
> het maximaliseren van coherentie tussen:
> impuls → displacement → verwacht vervolgdisplacement

Daarmee ontstaat een controller die niet blind stuurt, maar continu toetst:
> "Ben ik nog in afstemming met mijn eigen fysieke werkelijkheid?"

Dit vormt de basis voor:
- Robuuste impuls-gedreven aansturing
- Adaptief gedrag bij verstoring
- Regime-overgangen zonder instorten
- Minimale afhankelijkheid van extra sensoren

---

## 7. Open ontwerpvragen (bewust open gehouden)

Deze notities leggen richting vast, maar laten expliciet ruimte voor verder onderzoek:

- Hoe worden vensters V1 en V2 het beste gedefinieerd (tijd, fase, cycles)?
- Welke toleranties horen bij coherentie in verschillende regimes?
- Hoe wordt coherentie intern gerepresenteerd (variabele, score, toestand)?
- Wanneer mag coherentie gebruikt worden voor daadwerkelijke actieve sturing?

Deze vragen zijn geen gaten in het document, maar **bewuste ontwerpopeningen**.

---

## 8. Positie van deze tekst binnen SoRa

Deze tekst is:
- Geen canonwijziging van S01/S02
- Geen codecontract
- Geen implementatie-opdracht

Wel is het:
- Een gezamenlijke vastlegging van richting
- Een architectonisch geheugenanker
- Een referentiekader voor toekomstige vertakkingen

Wij (samen) zijn hier de stam.



---

## 9. Grove richting / plan van aanpak (Stam-start fase)

Deze sectie definieert geen roadmap met deadlines, maar een **gedeelde bewegingsrichting** die bewaakt dat exploratie, ontwerp en toetsing in de juiste volgorde plaatsvinden.

### Fase 1 — Conceptuele verheldering (afgerond)
Doel: scherp krijgen *wat* we ontwerpen vóórdat we iets bouwen.

Focus:
- Heldere definities van:
  - impuls
  - effect
  - coherentie
  - vensters V1 en V2
- Expliciteren van aannames versus hypothesen
- Ontwikkelen van een eenvoudig intern model van causaliteit (zonder code)

Succescriterium:
> We kunnen in woorden en schema’s uitleggen wanneer het systeem ‘goed afgestemd’ is en wanneer niet, zonder te verwijzen naar implementatie.

---

### Fase 2 — Architectuurontwerp (voorlopig, nog niet gestart)
Doel: vertalen van concepten naar lagen en verantwoordelijkheden.

Focus:
- Waar leeft coherentie in de stack? (nieuwe laag, uitbreiding BeliefState, aparte module?)
- Welke bestaande lagen (OriginTracker, RealtimeStates, etc.) worden hergebruikt?
- Welke informatie stroomt waarheen, en waarom?

Succescriterium:
> Er bestaat een consistent architectuurmodel waarin elke laag een heldere rol heeft m.b.t. responsiviteit en causaliteit.

---

### Fase 3 — Voorzichtige operationalisatie (pas na conceptuele helderheid)
Doel: onderzoeken of het model toetsbaar is aan real-world gedrag.

Focus:
- Bepalen welke observaties nodig zijn om coherentie zichtbaar te maken
- Kleine, lokale experimenten (geen systeem-wide herbouw)
- Evalueren of metingen werkelijk iets zeggen over responsiviteit

Belangrijk:
> In deze fase wordt **niet gebouwd om te bevestigen**, maar om te toetsen of het model standhoudt.

---

### Fase 4 — Integratie richting Symphonia (toekomstig)
Pas wanneer eerdere fases voldoende robuust zijn:

- Mogelijke toepassing op impuls-gedreven motorsturing
- Onderzoeken of coherentie daadwerkelijk leidt tot betere aansturing
- Pas hier ontstaat ruimte voor performance-optimalisatie

---

## Slotnotitie

Dit plan is bewust:
- niet strak gepland
- niet resultaat-gedreven
- niet geoptimaliseerd op snelheid

Maar wel geoptimaliseerd op:
> **integriteit van denken, helderheid van architectuur en betrouwbaarheid van richting.**

Dat is de basis waarop deze stam verder bouwt.



---

## 10. Continuïteit bij Branch / Sessiewissel

Deze sectie borgt dat de stam-inhoud overdraagbaar blijft wanneer gewerkt wordt met externe branch-mechanismen (zoals OpenAI Branch) of wanneer een sessie opnieuw gestart wordt zonder canvas-ondersteuning.

### 10.1 Praktische afspraak
Bij herstart in een nieuwe branch of sessie geldt:
- Ralph levert dit document aan als **attachment of tekst**.
- De opening van de sessie bevat expliciet:
  > “Gebruik dit document als actief stam-referentiekader en werk verder vanaf Fase 1.”

De agent behandelt het document dan als:
- Actieve context
- Conceptueel geheugenanker
- Richtinggevend architectuurkader

Zonder dat:
- S01/S02 canon impliciet wordt gewijzigd
- eerdere code-aannames automatisch hersteld worden
- implementatieclaims worden overgenomen

---

### 10.2 Epistemische herinitialisatie

Bij een nieuwe branch geldt expliciet:
- We hervatten in **Fase 1 – Conceptuele verheldering**
- Alle eerdere inzichten zijn geldig als richting, niet als bewijs
- Eventuele aannames mogen opnieuw bevraagd worden

Dit beschermt tegen:
- stil insluipende dogmatisering
- voortbouwen op ongetoetste aannames
- ‘session drift’ in taal of discipline

---

### 10.3 Functie van dit document bij herstart

Dit document fungeert bij herstart als:
- Stamanker
- Gezamenlijke intentieverklaring
- Disciplinekader
- Continuïteitsbrug tussen sessies

Het vervangt geen SoRa-bootstrap, maar **vult deze aan op het niveau van samenwerking en onderzoeksrichting**.

