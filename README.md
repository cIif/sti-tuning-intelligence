# 🏁 STI Tuning Intelligence

> A safety-gated, speed-density **tuning pipeline** for one built 2018 WRX STI — it reads the car's datalogs, reproduces a proven VE-first method to generate VE/boost ROM candidates, gates every map behind hard fail-closed checks, and acts as a careful tuner-mechanic: **deciding and prescribing, but never relaxing a safety line.**

![tests](https://img.shields.io/badge/tests-560%20passing-brightgreen)
![pipeline](https://img.shields.io/badge/decode%20pipeline-39%2F39-brightgreen)
![python](https://img.shields.io/badge/python-3.13-blue)
![ui](https://img.shields.io/badge/UI-NiceGUI-ff453a)
![tuning](https://img.shields.io/badge/DimeMod-speed--density-orange)
![safety](https://img.shields.io/badge/gates-fail--closed-success)

**What it is, plainly:** a **deterministic pipeline** — *not* an "AI tunes your car" gimmick. It decodes the actual ECU tables to real units, applies the math of a proven speed-density (VE-first) method, and gates every candidate behind hard safety checks. A human and a wideband still do the real tuning — this assists, diagnoses, and blocks unsafe maps.

The car: **a built 2018 WRX STI — EJ257 short block, a large Garrett-style turbo, E85, big injectors, on a DimeMod speed-density calibration.** Every value here was learned from *this* build's real data and is calibrated to it.

> ⚠️ **Personal senior-design project for one specific car.** It is *not* a substitute for a professional tuner or a dyno. Don't flash anything off it without a wideband, a dyno when it matters, and your own judgment.

---

## How it started

I'm **not a professional tuner** — I'm an engineering student who's spent **~a year tuning this one car**. What I *can* do is analyze data, so I built an analyzer to do exactly that — and that's how I learned to **decode and re-code the ROM tables and maps** in the first place. The analyzer grew into this whole pipeline.

## Built on a year of real data

It isn't guessing from defaults. The whole thing is grounded in that **year of actually tuning this car** — every map revision and datalog, plus a vetted reference library covering the build, the parts, the fuel, and the tuning concepts behind each decision. The pipeline learns the **method** and the **safe envelope** directly from that history, so its corrections and its limits come from what this car has actually done, not from a generic table.

## Where it's at (honest status)

- **Stage 1 (base fueling / VE) — in progress, going well.** Across attempts the VE corrections it wants are *shrinking* (median ≈ 5% → ≈ 2%) — the air model is converging on target. Idle sits steady at stoich with a tiny closed-loop trim; cruise fueling reads dialed (0 cells off target).
- **No WOT pulls yet — by design.** The method is a ladder: **idle → cruise → ~50% pull → full pull → only then add boost.** The app won't green-light a pull until idle + cruise are confirmed clean (and not misfiring).

---

## The loop it automates

```
 datalogs ─┐
 base ROM ─┴─▶ features dataset ─▶ VE / boost candidate ─▶ ⛔ SAFETY GATES ─▶ review + package ─▶ pre-flash GO/NO-GO ─▶ EcuFlash
              (build_ml_dataset)   (generate_stage)        (gate_candidate,     (report/package)   (flash_checklist)
                                                            knock_margin,
                                                            safety_margin)
```

**The method it reproduces** (learned from a year of this car's maps + logs): **VE first → hold timing → build boost incrementally → never stack boost + timing in one map** (the "Send-It" pattern that precedes engine death). A few maps perfect a stage; a hardware limit gets a *hardware* prescription.

---

## ✨ Features at a glance

| Subsystem | What it does |
|---|---|
| 🛡️ **Safety gates** | A multi-layer, **fail-closed** wall (ROM identity, per-cell envelope, boost×timing coupling, boost ceiling, knock-zone, absolute AFR-lean ceiling, VE-lean, stage readiness, pre-flash GO/NO-GO) that bounds every candidate. |
| ⚙️ **Tuning pipeline** | Logs → features → **VE correction** (sparse anchors or a full smoothed surface) → **boost target** scaled to the stage goal → gated candidate `.bin`. |
| 🧠 **Decision engine** | ADVANCE / KEEP-TUNING / BLOCKED per stage, a per-attempt logging ladder, a single danger-first "do this next" directive, auto-diagnosis of tune-vs-hardware, and an outcome-learning loop that calibrates the score against your real results. |
| ❤️ **"Everything dialed" health** | Idle quality + idle trim, cruise-VE trims, misfire/roughness, **per-cylinder knock balance**, exhaust-AVCS (SAVCS) parked-vs-actuating, fuel delivery, ethanol dropout — each judged TUNE vs HARDWARE. |
| 📊 **Interactive log graphs** | A DataZap-style datalog viewer — overlay/compare logs, normalized single axis, smoothing, hover crosshair, knock markers, WOT-region shading, a virtual dyno. |
| 🔒 **Data integrity (the "Brain")** | A centralized invariant guard + base-SHA provenance that kills silent data bleed across cars / stages / attempts / re-derives. |
| 🖥️ **Web UI** | A 12-tab NiceGUI app: tune, review, diagnose, graphs, build health, maps, history, a car-aware assistant, and a cited-research tool. |
| 🔬 **Decode & knowledge** | Decodes ~45 ECU tables to real units, a verified DimeMod logger-param catalog, and a cited knowledge base. |

---

## 🛡️ Safety gates — the centerpiece

Every gate is designed to **FAIL CLOSED**: when a signal is missing, undecodable, or the corpus is empty, it refuses or flags HIGH RISK rather than passing. **Owner-review values** (wastegate spring, boost ceiling, AFR lean limit, target whp) are surfaced and honored — **never silently changed.**

| Gate | Blocks / flags | Fails closed |
|---|---|:--:|
| **ROM identity** (`validate_rom`) | A base or candidate that isn't this car's specific DimeMod calibration — a foreign ROM reads garbage at the hardcoded offsets | ✅ |
| **ROM size** | A candidate whose byte length differs from its base (corrupt/wrong image) | ✅ |
| **Per-cell envelope** | Any VE/timing/AFR cell outside the per-cell `[min,max]` learned from the clean map corpus | ✅ (empty corpus ⇒ all violate) |
| **Diff-step sanity** | A single-cell jump larger than the biggest step ever made historically | ✅ |
| **Boost × timing coupling** | Raising **both** boost and timing vs base — the "Send-It" stack | ✅ |
| **Boost hard ceiling** | Peak commanded boost over the absolute backstop (owner-review) | ✅ |
| **Boost review ceiling** | Flags the upper band as dyno+wideband (Stage-3) territory (advisory) | — |
| **Boost step cap** | Raising any boost cell more than a few psi in one candidate (forces gradual builds) | ✅ |
| **Denylist** | Building from / emitting a known engine-failure ("Send-It") tune — identity by content-hash, survives renames | ✅ |
| **Timing-in-knock-zone** | Raising timing in a historically-knocking RPM×load zone | ✅ (no knock map ⇒ HIGH RISK) |
| **VE-only-lean-at-boost** | Reducing boost-cell VE >15% (leans the real WOT mixture — the meltdown direction) | ✅ |
| **Absolute AFR ceiling** | Commanded open-loop AFR in the **sustained-WOT box** (≥2.4 g/rev AND ≥3200 rpm) leaner than the rich floor (≈ λ0.82) | ✅ |
| **Boost-raised-while-leaned** | Adding boost on a too-lean mixture (base-relative) | ✅ |
| **Forward safety margin** | A 0–100 gradient of headroom to each danger line — surfaces the *tightest* before a gate trips | ✅ |
| **Stage readiness** | ADVANCE / KEEP-TUNING / **BLOCKED** — blocks boost on an unvalidated VE; holds on knock, lean WOT, un-converged VE, rough idle, misfire, or wrong wastegate headroom | ✅ |
| **Pre-flash GO / NO-GO** | Bundles size + denylist + identity + gate_candidate + knock_margin + checksum + base-SHA + wideband + ethanol≠0 into one read-only verdict | ✅ |
| **Subaru checksum** | Verifies/fixes the SH7058 checksum so the candidate is well-formed (orthogonal to the content gates) | — |

<details><summary><b>Wastegate-spring logic (be the mechanic)</b></summary>

Stage readiness reads wastegate **duty-cycle headroom**, not just whether boost hit target:

- **Spring too big** — target made at WGDC < 5% (no room to pull boost for cold/altitude) → prescribes a **softer spring**.
- **Out of wastegate / spring too small** — can't reach target with WGDC pinned ~100%, or made target at >95% duty → prescribes a **stiffer spring / 4-port EBCS**.
- **Target window:** boost at WOT with **30–40% WGDC headroom, ≥15 psi**.
</details>

---

## ❤️ "Everything dialed" health rollup

Beyond AFR — every log is auto-diagnosed for the things that have to be right *before* you add power, each classified **TUNE** vs **HARDWARE**:

| Check | What it catches |
|---|---|
| **Idle quality** | rpm stability (speed-gated so coast-down isn't mistaken for idle) + idle AFR + the **closed-loop idle trim** — a big persistent trim means the idle VE is off, not just "lumpy cam". |
| **Cruise VE — trims** | the per-cell **median closed-loop trim** is the real VE error; an *independent* confirmation the cruise fueling is dialed (not just measured AFR). |
| **Misfires** | the per-cylinder **Roughness Monitor** channels (the ECU's own misfire detector) — flags counts, distinguishes a real misfire from built-cam idle lumpiness, says so when they aren't being logged. |
| **Per-cylinder knock balance** | one cylinder whose knock-sum rises harder than the others — the single-weak-cylinder failure mode — flagged HIGH before WOT. |
| **Exhaust AVCS (SAVCS)** | judges the exhaust cam by **OCV duty/current modulation**, not cam-advance angle — so a *parked* single-AVCS cam reads correctly as "not actuating," and the tune adapts around it. |
| **Fuel delivery** | rail-vs-manifold ΔP under boost (the failing-pump catch) + injector-duty / fuel-supported-HP ceiling. |
| **Ethanol** | flex-sensor voltage-dropout detector — **never WOT while ethanol reads 0** (the ECU would think it's on pump gas → lean). |

The same signals gate the **per-attempt ladder**: idle → cruise → ~50% pull → full pull → boost. Cruise/idle/misfire must be clean before it suggests a pull; once you're logging **pulls only** (the normal workflow once part-throttle is dialed), it judges the pull and doesn't nag for cruise it already has.

---

## 📊 Interactive log graphs

A built-in, DataZap-style datalog viewer (the **Graphs** tab) — read-only, never touches the tune:

- **Overlay & compare** one or several logs (compare pulls/attempts).
- **One normalized y-axis** — every channel scaled to 0–100% of its own (robust 2/98-percentile) range so they all disperse instead of squashing; hover shows the **real** values + units.
- **Many channels at once**, smooth splines + a moving-average **smoothing** dial, a hover **crosshair**.
- **⚠ knock markers** wherever the log knocked, **WOT-region shading** so you see where the pull is, and **"Focus pull"** auto-zoom.
- **Quick-set presets** (Fueling / Knock / Spool / Cams), remembered selection per car, adjustable height.
- A **virtual dyno** overlay (wheel-hp + torque vs RPM) modelled from the same logs.

---

<details>
<summary><h2>⚙️ Tuning pipeline &amp; generators</h2></summary>

| Module | Role |
|---|---|
| `make_tune.py` | The ROM engine: `TABLES` (verified VE/boost/timing/AFR specs), `read_raw`/`write_raw` (raw↔units), `validate_rom`, `gate_candidate`, `boost_envelope`, denylist. |
| `generate_ve.py` | Per-cell VE corrections from **one** log — closed-loop trims for cruise/idle, wideband-vs-target for WOT — clamped + gated. |
| `aggregate_ve.py` | **Primary Stage-1 route:** aggregates many logs into one full-coverage VE grid, per-cell clamped to the envelope + max historical step; reports the **magnitude** of the remaining corrections (how converged it is). |
| `generate_fuller_ve.py` + `ve_fill.py` | A **full smoothed VE surface** from sparse anchors: damp, interpolate within the data hull only, light smooth, step-cap, confidence-tag — no extrapolation. |
| `generate_boost.py` | Scales a **proven** Target Boost shape to the stage's peak, envelope-clamped, safety-mode capped, denylist-refused. |
| `generate_stage.py` | The staged orchestrator: composes VE + boost into a full candidate `.bin`, leaves timing safe, refuses boost on unvalidated VE. |
| `ve_cross_check.py` | Validates the decoded VE against the ECU's **own** logged SD-VE — confirms decode correctness and that a flashed candidate runs the intended VE. |
| `build_ml_dataset.py` | Turns the log archive into the features dataset (per-log health/perf features, combined rows, byte-level map diffs) with fuel-aware AFR/Lambda handling. |
| `report_candidate.py` / `package_candidate.py` | Human diff report in real units / a tuner-ready `PACKAGE.md` that re-runs every gate and marks **DO NOT FLASH** on any failure. |

</details>

<details>
<summary><h2>🧠 Decision engine, diagnostics &amp; learning</h2></summary>

| Module | Role |
|---|---|
| `stage_readiness.py` | ADVANCE / KEEP-TUNING / BLOCKED + a deterministic 0–1 confidence; the per-attempt ladder, the pre-pull idle/cruise/misfire gate, and the pulls-only handling. |
| `log_ladder.py` | The pure cruise → 50% → 100% → boost rung decision, gated on prior-log quality. |
| `next_action.py` | Folds readiness + spring/boost diagnosis + health into **one** danger-first directive (invents no tuning logic). |
| `autodiagnose.py` | Classifies a log's problem as TUNE vs MECHANICAL / FUEL / CONDITIONS — incl. idle quality + trim, cruise trims, misfire/roughness, per-cylinder knock balance, exhaust-AVCS state, and the boost-capability (spring) call. |
| `logviz.py` | The interactive datalog-graph figure builder (overlay, normalize, smooth, crosshair, knock markers, WOT shading, virtual dyno) — pure + unit-tested. |
| `outcome.py` | Outcome learning — compares the pre-flash 1–10 score band to your recorded good/bad+dyno result and surfaces optimistic/conservative bias. |
| `trends.py` | Cross-attempt direction: VE converging, per-cylinder knock rising, leanest WOT AFR drifting. |
| `afr_residuals.py` | Per-cell measured-vs-commanded AFR map — exactly which VE cells to fix and where to log next (idle band excluded; it's closed-loop). |
| `fuel_delivery.py` / `fuel_headroom.py` | Failing-pump catch (rail-vs-manifold ΔP under boost) and injector-duty / fuel-supported-HP ceiling. |
| `timing_advisor.py` | Per-cell "pull this much base timing" from learned knock — **advisory, never writes a ROM**. |
| `decel_stall.py` / `decel_idle_advisor.py` | Classifies stall events (fueling vs air-control) and maps each to the exact decoded idle/overrun ROM table. |
| `ethanol_health.py` | Flex-sensor voltage-dropout detector — never WOT while ethanol reads 0. |
| `run_diag.py` | Severity-ranked anomaly timeline + boost-transient (overshoot/oscillation) quality. |
| `log_report_card.py` / `log_quality.py` | "Safe to generate a VE map from this log?" gate + a RE-LOG gate for unusable data. |
| `logging_profile.py` | Build-aware RomRaider logging profiles (WIDE / FAST / STALL / SD-VE), dropping channels the build makes meaningless and staying under the ECU's SSM poll budget. |

</details>

<details>
<summary><h2>🔒 Data integrity — "the Brain"</h2></summary>

Centralized so a careless edit can't put wrong data on the engine. **No data bleed** across cars, stages, attempts, or re-derives.

- **`check_integrity` — 8 invariants:** base-map present, no duplicate stages, no orphan stages, acceptance pointer sane, no score-without-candidate, **candidate-base-stale**, candidate-logs-deleted, health-directive coherence. `repair_integrity` applies only whitelisted **non-destructive** fixes and never clears owner acceptance.
- **Base-SHA provenance keystone:** every candidate freezes the `base_sha` it was built from; `candidate_base_stale` catches the **silent re-base bleed** (an upstream VE re-derive under the same `candidate.bin` filename) so score/diff/changelog flag and refuse.
- **Leak-free data model:** each `Car` is a frozen dataclass scoped to its own `output/cars/<id>/` root; atomic unique-temp JSON writes; a corrupt read **raises** (never degrades a save into wiping the config); `.gen` scratch is **unique per run** (the audit-reproduced cross-attempt byte bleed).
- **Lifecycle:** `delete_attempt` / `reset_stage` / `delete_stage` / `reset_tuning` — dynamically delete attempts or stages, swap the base map, or start from scratch; stage chaining self-heals the gap.
- **Frozen change log:** `CHANGELOG.md` between attempts **and** between stages (with per-table reasons and ⟲ cross-stage "refined Stage-1 VE" tags), rendered from a diff snapshot frozen at generation so history can't silently rewrite.
- **Part-Aware-Tuning:** a deterministic part→impact rule table (unknown ⇒ critical) flags the base OUTDATED + `ve_needs_rederive` on a calibration-critical hardware/cam/base change; `backup_car` zips the whole car as a restore point.

</details>

## 🖥️ Web UI (12 tabs)

A dark, single-page **NiceGUI** app (`python src/tuner/webui/app.py`, port `8504`) — per-car selector, stage/fuel badges, everything scoped to the active car (no cross-car leak).

| Tab | Purpose |
|---|---|
| **Dashboard** | Car at-a-glance: safety mode, current base map, stage progress, latest dyno whp. |
| **Tune** | The whole per-stage loop on one screen: logs → dyno → generate candidate → result, with a step guide + per-log quality gate. |
| **Review & Package** | Change report in real units, the knock/forward-safety gate, and a downloadable tuner package + candidate `.bin`. |
| **Diagnose** | Autodiagnose a log; split DTCs into deleted-part side-effects (safely maskable) vs real faults (flagged, never auto-disabled). |
| **Graphs** | The interactive datalog viewer — overlay/compare channels & pulls, knock markers, WOT shading, virtual dyno. |
| **Build & base health** | Base-staleness verdict, build drift vs the base's part fingerprint, and the auto-research review queue. |
| **Maps** | Set/persist the base map and decode any of ~45 ECU tables to real units; knob-evolution charts. |
| **History** | Every stage + attempt (score/verdict/logs/candidate), delete an attempt/log, view the integrity guard + one-click backup. |
| **Assistant** | The car's persistent health record + an optional Q&A helper that answers from *this* car's own knowledge base and writes durable findings back. |
| **Research** | Ask anything → a cited, confidence-labeled report (CONFIRMED / ANECDOTAL / NOT_SETTLED) exportable to PDF. Never feeds a flash gate. |
| **Learn** | Read-only library: cited reference knowledge, prior tuning history, and the learned safe-envelope bounds. |
| **Settings** | Per-car JSON editors with validate-on-save, written through the scoped accessors. |

<details>
<summary><h2>🔬 Decode &amp; domain knowledge</h2></summary>

| Layer | What |
|---|---|
| `decode_table_values.py` | Reads any named ECU table from a 1 MB map as a real-units grid (restricted-AST scaling, no `eval`). |
| `build_table_registry.py` | Decodes **every** 3D table into a machine-readable registry (address, dims, scaling, decoded axes) that gates/diagnosis/assistant consume. |
| `decode_maps.py` | Attributes every changed byte across map revisions to its named table — the main tuning knobs. |
| `subaru_checksum.py` | Reverse-engineered SH7058 checksum (compute/verify/fix) so candidates are checksum-correct. |
| `dimemod_features_state.py` | Walks the def for every DimeMod feature table (ALS / Launch / FFS / MapSwitch / Valet / Flex) + state. |
| `build_master_timeline.py` | A single correlated timeline of the car's history (logs + map revisions), chronologically. |

**`output/knowledge/`** — a cited knowledge base built over the year: the complete setup reference (build, fuel, full sensor/wiring map, how each calibration was discovered, honest open-uncertainties), the reconstructed method, the adversarially-verified WOT/boost discipline, and a symptom → diagnosis → **specific counter** hardware table (spring, BOV, MAP ceiling, pump, AVCS, ethanol).

**AFR-channel model (settled, never re-litigated):** the factory **front** O2 is wideband-type but **hard-clips under boost → untrusted at WOT**; the AEM wideband (rear-O2 tap) is the trusted WOT/boost reference. The pipeline only feeds the unclipped channel to the WOT fueling checks.

</details>

---

## 🚀 Quick start

```bash
# 1. install deps into the project venv
./.venv/Scripts/python.exe -m pip install -r requirements.txt

# 2. launch the tuning UI  (http://localhost:8504)
PYTHONPATH=src python src/tuner/webui/app.py
```

The whole pipeline runs with **no external service**; only the optional in-app Q&A helper needs a model key, and everything else works without it.

**Run the tests** (the project is TDD; both suites stay green before every commit):

```bash
PYTHONPATH=src python tests/run_all.py     #  → 560/560 UI tests
PYTHONPATH=src python verify_pipeline.py   #  →  39/39 decode-pipeline checks
```

---

## 📁 Repository layout

```
make_tune.py  generate_*.py  aggregate_ve.py  knock_margin.py  safety_margin.py
stage_readiness.py  autodiagnose.py  preflight.py  …          # ~60 tuning/analysis modules (top level)
src/tuner/webui/          # the NiceGUI app: app.py · backend.py · integrity.py · cars.py · logviz.py · 12 tabs/
tests/                    # the test suite (run_all.py)   +   verify_pipeline.py
output/knowledge/         # the cited domain-knowledge base (setup reference = source of truth)
output/cars/<id>/         # per-car isolated state: base map, stages, attempts, health, changelog
reference/                # the DimeMod RomRaider def (logger-param source of truth)
```

---

## 🧪 Engineering discipline

- **TDD** (RED → GREEN); both suites green before every commit (`tests/run_all.py` + `verify_pipeline.py`).
- **Fail-closed everywhere** — a missing/undecodable signal refuses, never passes.
- **Owner-review values are sacred** — wastegate spring, boost ceiling, AFR lean limit, target whp. The system *surfaces* them; it never silently overwrites them.
- **Read everything first** — claims about the car or the method are grounded in the full corpus (logs, maps, notes), with real `file:line` citations.

---

*Built and tested on one car as a college senior-design project. It assists a human tuner — it does not replace one.*
