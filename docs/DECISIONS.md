# Blackhill — Decision Register

The only place a business or architecture rule becomes binding.

**Status rules**
- `FROZEN` — approved by Ahmad. Binding on every agent. Only Ahmad moves a row to FROZEN,
  and only in a commit whose message starts with `DECISION:`.
- `PROPOSED` — a recommendation (from Claude, ChatGPT or Codex). Not binding.
- `OPEN` — needs an answer. Agents must not guess; they stop and ask.
- `SUPERSEDED` — replaced; keep the row, point to the replacement.

An AI agent may add PROPOSED/OPEN rows. It may never write FROZEN.

---

## FROZEN

| ID | Decision | Date | Source |
|---|---|---|---|
| F-01 | Support both permanent/direct-hire and travel/contract healthcare staffing. | 2026-10-31 | Ahmad, kickoff |
| F-02 | Initially 1–5 internal staff users. | 2026-10-31 | Ahmad, kickoff |
| F-03 | Candidate login is email-based. (Method: see D-01.) | 2026-10-31 | Ahmad, kickoff |
| F-04 | Production careers hostname: careers.blackhillinternational.com | 2026-10-31 | Ahmad, kickoff |
| F-05 | First operational release supports BOTH public applications AND recruiter submissions of clinicians to facility job orders, with per-submission stages. | 2026-10-31 | Ahmad, kickoff |
| F-06 | No full SSN at intake. Last-4 at most, and only once D-07 is frozen. | 2026-10-31 | Ahmad, kickoff |
| F-07 | Blackhill is fully isolated from PMS: own repo, database, storage, secrets, accounts, deployment. No PMS data ever enters Blackhill. | 2026-10-31 | Ahmad |
| F-08 | Product direction: a healthcare staffing operating platform (clinician → facility submission → credentialing → offer → placement → redeployment), not an applicant portal. | 2026-10-31 | Ahmad ("yes that's perfect") |
| F-09 | Reuse of PMS ATS code is optional. Design from first principles; port a PMS component only when it fits the Blackhill design. | 2026-11-01 | Ahmad |
| F-10 | Roles: Claude Code builds. ChatGPT reviews. Ahmad decides. No agent approves its own work or freezes a decision. | 2026-11-01 | Ahmad |

---

## OPEN — blocks the first slice (answer these first)

| ID | Question | Options | Recommendation |
|---|---|---|---|
| D-00 | Codebase strategy | (a) keep adapting the PMS export in ~/blackhill-ats; (b) fresh Django project with Blackhill's own models, porting selected PMS modules (documents, capabilities/scopes, audit, form versioning) deliberately | **(b)**. No data exists yet; the audit found the core models are PMS-shaped (CNIC identity, outcomes on Application, Pakistan-gated searchability). Porting proven infrastructure into a clean domain model avoids carrying 62k lines of mass-hiring logic and tests for the wrong product. Makes migration-reset question moot. |
| D-01 | Email login method | 6-digit one-time code · magic link · password | **One-time code**, ~10 min expiry, rate-limited. Links break across devices/in-app browsers. |
| D-02 | Claiming a recruiter-imported profile | name+DOB match · invitation link · staff-reviewed claim | **Invitation sent by staff to the profile's email**, or staff-reviewed claim. Never name/DOB alone. |
| D-03 | Staff access | same host · separate host; MFA timing | **Separate staff hostname; MFA required before any real candidate data.** |
| D-04 | Submission stages (initial list) | — | Placeholder: Shortlisted → Submitted → Client reviewing → Interview → Offer → Placed, plus Rejected/Withdrawn from any stage. Needs a Blackhill recruiter's confirmation. |
| D-05 | Candidate readiness | one "complete" flag · separate states | **Three independent facts:** searchable (any staff-visible record), profile complete, submission-ready. |

## OPEN — needed before go-live

| ID | Question | Recommendation |
|---|---|---|
| D-06 | Credential model | Three records: claim (what clinician says) → evidence (document link) → verification (who, when, source). Status derived; upload ≠ verified. |
| D-07 | SSN-last-4: purpose, optional/required, who can see, retention | Not collected until purpose is written down by Blackhill. |
| D-08 | Hosting region and provider | US region. |
| D-09 | First disciplines and states | Ask Blackhill which roles they place first (e.g. RN + 2 allied). Seed only those. |
| D-10 | Pay display and facility-name visibility on public postings | Configurable per posting; check state pay-transparency rules. |
| D-11 | E-signature, email and dialer providers | Get Blackhill's current vendors before designing adapters. |
| D-12 | Data retention / deletion | Needs counsel. |

## PROPOSED

(Agents add rows here. Ahmad promotes to FROZEN or rejects.)

| ID | Proposal | Proposed by | Date |
|---|---|---|---|

---

## Inputs (not binding)
- Codex architecture drafts (2026-10-31): docs/drafts/ — Consideration + ClientSubmission,
  code-defined status sets, JobOrder vs Vacancy, pay rates in a restricted table.
- Sprint 0 audit: ~/blackhill-audit/sprint0_report.md
