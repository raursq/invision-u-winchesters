# inVision U — Solution Architecture

## 1. System overview

```mermaid
graph TB
    subgraph Frontend["Frontend · Next.js 16 / React 19"]
        APPLY[Applicant portal<br/>five-step application]
        SCEN[Scenarios<br/>conversation partner · EN / RU / KZ]
        DASH[Admissions dashboard<br/>committee and admin only]
    end

    subgraph Backend["Backend · FastAPI"]
        AUTH[auth · roles and JWT]
        CAND[candidates]
        LEDGER[ledger pipeline<br/>extract → verify → rate → assemble]
        SCENR[scenarios]
        OVR[overrides · append-only]
        COMM[committee<br/>pre-brief · decision memo · PDF]
        FAIR[fairness audit · evaluation]
        PRIV[privacy.py<br/>PII stripped before any model call]
        LLM[llm.py<br/>the only model client]
    end

    DB[(SQLite · SQLModel + Alembic)]
    CLAUDE[Claude API<br/>optional: demo mode without a key]

    APPLY --> CAND
    SCEN --> SCENR
    DASH --> LEDGER & OVR & COMM & FAIR
    CAND --> LEDGER
    SCENR --> LEDGER
    LEDGER --> PRIV --> LLM --> CLAUDE
    AUTH & CAND & LEDGER & SCENR & OVR & COMM & FAIR --> DB
```

The browser only talks to the frontend; Next.js proxies `/api` to the backend.
Every router checks the caller's role (`backend/routers/guards.py`).

## 2. Applicant flow

```mermaid
graph LR
    A[Sign up] --> B[Personal info] --> C[Education] --> D[Essay & motivation]
    D --> E[Activities & projects] --> F[Review & submit]
    F --> G[Evidence ledger built]
    F --> S[Scenario<br/>pick a situation and a language]
    S --> T[3–5 replies to the conversation partner]
    T --> U[Replies rated on the same rubric<br/>weight 0, the applicant sees no level]
```

## 3. Evidence ledger pipeline

```mermaid
graph LR
    SRC[Applicant's own text<br/>essay · interview notes · recommendation] --> X[Extract<br/>quote proposals per indicator]
    X --> V{Verbatim check<br/>is the quote in the source?}
    V -->|no| DROP[Dropped]
    V -->|yes| R[Rate each indicator<br/>on the BARS scale]
    R --> AS[Assemble<br/>level derived by rule R0–R3]
    AS --> L[(Competency rating<br/>level · quotes · ATOLA · source)]
```

- **Levels:** High / Normal / Weak / *No evidence*. "No evidence" is its own
  state, never a low score. A claim without described behaviour is capped at
  Normal.
- **Determinism:** the competency level is derived from indicator ratings by a
  fixed rule, so the same evidence always yields the same level.
- **Versions:** rubric and prompt content are hashed in
  `backend/ledger/versions.lock.json`; CI fails on a change without a version bump.
- **Demo mode:** without a model key, `backend/ledger/demo.py` replaces the
  model calls (hand-authored quotes for the 16 seed applicants, rule-based
  extraction for new ones). Every quote still passes the real verbatim check,
  and every view is labelled *"Demo mode · no live model"*.

## 4. Committee flow

```mermaid
graph LR
    D[Dashboard<br/>search · filters] --> C[Committee Card<br/>level + quotes per competency]
    C --> SC[Scenario result<br/>shown next to it, weight 0]
    C --> O[Change level<br/>reason code required]
    O --> H[(Override history<br/>AI level stays visible)]
    C --> IB[Interviewer brief<br/>what to probe]
    C --> M[Decision memo<br/>RU / KZ, PDF, signatures]
    D --> FA[Fairness audit<br/>High rate per group, 0.8 review line]
```

## 5. Fairness

- Scoring never reads school type, region, GPA, income, gender, application
  language, essay length or manner of speech
  (`tests/test_no_demographics_in_scoring.py` fails the build otherwise).
- The fairness audit compares the share of each group rated High per
  competency, with 95% bootstrap intervals; groups with n < 10 are marked
  "Not enough data".
- Swap-and-rescore changes name, region, school type, language and speech
  style and checks whether any level flips.

## 6. Data model

| Table | Holds |
|-------|-------|
| `applicants`, `artifacts` | applications and their documents |
| `users` | accounts and roles (applicant, interviewer, committee, admin) |
| ledger runs and scores | stored evidence ledgers with versions |
| `overrides`, audit log | committee changes, append-only |
| `feynman_sessions`, `scenario_results` | scenario conversations and their ratings |
| signatures | decision-memo sign-off |

Migrations live in `backend/migrations/`; `python -m backend.db init` brings a
database to head and seeds the demo data.
