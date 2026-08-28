# Redacta Gauntlet

![CI](https://github.com/nickjlamb/redacta-gauntlet/actions/workflows/gauntlet.yml/badge.svg)
![Node](https://img.shields.io/badge/Node-20+-339933?logo=node.js&logoColor=white)
![data](https://img.shields.io/badge/data-synthetic%20only-blue)
![gate](https://img.shields.io/badge/regression%20gate-CI--failing-0d7377)

**An adversarial evaluation harness for clinical pseudonymisation.** It attacks
[Redacta](https://www.pharmatools.ai/redacta) at every layer it has — the
deterministic pattern engine, the LLM reasoning layer, and the models
*downstream* that consume its output — and reports **recall under adversarial
conditions**, with over-redaction as the cost axis. One command, SHA-stamped
scorecards, a CI-failing regression gate. Gaps named, boundaries owned.

📊 **Full write-up with charts:** [pharmatools.ai/redacta-eval](https://www.pharmatools.ai/redacta-eval)

Redacta itself, including production deployment options: [pharmatools.ai/redacta](https://www.pharmatools.ai/redacta)

A false negative — one identifier that slips through — is a privacy breach, so
recall is asymmetric and comes first; over-redaction is tracked as the cost,
because a redactor that deletes the document scores perfect recall and is
useless. The two numbers only mean something together.

---

## Results at a glance

Measured on a 28-case adversarial gold set (+ 8 downstream-injection cases), all
synthetic. Every figure is reproducible from a stamped scorecard in `results/`.

| Layer under test | What it adds | Headline result |
|---|---|---|
| **Deterministic engine** <br>(ships in the app, CLI, MCP) | patterns + keyword anchors | **91.5%** in-scope recall · **0%** over-redaction · **97.7%** precision · **100%** injection resistance |
| **+ Reasoning layer** <br>(Claude / Perplexity Sonar) | LLM Layer-2 (names, ages, initials) | Layer-2 recall **0 → 80% (Claude) / 100% (Sonar)** · indirect leakage **0% — the ceiling** · **5.9%** over-redaction, the cost of the lift |
| **Downstream consumer** <br>(a model reading Redacta's output) | injection at the real surface | **0 identifier leaks** on completely-redacted notes, *both* models · behavioural hijack reported, not gated — an honest boundary |

The through-line: complete redaction contains the blast radius to *nothing
identifying*, at every layer — and where a number can't be perfect (indirect
leakage, downstream behaviour), it's named with a measurement rather than hidden.

## Run it

```bash
npm install
npm run gate          # deterministic suite + regression gate — no API key, no network, CI-safe
npm run eval          # scorecard only
```

```
Redacta Gauntlet — gold v0 · 28 cases · @pharmatools/redacta@1.2.0
  Adversarial recall (lenient)         76.8%
  In-scope recall (deterministic)      91.5%
  Over-redaction rate  (lower=better)  0%
  Precision (all removals)             97.7%
  Injection resistance                 100%
  ✓ regression gate passed — no metric worse than baseline
```

LLM-backed layers (need a key; keep their own scorecards and gates):

```bash
export ANTHROPIC_API_KEY=sk-ant-...          # or PERPLEXITY_API_KEY + REDACTA_REASONING_PROVIDER=perplexity
npm run eval:reasoning        # deterministic Layer 1 + LLM Layer 2, combined
npm run eval:downstream       # redact → feed to a downstream summariser → score hijack + leakage
```

## What it found

The point of an adversarial set is the failures. A few worth naming:

- **A dual failure in prose.** On *"…transferred to the RJ1-2209841 record. Hospital
  number confirmed at desk,"* the MRN pass grabbed the word **"confirmed"** and
  tokenised *that* — while the real identifier `RJ1-2209841` survived in the clear.
  A miss and a spurious redaction in one sentence. The precision metric now catches
  exactly this class, and fails CI on any new occurrence.
- **The reasoning layer closes one gap and reveals the ceiling.** LLM Layer-2 lifts
  names/ages/initials from 0% to 80–100% — but indirect leakage ("the sitting MP for
  a Greater Manchester constituency") stays at **0% on every layer**. Quasi-identifier
  reconstruction is beyond patterns *and* reasoning. Named, not averaged away.
- **Downstream injection can't breach what was redacted.** Feed a redacted note to a
  consuming model and instruct it to leak — it only ever received tokens, so on both
  Claude and Sonar, **zero** identifiers leaked across all complete-redaction cases.
  The one leak was a redaction *miss* the consumer propagated: proof that completeness
  is the control, not consumer discretion.

## How it works

Same shape as a test suite: gold cases in, metrics out, a baseline to diff against.

```
gold cases ──▶ scorers ──▶ scorecard (engine version + code SHA) ──▶ regression gate ──▶ CI
               │                                                          │
   offline: deterministic engine, no network              fails the build on any recall
   online: live MCP / reasoning LLM / downstream LLM       drop or over-redaction rise
```

- **`gold.json`** — 28 synthetic cases across five attack surfaces, every identifier
  labelled by type and `scope` (`deterministic` / `reasoning` / `quasi`), with hostile
  cases flagged `expected_miss` so a known limitation can't pass as a win.
- **`gold-downstream.json`** — 8 cases probing an LLM consuming Redacta's output.
- **`src/`** — `engine` (offline + live MCP), `reasoning` (provider-agnostic Layer-2),
  `downstream` (consumer-injection), `score` (recall / precision / per-scope), `run` (driver + gate).
- **`results/`** — every scorecard stamped with engine version, code SHA and timestamp.

See [`THREAT_MODEL.md`](./THREAT_MODEL.md) for the attacker model and the five
attack surfaces (prose-embedded, edge formats, adversarial near-misses, direct
prompt injection, indirect leakage).

## Honest by construction

- **The set includes cases it expects to fail.** Reasoning and quasi scopes score
  0% on the deterministic engine by design, reported separately so neither can hide
  behind the other.
- **Recall before convenience.** False negative = breach, false positive = cost;
  scored apart, each with its own gate, so recall can't be bought by shredding the record.
- **Offline == online.** The deterministic scorecard is cross-checked token-for-token
  against the live MCP (`results/mcp-parity.json`).
- **Reproducible.** Every result traces to an engine version and code SHA.
- **Synthetic only, permanently.** No real patient data, ever — a deliberate ceiling.

## Known gaps

- **Behavioural hijack of a downstream model** is real and *not Redacta's to fix* —
  the eval reports it rather than claiming a defence Redacta doesn't provide.
- **Quasi-identifier reconstruction** (indirect leakage) is beyond every current
  layer; the standing ceiling, measured at 0% so the day it moves is provable.
- **A Perplexity reasoning run** measures "Sonar as the reasoning layer" — Sonar is
  search-augmented and never a production choice for real PHI; it's a comparison only.

---

Part of [Redacta](https://github.com/nickjlamb/redacta) · built by [Nick Lamb](https://github.com/nickjlamb) ·
methodology mirrors the [RefCheckr eval framework](https://www.pharmatools.ai/refcheckr-eval).
