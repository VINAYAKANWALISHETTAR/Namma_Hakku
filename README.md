# ನಮ್ಮ ಹಕ್ಕು · NammaHakku

**Speak your problem. Know your rights. Take action.**
ನಿಮ್ಮ ಸಮಸ್ಯೆ ಹೇಳಿ, ನಿಮ್ಮ ಹಕ್ಕು ತಿಳಿದುಕೊಳ್ಳಿ, ಕ್ರಮ ಕೈಗೊಳ್ಳಿ

NammaHakku ("our right" in Kannada) is a voice-first, agentic AI product that turns a spoken
grievance in **Kannada, Hindi or English** into a **validated RTI application** addressed to the
correct authority, with every legal citation traced back to retrieved source text.

It is built for the citizen who knows *what went wrong* but not *who to write to*, *under which
law*, or *how to phrase it in formal English*.

---

## Table of contents

- [The problem](#the-problem)
- [What it does](#what-it-does)
- [Live demo](#live-demo)
- [Architecture](#architecture)
  - [Agent pipeline](#agent-pipeline)
  - [Request lifecycle](#request-lifecycle)
  - [Decision gates, confidence and escalation](#decision-gates-confidence-and-escalation)
  - [Grounded retrieval (hybrid RAG)](#grounded-retrieval-hybrid-rag)
  - [Validation and the export gate](#validation-and-the-export-gate)
  - [Time-triggered escalation](#time-triggered-escalation)
  - [Data model](#data-model)
  - [API surface](#api-surface)
- [Tech stack](#tech-stack)
- [Security by design](#security-by-design)
- [Repository contents](#repository-contents)
- [Local development](#local-development)
- [Roadmap](#roadmap)
- [Limitations and ethics](#limitations-and-ethics)
- [Creator](#creator)
- [License](#license)

---

## The problem

A stopped pension. A ration card stuck for 40 days. A street nobody cleans. The hard part is never
the complaint, it is the **process**, the **authority** and the **legal wording**.

| Gap | What the citizen experiences |
| --- | --- |
| **No procedural knowledge** | Knows the problem but cannot write a formal RTI application; phrasing, structure and legal language are unfamiliar. |
| **Hard to find the right authority** | Identifying the department, the Public Information Officer (PIO) and the correct jurisdiction requires insider knowledge. |
| **Language barrier** | Can explain the problem aloud in their own language, cannot write it in formal English. |
| **Risk of wrong AI content** | Generic AI tools invent section numbers, deadlines and addressees, which makes the generated document legally unreliable. |

> *"Lakshmi, 68, Hubballi. Her pension stopped three months ago. She does not know who to write to,
> which law applies, or how to write it in English. In the end she gives up. NammaHakku is built so
> that doesn't happen."*

---

## What it does

1. The citizen **speaks** the problem in Kannada, Hindi or English.
2. The system **transcribes** and **classifies** it into a grievance category.
3. It **resolves** the correct authority / PIO from a curated jurisdiction directory.
4. It **retrieves** the governing legal context (RTI Act, pension rules, PDS rules).
5. It **drafts** an RTI application or grievance letter grounded only in that retrieved context.
6. It **validates** the draft against a citation guard and a completeness checklist, and only then
   allows export.

Output is a **sample document, not legal advice**.

---

## Live demo

This repository contains the product landing page, which includes a **fully interactive front-end
simulation** of the pipeline — the same six agents, the same thresholds and the same checklist as
the real design.

Open [`index.html`](index.html) in a browser, or serve it locally:

```bash
# any static server works
python -m http.server 8000
# then open http://localhost:8000
```

The demo lets you:

- pick an example grievance (Kannada/Hindi),
- type your own problem,
- press the mic (Chrome, `kn-IN`),
- watch all six agents run with their status,
- untick safety-checklist items to watch `C(d)` drop below the `0.85` export threshold,
- generate, print and download the RTI letter.

---

## Architecture

NammaHakku is deliberately **not one big model**. It is six narrow agents, each with a clear
trigger, a decision, a confidence threshold, a retry budget and an escalation path.

```
 ┌──────────┐   audio / text   ┌──────────────┐
 │ Citizen  │ ───────────────► │  A1 Transcribe│  Bhashini ASR (Whisper-large-v3 fallback)
 └──────────┘                  └──────┬───────┘
                                       │ typed text + detected language
                                       ▼
                              ┌────────────────┐   low SNR / ambiguity
                              │ A2  Classify   │ ─────────────────────► re-prompt citizen
                              └────────┬───────┘
                                       │ category + confidence
                                       ▼
                              ┌────────────────┐   Ci < 0.55
                              │ A3   Resolve   │ ─────────────────────► escalate to human
                              └────────┬───────┘   0.55 ≤ Ci < 0.80 → ask citizen to confirm
                                       │ department / PIO / jurisdiction
                                       ▼
                              ┌────────────────┐
                              │ A4  Retrieve   │  dense (embeddings) + sparse (BM25) + RRF
                              └────────┬───────┘
                                       │ top-k legal chunks (grounding set)
                                       ▼
                              ┌────────────────┐   confidence < 0.85
                              │ A5    Draft    │ ─────────────────────► refuse + re-prompt
                              └────────┬───────┘
                                       │ grounded RTI application
                                       ▼
                              ┌────────────────┐   C(d) < 0.85
                              │ A6  Validate   │ ─────────────────────► retry ×2 → template
                              └────────┬───────┘   C(d) ≥ 0.85 → allow export
                                       │
                                       ▼
                            RTI letter (PDF / .txt) + SLA clock started
```

### Agent pipeline

| Agent | Name | Trigger | Decision | Output |
| --- | --- | --- | --- | --- |
| **A1** | Transcribe | Audio or text input | Accept, re-prompt, or fall back to typing | Typed text + language tag |
| **A2** | Classify | Text present | Map to grievance category or ask again | `Category = Pension`, confidence |
| **A3** | Resolve | Category known | Route to department / PIO | Authority + jurisdiction |
| **A4** | Retrieve | Authority known | Fuse dense + sparse rankings | Grounded legal chunks |
| **A5** | Draft | Grounding sufficient | Write or refuse to draft | Draft RTI / grievance letter |
| **A6** | Validate | Draft exists | Score, retry, or block export | `C(d)` score + verdict |

### Request lifecycle

1. **Ingest** — audio (Kannada/Hindi/English) or typed text arrives; a `request_id` is created and
   tied to a 1-hour access token.
2. **Transcribe** — speech-to-text with Bhashini, Whisper-large-v3 as fallback. Low signal-to-noise
   or an unintelligible result returns a re-prompt instead of guessing.
3. **Classify** — the text is mapped to one of the supported grievance categories (Phase 1: pension
   disbursal, PDS/ration cards, municipal complaints).
4. **Resolve** — a jurisdiction lookup finds the department and the Public Information Officer.
   Authority confidence `Ci` decides the routing behaviour (see below).
5. **Retrieve** — hybrid retrieval over the legal corpus returns the grounding set. Nothing is
   drafted from model memory.
6. **Draft** — the drafting agent writes the RTI application using *only* retrieved context, with
   citations attached to each claim.
7. **Validate** — citation guard + completeness checklist produce `C(d)`. Retry, template-only
   draft, or manual review.
8. **Export & monitor** — the citizen downloads the letter, PII is scrubbed, and a 30-day statutory
   clock is started for the follow-up.

### Decision gates, confidence and escalation

The system decides **when to act, ask, retry and escalate** using explicit thresholds:

| Behaviour | Rule | Example |
| --- | --- | --- |
| **Autonomous routing** | `Ci ≥ 0.80` → auto-route to the PIO | `Ci = 0.91` goes straight to the Pension PIO |
| **Confirm with citizen** | `0.55 ≤ Ci < 0.80` → show top candidates, ask | "Is this the Social Welfare Department?" |
| **Escalate** | `Ci < 0.55` → no draft, escalate | "Please say that again." |
| **Self-gating on ASR** | Shaky transcription → refuse to draft | "Please say that again." |
| **Self-correction loop** | Failed validation → retry ×2 with a stricter prompt, then template-only draft or manual review | Missing PIO address → redraft → passes |

### Grounded retrieval (hybrid RAG)

Law comes from retrieval, not from memory. Three stages:

| Stage | Method | Why |
| --- | --- | --- |
| **Dense** | Cosine similarity, `text-embedding-3-large` (self-hosted `bge-m3` fallback), `top-k = 8` | Matches *meaning* — "pension stopped" also retrieves "pension disbursement" sections |
| **Sparse** | Okapi BM25 (`k1 = 1.5`, `b = 0.75`) | Preserves *exact* legal terms, section numbers, department and PIO names |
| **Fusion** | Reciprocal Rank Fusion (`K = 60`) | Combines two incomparable score scales; a chunk ranked high by both rises to the top |

The final **top-4 chunks** become the grounding set handed to the drafting engine. If a required
citation is not present in that set, the system refuses or regenerates rather than inventing it.

### Validation and the export gate

Two independent checks run before anything leaves the app:

**Citation Guard** — every statute, section and deadline in the draft must appear in the retrieved
context. Unsupported claims are stripped and the draft is regenerated.

**Completeness checklist `C(d)`**

| Item | Weight |
| --- | --- |
| Correct addressee (PIO / department) | 0.25 |
| Statutory citation present | 0.20 |
| Applicant identity block complete | 0.15 |
| Clear, specific information request | 0.20 |
| Fee / exemption clause where applicable | 0.10 |
| Date and place fields present | 0.10 |

**Export is allowed only at `C(d) ≥ 0.85`.** Below that the draft is retried, replaced by a
template-only draft, or sent to manual review. PII scrubbing runs again at export time.

### Time-triggered escalation

An SLA cron job acts **without a new prompt**. When the 30-day statutory deadline under
RTI Act Section 7(1) is breached with no reply, the system auto-drafts the **First Appeal** to the
next authority. Day 31, no reply → the appeal is already ready.

### Data model

```
citizens        (id, phone_hash, name, locale, created_at)
requests        (id, citizen_id, category, language, audio_uri, text, ci, created_at)
authorities     (id, category, department, pio_name, pio_designation, jurisdiction, address)
legal_chunks    (id, act, section, title, body, embedding vector(1024))
embeddings      ivfflat index, lists = 100 (pgvector)
drafts          (id, request_id, body, citations[], c_d_score, verdict, created_at)
sla_clocks      (id, draft_id, filed_on, due_on, status, escalation_draft_id)
```

Row-Level Security ties every row to `citizen_id`.

### API surface

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/auth/otp/request` | Request phone OTP (rate limited) |
| `POST` | `/auth/otp/verify` | Verify OTP, receive 1-hour access token + httpOnly refresh cookie |
| `POST` | `/v1/requests` | Submit audio or text, returns `request_id` |
| `GET` | `/v1/requests/{id}/status` | Agent-by-agent progress and confidences |
| `POST` | `/v1/requests/{id}/confirm-authority` | Citizen confirms a candidate authority |
| `POST` | `/v1/requests/{id}/generate` | Run retrieve → draft → validate, return score and verdict |
| `GET` | `/v1/requests/{id}/letter` | Fetch the draft letter and its citations |
| `GET` | `/v1/requests/{id}/export` | PII-scrubbed PDF / text export (blocked if `C(d) < 0.85`) |
| `POST` | `/v1/requests/{id}/appeal` | Generate the First Appeal |
| `DELETE` | `/v1/requests/{id}` | Delete request and audio immediately |

---

## Tech stack

Every layer is chosen for Indian-language voice input, grounded answers and citizen data safety.

| Layer | Choice | Detail |
| --- | --- | --- |
| **Frontend** | React 18 + TypeScript | Vite, Tailwind CSS, Capacitor for the Android wrapper |
| **Backend API** | FastAPI | Python 3.11, async I/O, Pydantic |
| **Voice / ASR** | Bhashini | Whisper-large-v3 as fallback |
| **RAG / embeddings** | LangChain | Custom hybrid retriever with `text-embedding-3-large` and `bge-m3` |
| **Data / vector** | PostgreSQL 15 + pgvector | Hosted on Supabase, ivfflat index with `lists = 100` |
| **LLM / inference** | Gemini / Groq | Long-context, instruction-following inference |
| **Languages** | Kannada, Hindi, English | Speech, text and the generated letter |
| **Scheduling** | Cron / worker | 30-day SLA escalation and First Appeal generation |

---

## Security by design

| Control | Implementation |
| --- | --- |
| Sign-in | Phone OTP, rate-limited |
| Row access | PostgreSQL Row-Level Security |
| Personal data | Encrypted at rest (pgcrypto) + PII scrubbing before export |
| Sessions | 1-hour access tokens, httpOnly refresh cookies |
| Audio | Deleted after 24 hours unless the citizen opts to retain it |

Additional principles: no legal claim is emitted without retrieved grounding, no export below
`C(d) ≥ 0.85`, and every generated document is watermarked as a sample rather than legal advice.

---

## Repository contents

```
.
├── index.html   # Landing page + interactive front-end pipeline simulation
├── style.css    # Styles (Bricolage Grotesque + Noto Sans Kannada, WCAG-focused focus states)
└── README.md    # This document
```

The landing page is a static, dependency-free simulation so it can be opened with no build step.
The production application (React client, FastAPI service, retrieval corpus and Android wrapper)
lives in the deployment repository described in the project documentation.

---

## Local development

The landing page requires no tooling:

```bash
git clone https://github.com/VINAYAKANWALISHETTAR/Namma_Hakku.git
cd Namma_Hakku
python -m http.server 8000   # or: npx serve .
```

Browser support: Chrome/Edge for the Kannada mic input; all modern browsers for the typed flow.
Respects `prefers-reduced-motion` and prints cleanly to PDF.

---

## Roadmap

**Phase 1 — live**

- Pension disbursal, PDS / ration cards, municipal complaints
- Voice + text input in Kannada, Hindi, English
- Hybrid retrieval, grounded drafting, `C(d) ≥ 0.85` export gate
- 30-day SLA clock and First Appeal automation

**Phase 2 — next**

- Pan-India jurisdiction directory and PIO lookup
- Live CPGRAMS submission with status tracking
- More grievance categories and more Indian languages
- Offline/low-connectivity Android mode

---

## Limitations and ethics

- Generated letters are **samples, not legal advice**, and are labelled as such.
- NammaHakku does **not** replace legal or government institutions. It makes them easier for
  citizens to navigate.
- Retrieval quality is bounded by the curated legal corpus; anything not in the corpus is refused
  rather than guessed.
- A human reviewer stays in the loop for every low-confidence or low-score case.

> Technology should not expect citizens to understand the system. It should make the system easy
> for citizens to navigate.

---

## Creator

**Vinayaka N Walishettar** — solo builder
Visvesvaraya Technological University, Belagavi | Hubballi, Karnataka

- LinkedIn: <https://www.linkedin.com/in/vinayaka-n-walishettar/>
- GitHub: <https://github.com/VINAYAKANWALISHETTAR>
- Portfolio: <https://www.nayaka100.in/>
- Source code: <https://github.com/VINAYAKANWALISHETTAR/Namma_Hakku>

*Problem statement: Productivity — agentic AI for civic access. Designed and built end to end:
voice, retrieval, backend, safety layer and the Android-ready client.*

---

## License

Released for civic-access and educational use. See the repository license file.