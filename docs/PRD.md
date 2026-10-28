# Pathya — Product Requirements Document

> **Your medicines, arranged correctly.** Enter what you're on; get back a conflict-free daily
> schedule that respects every absorption rule your prescriptions actually carry.

- **Status:** Draft v0.1
- **Last updated:** 2026-10-28
- **Owner:** Shrikant Shingare

---

## 0. How to read this document

Every item below is tagged:

- **[LOCKED]** — a decision explicitly made and approved. Do not change without a new decision.
- **[PROPOSED]** — my recommendation, awaiting your sign-off. Nothing here is final.
- **[OPEN]** — an unresolved question captured for a future decision.

Four decisions came from your answers and are **[LOCKED]**: curated rules table, India-first,
real product, Next.js + TypeScript. Everything else is flagged for you.

## [Edit] The below names do not suits well, because the make goal to the project is to optimize of taking supplments (the current, but in future we are thinking of adding like anazlye the diet and then give a detailed report what coudl be optimized like taking differnt high in diffiernt Micronutrient fodd with what and when to avoid same as supplements.)
**Name — [OPEN].** *Pathya* (पथ्य) is the Ayurvedic term for the regimen of diet and conduct
appropriate to a treatment — literally "what suits the body." It is exactly what this product
computes, it is India-native, and it is unclaimed in this category. Used as a placeholder
throughout; rename freely.

---

## 1. Summary

## [Edit] Also beside telling when to take, it should also tell which supplments to avoid taking with this and also any high in food like avoding high in calcium food with iron supplments,etc
**Pathya** takes the medicines and supplements a person is actually on, plus the times they
actually eat, and computes a **conflict-free daily schedule** — which item at which hour, on an
empty stomach or with a fatty meal, spaced from the things it must not meet.

It is not an interaction checker and not a reminder app. Those two categories are both crowded
(§2). Pathya occupies the gap between them: the *arrangement* problem. Given nine items, three
meal times, and roughly forty absorption constraints, what does a correct day look like?

The engine never alters a doctor's dose or frequency. It arranges strictly *within* the
prescriber's instruction, and when correct arrangement is impossible it says so and escalates
rather than quietly producing a schedule that looks fine and isn't.

---

## 2. Background & motivation

### 2.1 The market gap

The existing market splits cleanly in two, and nothing occupies the middle.

| Category | Examples | What it answers | What it doesn't |
|---|---|---|---|
| **Interaction checkers** | saviMon, MedSafeScan, Zets, WebMD, Supplement Safety Bible | "Is this pair risky?" | When to take anything |
| **Reminder / adherence** | Medisafe, MyTherapy, Round Health | "Did you take it?" — at times *you* chose | Whether those times are correct |
| **Supplement trackers** | StackIQ, SuppCo, Cronometer | What's in your stack, overlaps | Prescription medicines, scheduling |

The decisive data point: **saviMon** is the closest existing product to this idea and the most
complete — 40,000+ curated drug-food and drug-supplement alerts, 140k+ products, 73 nutrients,
HealthKit and Health Connect sync, free on iOS and Android — and it **explicitly does not offer
timing guidance**. StackIQ is the only product that surfaces timing at all; it is
supplement-focused and gates the analysis behind a Pro tier. Of the reminder apps, MyTherapy has
no timing or interaction guidance, Medisafe treats supplements as generic pills, and for Round
Health reminders are the entire product.

So the checkers tell you a pair conflicts and leave you to work out the schedule on paper. The
reminder apps will faithfully ping you at a time that is wrong. **Nobody computes the schedule.**

Full competitor detail and sourcing: `docs/market-research.md`.

### 2.2 Why the problem is real

A person on levothyroxine with a calcium supplement and a five-day course of ciprofloxacin is
holding four separation rules, two food rules, and three fixed frequencies in their head at
once. The constraints interact — and as §4 of `docs/rules-engine.md` demonstrates, that exact
regimen is **genuinely unsatisfiable** against ordinary Indian meal times. No amount of care
solves it; it needs a prescriber's decision. Today nothing tells the person that. They take
everything at breakfast and the levothyroxine quietly under-absorbs for a week.

### 2.3 Why now

The free drug-interaction APIs have shut down — NLM's RxNav interaction endpoints in January
2024, DrugBank's free checker and DDI API in March 2026. The commodity "throw two drug names at
an API" product just got harder to build, while the durable asset — a reviewed, cited rules
table — got relatively more valuable. The timing is in favour of the curated approach.

---

## 3. Goals & non-goals

### 3.1 Goals

1. **Compute a correct daily schedule**, not a list of warnings.
2. **Be trustworthy enough to show a pharmacist** — every rule cited, every claim checkable.
3. **Never overstep a prescriber.** Arrange within the instruction; escalate when arrangement fails.
4. **Work for Indian prescriptions**, where no brand registry exists to lean on.
5. **Handle the temporary course** (§7.2) as a first-class case, not an edit.
6. **Fail honestly** — "no data for this one" and "this cannot be arranged" are first-class outputs.


## [Edit]
### 3.2 Future Goals
1. Get a detailed things what they eat and then tell when to take any Micronutrient rich food and when to avoid

### 3.3 Non-goals

- Not an **interaction checker** — a conflict with no timing consequence is not the product.
- Not a **reminder app** in V1 (§8.1) — the incumbents do notifications well.
- Not **diagnostic**, and never a dose or therapy recommendation.
- Not a **nutrition coach** (§8.4).
- Not a **telemedicine or pharmacy** product — no consultations, no fulfilment.

---

## 4. Target audience & personas — [PROPOSED]

**Primary market:** Indian adults managing a multi-item daily regimen, self-serve.

| Persona | Who | Need |
|---|---|---|
| **The Polypharmacy Patient** (wedge) | 45+, 3–8 daily items, usually a thyroid, BP, diabetes or cardiac prescription plus supplements | "I take nine things. Am I taking them at the wrong times?" |
| **The Supplement Stacker** | 25–40, 4–10 self-chosen supplements, possibly one prescription | "Is my stack fighting itself, and am I overdosing on anything?" |
| **The Caregiver** | Adult child or spouse managing a parent's regimen | "Give me one sheet I can put on the fridge and show the chemist." |

**Adoption motion:** the Caregiver and the Stacker are the sharers — the one-pager (§7.7) is the
distribution mechanism. The Polypharmacy Patient is the retention case, because their regimen
changes (§7.2).

---

## 5. Product shape

Three objects, one engine:

```
  REGIMEN                 ANCHORS                      RULES TABLE
  what you're on          when you eat / wake / sleep  curated + cited
  dose + frequency        fat / dairy / caffeine flags groups, not pairs
        │                        │                           │
        └────────────────────────┼───────────────────────────┘
                                 ▼
                          SCHEDULER (pure, deterministic)
                                 │
          ┌──────────────────────┼──────────────────────┐
          ▼                      ▼                      ▼
     THE DAY              HANDLED FLAGS           ESCALATIONS
   slot-by-slot        "spaced 4h from your     "spacing cannot fix
   with reasons         calcium — here's why"     this → ask your doctor"
```

> **Design principle:** the user never learns the word "interaction." They enter what they take
> and when they eat; they receive a day. The pharmacology stays inside the engine.

---

## 6. The safety rails — [LOCKED]

These are hard product constraints, not guidelines. Every feature below is subordinate to them.

1. **Never contradict a prescriber.** Dose and frequency are *fixed inputs*. The app arranges
   within the doctor's instruction and never alters, splits, or drops a prescribed dose.
2. **Never advise stopping anything.**
3. **Non-spaceable interactions are escalated, never scheduled around.** See §7.5 — this is the
   single most important behaviour in the product.
4. **Unknown items produce an explicit "no data" state**, are excluded from the solver, and are
   never given an inferred rule.
5. **Every rule shows its citation.** The user, or their pharmacist, must be able to check it.
6. **No runtime LLM on the safety path.** The app reads only the reviewed table. An LLM may be
   used offline to *draft* candidate rows for human verification, never to answer at query time.
7. **An unsatisfiable regimen is reported as unsatisfiable.** The solver never silently relaxes a
   constraint to produce a clean-looking day.

---

## 7. V1 scope — the minimal version

This is the "upload your ongoing medicines and supplements → build an editable profile →
suggest optimal times" product you asked for, and nothing more.

### 7.1 Adding items — the three-tier input — [LOCKED]

India-first means there is no brand→molecule registry to rely on (§9.2). The way around it:
**Indian strips are legally required to print the generic composition.** So input degrades
gracefully and never dead-ends.

| Tier | Flow | Coverage |
|---|---|---|
| 1 | Search a curated brand-alias map — Thyronorm, Shelcal, Ecosprin, Glycomet, Metformin… | Top ~300–500 brands = large majority of real prescriptions |
| 2 | Not found → "type the composition printed on the strip." Parse `Thyroxine Sodium IP 50mcg` → molecule + strength | Everything sold in India |
| 3 | Unparseable → save as **unknown**, excluded from the solver, shown with an explicit "no data" badge | Residual |

Per item, captured: molecule(s) + strength, **dose and frequency exactly as the doctor said**
(OD / BD / TDS / QID / weekly / SOS), and a free-text "doctor's note" field for instructions the
schema can't hold. Frequency is stored as prescribed, never normalised away.

Supplement items additionally capture elemental amounts per ingredient, for §7.6.

### 7.2 The editable profile and temporary courses — [LOCKED]

Your flu case is not an edge case — **it is the highest-value moment in the product.** A short
antibiotic course is precisely when timing matters most, because that's the week someone's daily
calcium silently wrecks their ciprofloxacin.

So this is a data-model decision, not an edit screen:

- Every regimen entry has `start_date` and an optional `end_date` / `duration_days`.
- Adding a temporary course **recomputes the whole schedule** and shows a diff against the
  normal day: *"for the next 5 days, these three things move."*
- When the course ends, the schedule **auto-reverts**. No cleanup required.
- Items can be **paused** (travelling, out of stock) without being deleted.
- Full edit: add, remove, change dose or frequency, reorder.

This also answers the "someone might only use it once" worry directly: regimens change —
courses, dose adjustments, new supplements — and each change is a reason to return.

### 7.3 Meal anchors — [LOCKED]

Your calorimeter intuition was right, and so was your doubt: health platforms largely export
rather than stream. **But the solver does not need live intake data.** It needs anchors.

Onboarding asks for:

- Wake time, sleep time
- Breakfast / lunch / dinner times (approximate is fine)
- Per meal, three checkboxes: **contains fat**, **contains dairy**, **contains caffeine**

Three time pickers and nine checkboxes — about thirty seconds — and the entire schedule unlocks.
Fat matters for vitamin D, A, E, K and omega-3 absorption; dairy and caffeine are themselves
constraint participants.

**Why this is the right design, not a compromise:** a fixed-routine eater (your own example) is
served perfectly by declared anchors, and a chaotic eater isn't served by live data either —
their schedule would thrash. Anchors are also editable per weekday, which covers most real
variation. Nutrition import becomes a V2 refinement (§8.2), never a dependency.

### 7.4 The schedule — [LOCKED]

The primary output. A day broken into slots, each showing what to take, with what, and **why**:

```
07:00   Levothyroxine 50mcg        empty stomach · 90 min before breakfast
08:30   ☕ Breakfast
13:30   Shelcal 500 (Ca + D3)      with food · 6½h after levothyroxine
21:00   Shelcal 500 (Ca + D3)      with food · 7½h after the first dose
        Vitamin D3 60k  (Sundays)  with dinner — your fattiest meal
```

Every line carries its reason, because the reason is what makes the schedule followable and
checkable. Full algorithm in `docs/rules-engine.md` §3.

### 7.5 Flags — the spaceable / clinical split — [LOCKED]

**This is the central safety idea of the product.** The two categories must never be mixed:

| | **Spaceable** | **Clinical** |
|---|---|---|
| Mechanism | Absorption — chelation, pH, competition | Pharmacodynamic or metabolic |
| Example | Levothyroxine + calcium; ciprofloxacin + iron | St John's Wort inducing CYP3A4; two serotonergic drugs stacking; warfarin + vitamin K swings; ashwagandha + thyroid medication |
| Fixed by timing? | **Yes** | **No — not at any interval** |
| App behaviour | Solver resolves it; explains the spacing | Solver **refuses** to resolve; escalates to "discuss with your doctor or pharmacist" |

An app that reschedules around a *clinical* interaction is worse than one that says nothing: it
manufactures false reassurance about a risk that is still fully present. I'd be surprised if most
products in §2.1 draw this line at all — it is both the honesty of the engine and the hardest
thing for a competitor to retrofit, because it has to be encoded per rule from the start.

Flags are presented in two separate panels, never one merged list, each item citing its source.

### 7.6 Duplicate and upper-limit detection — [LOCKED]

A multivitamin plus a separate zinc plus a weekly D3 can quietly exceed tolerable upper intake.
Sum elemental amounts per ingredient across the whole regimen; compare against ULs; flag
overlaps ("three of your items contain vitamin D — combined 4,600 IU/day").

Cheap once composition is captured, a real safety win, and none of the reminder-tier apps do it.
**[OPEN]** UL reference source — NIH ODS values are the obvious default; confirm whether ICMR-NIN
RDA/UL figures should take precedence for an Indian audience.

### 7.7 The pharmacist one-pager — [LOCKED]

A printable/PDF single page: current regimen, the computed schedule, flagged items with
citations, and the open questions to ask. In India the pharmacist is often the de facto first
point of contact, which makes this the culturally correct artifact — and it keeps the product
firmly on the right side of §6.1: **it never overrides a prescriber, it prepares you to ask one.**

It also reframes single-use from a retention bug into a legitimate, valuable use case.

### 7.8 Consent, disclaimers, and data rights — [LOCKED]

See §9 for the regulatory basis. In V1: a standalone plain-language consent notice before any
health data is collected, a persistent "educational, not medical advice" disclaimer, and working
**export** and **delete-everything** functions. Built in from the first commit, not retrofitted.

### 7.9 AYUSH botanicals in the rules table — [LOCKED]

India-first makes this non-optional. Ashwagandha, triphala, giloy, brahmi and similar are
extremely common and several carry real interactions — ashwagandha with thyroid medication and
sedatives; giloy with reported hepatotoxicity. Curate ~15 of the most common with **explicit
confidence levels**. An honest `confidence: low` beats silence, and silence is precisely what
every US-built competitor offers an Indian user here.

---

## 8. Deferred — with reasoning

Each of these is deliberately out of V1. The reasoning matters more than the list.

### 8.1 Reminders and notifications — V2, first addition

Commodity feature, needs push infrastructure, and it is the one thing the incumbents already do
well. Shipping reminders before the schedule is trustworthy would make Pathya a worse Medisafe.
Shipping them *after* makes Pathya the only app whose reminders fire at defensible times.

### 8.2 Nutrition import — V2

Cronometer / MyFitnessPal CSV import, used to *suggest corrections* to declared anchors: "your
logs say breakfast is usually 10:40, not 08:00 — update?" Worth noting HealthKit and Health
Connect *do* permit live nutrition reads with permission; the real obstacle is that few calorie
trackers write to them, which is why import is the pragmatic path. **Live sync is never required.**

### 8.3 Strip-photo OCR — V2

The obvious successor to §7.1 tier 2. Not a V1 dependency, because typing a composition works.

### 8.4 Micronutrient food timing — deliberately reshaped

Your "when to eat high-iron food" idea. Positive nutrition timing has substantially weaker
evidence than absorption pharmacology, and building it well is a different product. The honest
minimal version is the **inverse**, which the existing engine already yields for free:
**food-avoidance windows.** "No dairy or coffee 08:00–10:00." "No grapefruit at all while on
this statin." Same rules table, same solver, defensible claims. **[PROPOSED]** ship avoidance
windows inside V1's schedule view; leave positive coaching out indefinitely.

---

## 9. Regulatory & privacy

### 9.1 DPDP Act 2023 + DPDP Rules 2025 — [LOCKED]

MeitY notified the DPDP Rules on 13 November 2025. Most substantive duties — standalone consent
notice, security safeguards, breach reporting, retention limits, data-principal rights — **apply
from 13 May 2027.** There is runway, but the obligations are cheap to build in now and expensive
to retrofit:

- Consent must be free, informed, specific, unambiguous, and withdrawable; stored with version,
  timestamp and purpose.
- Notice must stand alone, itemise what is collected, and explain purpose in plain language.
- Rights to access, correct and erase must actually work (§7.8).
- **[OPEN]** Data residency. Several practitioner summaries assert an India-residency
  requirement for health data; the Act's own transfer restrictions are narrower and tied to
  Significant Data Fiduciary designation. **Confirm with counsel before choosing a host region** —
  defaulting to an Indian region costs nothing and removes the question.

### 9.2 Medical device positioning — [PROPOSED]

Pathya presents cited reference information and arranges within a prescriber's existing
instruction. It makes no diagnosis, recommends no therapy, and alters no dose (§6). That posture
— plus prominent disclaimers and the §7.7 "go ask your pharmacist" framing — is what keeps it an
information tool rather than a clinical decision aid. **[OPEN]** confirm CDSCO software-as-medical-
device posture with Indian regulatory counsel before public launch.

### 9.3 Data sources and licensing — [LOCKED]

The curated table is built from **openly citable** sources — FDA labels via openFDA, NIH ODS fact
sheets, DSLD, and published clinical guidance. No licensed commercial interaction database is
used, so no licence constrains distribution. Every row records its source URL and access date.

---

## 10. Technical approach — [LOCKED]

- **Next.js + TypeScript**, one codebase.
- **The rules engine is a pure, dependency-free TypeScript module** with no I/O, no network, and
  no framework imports. It is the part that must be provably correct, so it must be trivially
  testable: regimen + anchors + rules in, schedule + flags out.
- **The rules table is YAML in the repo**, reviewed by pull request like code. A rule change is a
  diff someone can read — that is the entire reason the curated approach was chosen.
- **Golden-file tests**: a corpus of real regimens with hand-verified expected schedules. Any
  rules-table edit that changes an existing schedule must change a golden file visibly.
- Storage, auth and hosting: **[OPEN]** — see §13.

Detail in `docs/rules-engine.md`.

---

## 11. Milestones — [PROPOSED]

| Milestone | Goal | Done when |
|---|---|---|
| **M0 — Rules spine** | The long pole, first | ~30 molecules + ~15 supplements + groups in YAML, each cited; schema frozen |
| **M1 — Solver** | Pure engine | Scheduler passes the `rules-engine.md` §4 worked example, including the unsatisfiable branch |
| **M2 — Regimen UI** | Input + profile | Three-tier add, dose/frequency capture, edit, pause, temporary courses |
| **M3 — The day** | Primary output | Schedule view with per-line reasons; spaceable/clinical panels |
| **M4 — Safety features** | Trust | UL/duplicate detection, one-pager PDF, consent + export + delete |
| **M5 — Table expansion** | Coverage | ~120 molecules, ~40 supplements, ~15 AYUSH, ~300 brand aliases |
| **M6 — Beta** | Real users | Pharmacist review of the table; onboarding; 10 real regimens validated |

The sequencing is deliberate: **the rules table is the long pole and the app is a thin shell over
it.** Building the shell first would mean guessing at the schema the table needs.

---

## 12. Success metrics — [PROPOSED]

- **Correction rate** — % of new regimens where the computed schedule differs from what the user
  was already doing. This is the product's whole claim; if it's near zero, the thesis is wrong.
- **Escalation rate** — % of regimens producing a clinical flag or an unsatisfiable result. These
  are the highest-value outputs.
- **One-pager exports per regimen** — proxy for "did they actually take this to a professional."
- **Return-on-change** — % of users who come back to add a temporary course (validates §7.2).
- **Table integrity** — % of shipped rules with a verified citation. Target: 100%.

---

## 13. Open questions — [OPEN]

1. **Name** (§0) — Pathya, or something else.
2. **Data residency** (§9.1) — confirm with counsel; default to an Indian region meanwhile.
3. **CDSCO SaMD posture** (§9.2) — confirm before public launch.
4. **UL reference source** (§7.6) — NIH ODS vs ICMR-NIN.
5. **Storage and auth** (§10) — local-first with optional sync, or accounts from day one? Affects
   DPDP surface materially. Local-first is the cheaper compliance posture.
6. **Pharmacist review** — who verifies the table before beta? This gates launch credibility.
7. **Multi-language** — Hindi and regional languages. V1 English-only, but it affects the
   copy architecture if it's coming.
8. **Monetisation** — unaddressed. The one-pager is the natural paid artifact, but charging for
   a safety output needs thought.

---

## 14. Decision log

| # | Decision | Value |
|---|---|---|
| 1 | Product | Scheduling engine, not an interaction checker or reminder app |
| 2 | Knowledge base | Hand-curated YAML rules table, every row cited |
| 3 | Market | India-first |
| 4 | Intent | Real product for users |
| 5 | Stack | Next.js + TypeScript; engine as a pure module |
| 6 | Input strategy | Three-tier: brand alias → strip composition → explicit unknown |
| 7 | Meal data | Declared anchors; import is V2, live sync never required |
| 8 | Temporary courses | First-class data model (`start_date`/`end_date`), not an edit flow |
| 9 | Interaction model | Spaceable vs clinical, encoded per rule |
| 10 | Rule encoding | Groups, not drug pairs |
| 11 | Unsatisfiable regimens | Reported honestly; never silently relaxed |
| 12 | LLM use | Offline drafting only; never on the query-time safety path |
| 13 | AYUSH botanicals | In scope for V1 with explicit confidence levels |
| 14 | Micronutrient coaching | Reshaped to food-avoidance windows; positive coaching out |
| 15 | Reminders | Deferred to V2, after the schedule is trustworthy |
