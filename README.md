# 🏁 STI Tuning Intelligence

> A safety-gated, **speed-density tuning pipeline** I built for my own 2018 WRX STI — it reads the car's datalogs, reproduces a proven VE-first tuning method to generate ROM candidates, and refuses to let an unsafe map reach the engine. College senior-design project; one car only.

![tests](https://img.shields.io/badge/tests-558%20passing-brightgreen)
![pipeline](https://img.shields.io/badge/decode%20pipeline-39%2F39-brightgreen)
![python](https://img.shields.io/badge/python-3.13-blue)
![ui](https://img.shields.io/badge/UI-NiceGUI-ff453a)
![safety](https://img.shields.io/badge/gates-fail--closed-success)

**What it is, plainly:** a **deterministic pipeline** — *not* an "AI tunes your car" gimmick. It decodes the actual ECU tables to real units, applies the math of a proven speed-density (VE-first) method, and gates every candidate it generates behind hard safety checks. A human and a wideband still do the real tuning — this assists, diagnoses, and blocks unsafe maps.

> ⚠️ **Personal senior-design project, calibrated to ONE specific built 2018 STI** (big turbo, E85). Not a product, not a tuner replacement, not a dyno. Every threshold was learned from this one car's own data and is tailored to it.

---

## Where it's at (honest status)

- **Stage 1 (base fueling / VE) — in progress, going well.** Across attempts the VE corrections it wants are *shrinking* (median ≈ 5% → ≈ 2%) — the air model is converging on target. Idle sits steady at stoich with a tiny closed-loop trim; cruise fueling reads dialed (0 cells off target).
- **No WOT pulls yet — by design.** The method is a ladder: **idle → cruise → ~50% pull → full pull → only then add boost.** It won't green-light a pull until idle + cruise are confirmed clean.

## Built on a year of real data

It isn't guessing from defaults. The whole thing is grounded in **~a year of actually tuning this car** — every map revision and datalog, plus a vetted reference library I built up covering the build, the parts, the fuel, and the tuning concepts behind each decision. The pipeline learns the **method** and the **safe envelope** directly from that history, so its corrections and its limits come from what this car has actually done, not from a generic table.

## The method it reproduces

**VE first → hold timing → build boost incrementally → never stack boost + timing in one map.** A few maps perfect a stage; a hardware limit gets a *hardware* prescription (e.g. it reads wastegate-duty headroom and will tell you "the spring's too soft, go stiffer" instead of chasing it in software).

## What it actually does

- **Decodes the ROM** — ~45 ECU tables to real units (VE, target boost, timing, open-loop AFR, idle/overrun), checksum-correct.
- **Generates VE candidates** — per-cell corrections from the wideband + closed-loop trims, clamped to a safe envelope learned from the car's *own* clean map history → a flashable `.bin`.
- **Hard safety gates (fail-closed)** — every candidate must pass: ROM identity, per-cell envelope, no boost×timing stacking, a hard boost ceiling, a knock-zone check, an absolute rich-AFR floor under boost, VE-lean-under-boost, and a pre-flash GO/NO-GO. **If a signal is missing or undecodable it REFUSES — it never passes on doubt.**
- **Per-attempt readiness + scoring** — tells you **ADVANCE / KEEP TUNING / BLOCKED**, scores each candidate 1–10 by how *converged* the fueling actually is, and folds everything into one "do this next" directive.
- **Reads the car's health** — flags rough idle, misfires (per-cylinder roughness), fuel-pressure-vs-boost, ethanol dropout, AVCS, knock — and decides whether each is a **tune** fix or a **hardware** fix.
- **Interactive log graphs** — overlay channels/pulls on a normalized single axis, smoothing, hover crosshair, knock markers, WOT-region shading, and a virtual dyno.

## Adjusted for the car's health (a few examples)

- **Won't allow WOT while the ethanol sensor reads 0** — a known sensor dropout means the ECU thinks it's on pump gas, which would run dangerously lean on E85.
- **Trusts the right wideband under boost** — the factory front O2 saturates rich and lies at WOT, so it's ignored there; only the unclipped wideband feeds the WOT fueling check.
- **Boost stays BLOCKED until the VE is re-validated** for the current cam/hardware — you don't add power on an unverified air model.

## Engineering

- **TDD throughout** — **558 UI tests + 39 decode-pipeline checks**, both green before every commit.
- **Fail-closed everywhere** — a missing/undecodable signal refuses, never passes.
- **Safety limits are surfaced, never silently changed** — boost ceiling, rich-AFR floor, wastegate-spring window, target whp are flagged for human review.
- **Leak-free per-car data model** with a centralized integrity guard so wrong data can't reach a flash.

## Stack

Python 3.13 · NiceGUI web UI · pandas / numpy / matplotlib / Plotly · a from-scratch SH7058 checksum + ROM table decoder.

---

*Built and tested on one car as a college senior-design project. It assists a human tuner — it does not replace one. Don't flash anything off it without a wideband, a dyno when it matters, and your own judgment.*
