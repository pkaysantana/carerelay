# Flowless

**A closed-loop diagnostic coordination layer that lets specialist diagnostic requests travel with the patient to a convenient collection site, while routing results back to the clinical team responsible for their care.**

👉 **Live prototype: [flowless2.netlify.app](https://flowless2.netlify.app)**

> Hackathon prototype (Tandem Health × NXGN, September 2026) using synthetic data. Not for clinical use. Not affiliated with or endorsed by the NHS.

Digital ordering, result routing and disease-specific monitoring already exist in the NHS. For how this prototype relates to them, see [Where Flowless fits](#where-flowless-fits).

---

## The problem

A patient on long-term monitoring — anticoagulation, thyroid, kidney function — is asked to have bloods taken every few weeks, often by a hospital specialist team that may be a long way from where they live. Where the requesting team and the collection site do not share an order-communications system, the request tends to stay bound to the place that wrote it: the patient travels back to the hospital, or the request is re-keyed by a GP surgery or community unit, and the result can land with a clinician who did not order it, or with no one. How often this happens, and in which pathways, is something we have not measured (see [What remains to validate](#what-remains-to-validate)).

## The solution

Flowless separates *who requests* from *where the sample is taken* from *where the result must go*:

1. A specialist creates a **monitoring plan** once (tests, schedule, reasonable adjustments).
2. Each occurrence becomes a portable **request** carrying an **opaque access code** — a QR that contains no clinical or demographic data, only a token.
3. The patient presents the code at **any participating collection unit**. The unit resolves the token, sees the reasonable adjustments first, takes the sample and prints a specimen label with the same token as a barcode.
4. The **laboratory** scans the label, receives the specimen and enters the result.
5. The result is **routed automatically** back to the responsible specialist team. In the prototype, routing is resolved from the requesting site's ODS code and requester metadata; a production design would anchor operational ownership to the responsible clinical service or team, keeping the individual requester for provenance. If no route exists, the result is flagged as `ROUTING_FAILED` rather than silently lost.
6. Recurring plans schedule the next occurrence as soon as the current result is reviewed.

## Demo flow

**Specialist → Patient → Collection Unit → Laboratory → Responsible Specialist Team**

The header tabs switch between actors. State lives in the browser tab, so run the whole flow without reloading.

1. **Hospital** — start on Aisha Demo-Kaur, an INR plan repeating every 28 days. The access-code panel shows what the QR carries: the code and nothing else.
2. **Issue QR to patient** → **Notify patient** (simulated email/SMS) or **Print patient letter** (the paper route, with the same QR and code in plain type).
3. **Collection unit** — the code is pre-filled or can be typed. Reasonable adjustments are the first thing on the page. **QR presented and scanned** → **Sample collected** → **Print specimen label** (Code 39 barcode, nothing identifying).
4. **Laboratory** — scan/type the code → **Lab receives specimen** → enter a result.
5. Back on **Hospital** the result is awaiting review, addressed to the named clinician at the resolved site. **Review** it and the next occurrence is scheduled.
6. **Failure case** — open Tom Demo-Whyte. His request came from a site absent from the results directory, so the result has nowhere to go and is held for a human to route.
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
| Integration with any order-communications system, LIMS, EPR or GP system | **None.** No HL7, FHIR or MESH messaging. |

## Where Flowless fits

Flowless is not intended to replace:

- pathology order-communications systems;
- laboratory information systems;
- electronic patient records;
- disease-specific monitoring tools.

Existing products already cover substantial parts of the workflow at meaningful NHS scale: electronic requesting, specimen labelling at collection, returning the result to the requesting clinician and location, sign-off of results, and recall of patients overdue for monitoring. The [incumbent landscape](#incumbent-landscape) below sets out what we could verify.

The narrower question this prototype explores is what happens to *responsibility for a result* when a specialist-authored request is fulfilled outside the requester's own systems, so that requester, patient, collection site, laboratory and responsible clinical service do not all sit inside one order-communications deployment.

Professional guidance is clear on the default. The BMA states that ["the responsibility for ensuring that results are acted upon rests with the person requesting the test"](https://www.bma.org.uk/advice-and-support/gp-practices/communication-with-patients/duty-of-care-when-test-results-and-drugs-are-ordered-by-secondary-care), and that it passes to someone else only by prior agreement. It also advises [caution about systems that automatically send results to clinicians who did not order the test](https://www.bma.org.uk/advice-and-support/nhs-delivery-and-workforce/primary-and-secondary-care/acting-upon-electronic-test-results). This is a default rather than an absolute rule: shared-care agreements and emergency departments are recognised exceptions, and local protocols vary.

The prototype's design concern is therefore accountability, not only moving data:

```
request created
  → request fulfilled elsewhere
  → specimen processed
  → result generated
  → responsible clinical service receives it
       └─ if that cannot be completed
            → ROUTING_FAILED
            → visible, and a human has to resolve it
            → never silently dropped
```

Two limits on what this demonstrates:

- The prototype addresses a result to the requesting clinician and site. That is broadly similar to the requester/location addressing model documented for ICE, not a new one.
- It shows the behaviour working in a browser. It does not show that the behaviour is missing from, or hard to add to, systems already deployed.

## Incumbent landscape

Figures are the vendors' own. The last column lists questions, not claims.

| System | Primary scope | Relevant existing capability | Overlap with Flowless | Open question for Flowless |
| --- | --- | --- | --- | --- |
| **[Clinisys ICE](https://www.clinisys.com/uk/en/products/clinisys-ice-en/)** | Order communications and results reporting for pathology, radiology and other diagnostics, across primary, secondary and community care. Over 140 UK implementations and 93 million requests a year. | Multi-Org lets several organisations share one ICE. OpenNet and Gateway let users view results and place orders across separate ICE systems. Specimen labels are generated at collection. Requests can be postponed and collected later. Results return to the requesting clinician and location, with a Results Acknowledgement sign-off, an unfiled-reports view and an audit trail. | High. A request collected somewhere else, a label printed at collection, a result routed to the requester and a sign-off step all exist inside a connected ICE estate. | How often does a specialist request need fulfilling outside the requester's ICE estate (another region, another supplier), and what happens to the result when it does? |
| **[ICE Portal](https://www.clinisys.com/uk/en/news/black-country-pathology-services-and-clinisys-use-ice-portal-to-link-non-connected-care/)** | Web ordering and results for settings with no order-communications access. Built with Black Country Pathology Services for COVID-19 testing in care homes; Clinisys also names prisons and schools. | Barcoded request forms matched at the laboratory. Results return to the portal for authorised users. | High for the idea that a non-connected setting can take part in electronic requesting. | The Portal's user is the non-connected setting acting as requester. We could not establish from public material whether it also serves a site that only collects on behalf of a distant specialist. |
| **[INRstar](https://invitaintelligence.com/inrstar)** | Anticoagulation dosing and decision support (inVita intelligence). 2,200 NHS services and over 155,000 active patients, across GP practices, hospital clinics, community pharmacies and self-testing patients. | Integrates with GP systems, LIMS and EHRs. A patient app supports self-testing. [External Patient Lookup](https://help.inrstar.co.uk/Article/10fc3b4e-de1c-4e7b-bcab-7fb38d0af62f) gives access to patients managed by another service. | High. The demo's headline scenario, a recurring INR, is already served, often by point-of-care testing with no laboratory round trip. | INR monitoring is probably a weak starting point. We chose it as a familiar recurring test, not because the pathway is unserved. |
| **[DAWN](https://www.4s-dawn.com/high-risk-medication-monitoring/)** (4S) | Monitoring of patients on high-risk medication (DMARDs, biologics, anticoagulants), used mainly by specialist teams. Over 200 organisations worldwide. | Acquires pathology results through interfaces. Flags abnormal results. Work lists and SMS, email or letter recall for overdue bloods. [Shared-care communication with GPs](https://www.4s-dawn.com/rheumatology/). | High for the recurring plan, overdue detection and specialist ownership of the result. The second seeded demo case, baseline bloods before methotrexate, is a DAWN use case. | When the blood is taken outside the specialist trust's laboratory catchment, does the result still reach DAWN automatically? We do not know. |
| **EPR / GP-system result-routing controls — illustrative example** | Results inboxes in hospital EPRs and GP systems. | Routing a result to the ordering clinician is widely described as standard behaviour. We did not conduct a systematic survey of Epic, Oracle Health or other major EPRs. As one example, not evidence for every EPR, a single trust's [procedure](https://www.cpft.nhs.uk/download/cl93-management-of-pathology-results.pdf?ver=19557) for SystmOne shows results being re-assigned by hand when they arrive unassigned or against the wrong clinician, and escalated if left unfiled. | High. Requiring a human to clear a result that has no owner is existing governance practice. An explicit failure state is not new. | Do these controls hold when a result crosses an organisational boundary? The same procedure notes that results from out-of-county laboratories are processed manually. |
| **Flowless prototype** | Browser-only demonstration of request → collection → laboratory → routed result. No backend and no integrations. | An opaque token on the request. Routing computed from the requesting site and clinician. An explicit `ROUTING_FAILED` state. A unit-tested state machine. | — | Is any of this needed as a separate layer, rather than as configuration or extension of the systems above? |

One distinction may be worth testing. The incumbent controls we could verify deal with results that arrived and were not yet acknowledged. `ROUTING_FAILED` models a result whose destination could not be resolved at all. We found no public documentation of how deployed systems handle that second case, which is not evidence that they handle it badly.

## What remains to validate

The hackathon prototype does not establish commercial differentiation. Before treating Flowless as a company, these questions would need answers from people who run these services:

- Where do current ICE and EPR deployments actually fail across organisational boundaries, and how often?
- Which pathways still depend on GP re-keying or manual forwarding of hospital requests?
- Is ownership of a result by a clinical service, rather than a named individual, inadequately represented today? The prototype does not represent it either.
- Is a result whose requester cannot be resolved already handled adequately by laboratories and incumbent systems?
- Who would buy or deploy a coordination layer instead of extending an existing order-communications platform?
- Which standards and APIs (HL7 v2, FHIR, MESH) would make Flowless complementary to those platforms instead of duplicating them?
- Is the highest-value starting point pathology, recurring medicines monitoring, community diagnostics, or something else?

## Post-hackathon feedback

Tandem Health's written feedback credited three things: the explicit routing-failure state, keeping ownership of the result with the requesting team, and a demo that separates real from simulated functionality. It also challenged us to address incumbent products, naming Clinisys ICE and ICE Portal, INRstar, DAWN and the result routing built into every major record system. The three sections above are our response.

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

## Sources / further reading

Checked September 2026. Vendor pages describe what products are designed to do, not how any particular deployment is configured.

- Clinisys — [ICE product page](https://www.clinisys.com/uk/en/products/clinisys-ice-en/), [ICE brochure (PDF)](https://www.clinisys.com/app/uploads/2021/03/ICE_Brochure.pdf), [ICE Portal with Black Country Pathology Services](https://www.clinisys.com/uk/en/news/black-country-pathology-services-and-clinisys-use-ice-portal-to-link-non-connected-care/)
- HTN — [Clinisys on order communications in integrated care systems](https://htn.co.uk/2023/02/16/feature-darren-ransley-of-clinisys-on-order-communications-and-results-reporting-in-the-digital-ics/)
- The Pathology Centre — [ICE order communications training manual (PDF)](https://www.thepathologycentre.org/wp-content/uploads/2018/02/ICE-Pathology-Training-Manual.pdf)
- inVita intelligence — [INRstar](https://invitaintelligence.com/inrstar), [External Patient Lookup](https://help.inrstar.co.uk/Article/10fc3b4e-de1c-4e7b-bcab-7fb38d0af62f)
- 4S DAWN — [High-risk medication monitoring](https://www.4s-dawn.com/high-risk-medication-monitoring/), [Rheumatology](https://www.4s-dawn.com/rheumatology/)
- BMA — [Duty of care when test results and drugs are ordered by secondary care](https://www.bma.org.uk/advice-and-support/gp-practices/communication-with-patients/duty-of-care-when-test-results-and-drugs-are-ordered-by-secondary-care), [Acting upon electronic test results](https://www.bma.org.uk/advice-and-support/nhs-delivery-and-workforce/primary-and-secondary-care/acting-upon-electronic-test-results)
- NHS England — [Clinical messaging and test results](https://www.england.nhs.uk/long-read/clinical-messaging-and-test-results/)
- Cambridgeshire and Peterborough NHS Foundation Trust — [Management of Pathology Results SOP (PDF)](https://www.cpft.nhs.uk/download/cl93-management-of-pathology-results.pdf?ver=19557)

## Team

Built at the Tandem Health × NXGN hackathon by [Daniel Emelike](https://github.com/DanielEmelike) and [pkaysantana](https://github.com/pkaysantana), with the wider Flowless team.

## History

Earlier explorations from the hackathon (a React/Vite referral co-pilot and a first "Care Relay" implementation) are preserved as tags: [`archive/react-vite-hackathon`](https://github.com/pkaysantana/flowless/tree/archive/react-vite-hackathon) and [`hackathon-final-2026-09-05`](https://github.com/pkaysantana/flowless/tree/hackathon-final-2026-09-05) (the exact state deployed at submission).
