# Flowless

**A closed-loop diagnostic coordination layer that lets specialist diagnostic requests travel with the patient to a convenient collection site, while routing results back to the clinical team responsible for their care.**

👉 **Live prototype: [flowless2.netlify.app](https://flowless2.netlify.app)**

> Hackathon prototype (Tandem Health × NXGN, September 2026). All patient data is synthetic. Not a medical device, not affiliated with or endorsed by the NHS.

---

## The problem

A patient on long-term monitoring — anticoagulation, thyroid, kidney function — is asked to have bloods taken every few weeks, often by a hospital specialist team that may be a long way from where they live. Today the request is bound to the place that wrote it: the patient travels back to the hospital, or the request is re-keyed by a GP surgery or community unit, and the result frequently lands in the wrong inbox or none at all. The paperwork, not the medicine, is what makes monitoring brittle.

## The solution

Flowless separates *who requests* from *where the sample is taken* from *where the result must go*:

1. A specialist creates a **monitoring plan** once (tests, schedule, reasonable adjustments).
2. Each occurrence becomes a portable **request** carrying an **opaque access code** — a QR that contains no clinical or demographic data, only a token.
3. The patient presents the code at **any participating collection unit**. The unit resolves the token, sees the reasonable adjustments first, takes the sample and prints a specimen label with the same token as a barcode.
4. The **laboratory** scans the label, receives the specimen and enters the result.
5. The result is **routed automatically** back to the responsible specialist team, resolved from the requesting site's ODS code. If no route exists, it is flagged as `ROUTING_FAILED` rather than silently lost.
6. Recurring plans schedule the next occurrence as soon as the current result is reviewed.

## Demo flow

**Specialist → Patient → Collection Unit → Laboratory → Responsible Specialist Team**

The header tabs switch between actors. State lives in the browser tab, so run the whole flow without reloading.

1. **Hospital** — start on Aisha Demo-Kaur, an INR plan repeating every 28 days. The access-code panel shows what the QR carries: the code and nothing else.
2. **Issue QR to patient** → **Notify patient** (simulated email/SMS) or **Print patient letter** (the paper route, with the same QR and code in plain type).
3. **Collection unit** — the code is pre-filled or can be typed. Reasonable adjustments are the first thing on the page. **QR presented and scanned** → **Sample collected** → **Print specimen label** (Code 39 barcode, nothing identifying).
4. **Laboratory** — scan/type the code → **Lab receives specimen** → enter a result.
5. Back on **Hospital** the result is awaiting review, addressed to the named clinician at the resolved site. **Review** it and the next occurrence is scheduled.
6. **Failure case** — open Tom Demo-Whyte. His sample was taken at a site absent from the results directory, so the result has nowhere to go and is held for a human to route.
7. **Patient** — NHS App concept mockup (clearly labelled as a concept).

**Reset demo** in the header restores the seeded scenarios at any point.

## What is real and what is simulated

| | Status |
| --- | --- |
| Workflow state machine, guards, role enforcement | **Real.** Pure functions, unit tested. |
| QR codes | **Real.** Hand-written ISO/IEC 18004 encoder; no CDN, no network. |
| Specimen barcode | **Real** Code 39 — scan the screen with a barcode app. |
| SNOMED CT codes | **Real** concept ids for the six test panels (procedure codes for requests, observable codes for results); see `js/data/testPanels.js` for sources and caveats. |
| ODS trust code `RRK` | **Real** (University Hospitals Birmingham NHS FT). Site and ward codes are structured like the real thing, names illustrative. |
| Result routing | **Real logic** against a directory — but the directory is three rows of demo data. |
| Cross-device phone scanning | **No.** Single laptop, hash-routed screens. |
| Email / SMS delivery | **Simulated.** Sending writes a history note; no provider is contacted. |
| NHS App | **Concept mockup only.** No API, no affiliation. |
| Backend, database, authentication, multi-device sync | **None.** In-memory, single tab. |

## Run locally

No build step, no install, no network.

```bash
git clone https://github.com/pkaysantana/flowless.git
cd flowless
python3 -m http.server 8080      # then open http://localhost:8080
```

Or simply open `index.html` in a browser — the app uses plain `<script>` tags rather than ES modules precisely so `file://` works.

## Tests

```bash
node tests/run.js                # 31 tests, no framework, no install
```

Covers the transition table and guards (including that `requiresHuman` steps reject the `system` actor and that each step is pinned to one role), recurrence, routing resolution and failure, and a privacy property asserting across 300 tokens that no patient identifier leaks into an access code.

## Lifecycle

```
REQUEST_CREATED → QR_ISSUED → PRESENTED_AT_COLLECTION → SAMPLE_COLLECTED
  → LAB_PROCESSING → RESULT_AVAILABLE
      ├─ routed   → AWAITING_CLINICIAN_REVIEW → REVIEWED   (terminal)
      └─ no route → ROUTING_FAILED ⇄ AWAITING_CLINICIAN_REVIEW
  any non-terminal state → CANCELLED   (terminal)
```

`js/domain/workflow.js` holds this as a transition table. `transition()` is a pure function and the only thing that grants permission; `store.applyStep()` is the only thing that writes state, and it asks first — so a guard cannot be bypassed by adding a screen. Routing is computed from the ODS lookup, never chosen by a caller. A missing value is never defaulted: it renders as *Not recorded*.

## Project layout

```
index.html              The shell. Script order matters: lib → domain → data → store → ui.
css/app.css             All styles, including print styles for the letter and label.
js/lib/                 QR encoder and Code 39 barcode. No dependencies.
js/domain/              Pure logic, no DOM: types, workflow, tokens, recurrence, routing.
js/data/                Editable demo content: SNOMED picklist, ODS directory, seeded plans.
js/store/               In-memory store — the only file a real backend would replace.
js/ui/                  One file per screen: console, new request, collection unit, lab,
                        NHS App mockup, print views, hash router.
tests/run.js            Test runner.
netlify.toml            Static deploy of the repository root.
```

## Team

Built at the Tandem Health × NXGN hackathon by [Daniel Emelike](https://github.com/DanielEmelike) and [pkaysantana](https://github.com/pkaysantana), with the wider Flowless team.

## History

Earlier explorations from the hackathon (a React/Vite referral co-pilot and a first "Care Relay" implementation) are preserved as tags: [`archive/react-vite-hackathon`](https://github.com/pkaysantana/flowless/tree/archive/react-vite-hackathon) and [`hackathon-final-2026-09-05`](https://github.com/pkaysantana/flowless/tree/hackathon-final-2026-09-05) (the exact state deployed at submission).
