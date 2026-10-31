# BH-STAFF-01 — Facility Submissions

Status: **DRAFT for approval. Not implemented.** Depends on BH-AUTH-01 for the candidate identity service. Submission
**status/stage values and transition rules are deliberately unspecified** (see §6): this spec defines the mechanism, not
the vocabulary.

## 1. Goal

A recruiter takes an **existing Candidate**, submits them to a **facility job order**, and tracks that submission
through its own stages, independent of the candidate's profile and of any candidate-initiated Application. Access is
limited by capability and by data scope, and the same candidate cannot be actively submitted to the same job order twice
without an approved exception.

## 2. Why this is new (inherited code does not provide it)

- `Application` is the *candidate's own* application to a `Vacancy`, bound to a form version, eligibility attempt,
  declaration and snapshot (`apps/applications/models.py:69-210`). A recruiter submission is a different business
  object, so reusing it would break the inherited invariants **Candidate ≠ Application** and **Vacancy status ≠
  Application status**.
- `Client` (`apps/vacancies/models.py:31-70`) holds name/logo/slug only; there are no facilities, contacts, addresses or
  job-order attributes. `Client` is also the **staff data-scope boundary** (`apps/access_control/scope.py:28-56`) and the
  public-careers brand.
- Reusable patterns: `ApplicationStageEvent` (append-only history), bulk stage transition service
  (`applications/services.py:418,480`), capability decorator (`access_control/capabilities.py:114`), client scoping with
  fail-closed 404s, per-domain audit events, the Databank read models (`candidate_database`).

## 3. Domain model (new app `apps/submissions`)

| Model | Key fields / rules |
|---|---|
| `Facility` | `client` FK (existing `Client`, PROTECT), name, structured US address (line1/2, city, state, ZIP), time zone, `is_active`. Unique (client, name). |
| `FacilityContact` | facility FK, name, role, email, phone, `is_primary`. Contact PII: not exposed to candidates. |
| `JobOrder` | `facility` FK (PROTECT), reference (unique per client, human-readable), title/specialty, status (open/on hold/filled/closed: simple, **job-order status ≠ submission stage**), openings count, start date, optional free-text notes. Rates, shifts, duration and travel-vs-perm are **excluded** until decided (D2). |
| `Submission` | `candidate` FK (PROTECT, UUID), `job_order` FK (PROTECT), `submitted_by` (staff user), `stage` FK to `SubmissionStage`, `is_active` bool, `exception` nullable FK, timestamps. |
| `SubmissionStage` | Data-driven rows: key, label, `is_initial`, `is_terminal`, order. **Contents unspecified here** (§6). |
| `SubmissionStageEvent` | Append-only: submission, from_stage, to_stage, actor, reason/note, timestamp. Immutable via `save()`/`delete()` guards, like `ApplicationStageEvent`. |
| `SubmissionSnapshot` | Immutable JSON captured at submit time: candidate profile fields sent, document version IDs and sha256, schema version. Required by the inherited principle *current profile ≠ historical submitted truth*. |
| `SubmissionException` | Approved duplicate exception: submission, requested_by, approved_by (different person, holds the override capability), reason (required), timestamp. |
| `SubmissionAuditEvent` | View/export of candidate data through the workspace (who, which candidate, which fields class). |

`JobOrder` is **not** a `Vacancy`. A later optional link from `JobOrder` to a public `Vacancy` can be added if Blackhill
publishes jobs; that is outside this slice (D1).

## 4. Rules

1. **Reuse the Candidate.** Selection is from existing Candidates the recruiter's scope allows. Creating a Candidate
   from the workspace goes through BH-AUTH-01's single email-keyed service; never a second path.
2. **No duplicate active submission.** Partial unique index on `(candidate, job_order) WHERE is_active AND
   exception_id IS NULL`. `is_active` is set false only by the transition service when the target stage `is_terminal`.
   Creation runs in a transaction that locks the candidate row and re-checks, so concurrent submits cannot both pass.
   A second active submission requires a `SubmissionException` approved by a user holding
   `submission.override_duplicate` who is **not** the requester.
3. **Independent history.** Every stage change goes through one service function, writes a `SubmissionStageEvent`, and
   never edits previous events. A submission that is withdrawn/closed is retained, never deleted.
4. **Snapshot on submit.** The packet sent to the facility is frozen; later profile or document changes do not alter it.
5. **Candidate profile gate.** Submitting requires the BH-AUTH-01 profile-completeness rule to be satisfied (D5 may
   relax it per job order).
6. **Authorization is server-side and capability-based** (never role names). New capability codes, registered in
   `access_control/capabilities.py` with sensitivity flags:
   `facility.view`, `facility.manage`, `joborder.view`, `joborder.manage`, `submission.view`, `submission.create`,
   `submission.transition`, `submission.override_duplicate` (sensitive), `submission.export` (sensitive).
   Existing `candidate.view*` capabilities still govern candidate data shown inside the workspace.
7. **Data scope.** Reuse the client scope (`ALL` or selected clients): a recruiter sees only facilities, job orders and
   submissions whose `facility.client` is in scope; crafted URLs return 404; no profile ⇒ fail closed. Whether a
   recruiter additionally sees only *their own* submissions or the whole client team is decision D3.
8. **Sensitive data.** The workspace shows only what the viewer's capabilities permit; documents are downloaded through
   the existing protected, audited document view (capability + scope + audit event before bytes). Masked identity fields
   stay masked.

## 5. Recruiter submission workspace (minimum useful slice)

Presentation is part of done: uses the shared design system, dense layout for staff, no default-Django screens.

1. **Job orders list** (scoped): reference, facility, title, status, openings, active-submission count; filters; search.
2. **Job order detail**: header, facility/contact summary, submissions table (candidate, submitted by/date, stage,
   last change), "Submit a candidate" action.
3. **Submit candidate**: search/select from the scoped Databank (reuse `candidate_database` queries), show profile
   completeness, warn on an existing active submission and link to it, confirm, create + snapshot.
4. **Submission detail**: snapshot summary, stage timeline, stage-change control (options come from the stage table;
   empty until §6), exception request/approval.
5. Facility and job-order create/edit (capability-gated). Client-facing portal, email packets and exports are out of scope.

## 6. Deliberately not decided: submission stages

Not implemented, not seeded, not named in code: the stage vocabulary, which transitions are legal, who may make each
transition, which stages are terminal, whether any stage is visible to the client or the candidate, and whether stages
differ for permanent vs travel/contract placements. This slice ships the table, the event log, the single transition
service and a single placeholder `is_initial` stage only if required to create a row, plus tests that prove the
mechanism with fixture stages. The real values arrive in a follow-up spec (BH-STAFF-02) once approved.

## 7. Acceptance criteria

1. A recruiter with `submission.create` in scope submits an existing Candidate to an in-scope job order; a snapshot and
   an initial stage event exist.
2. The same candidate/job order cannot have two active submissions (UI, service and DB all refuse); a concurrent
   double-submit yields one row; an approved exception from a different user allows exactly one more.
3. A recruiter outside the client scope gets 404 for that facility, job order, submission and candidate-in-workspace;
   removing a capability removes the action server-side even if the URL is guessed.
4. Stage changes only through the service; events are append-only; attempting to update or delete an event raises.
5. Editing the Candidate profile afterwards does not change the submission snapshot.
6. Withdrawn/closed submissions remain listed (filterable) and still block nothing once inactive.
7. Document access from the workspace writes the existing sensitive-access audit event before streaming.
8. Candidate-facing views never show submissions, facilities or contacts.

## 8. Test plan

Heavy: scope and capability enforcement (matrix of capability × scope × object), duplicate prevention incl. concurrency
and exceptions, snapshot immutability, event immutability, document access, candidate-surface leakage, completeness
gating. Light: list/detail layout, with one mobile-width overflow check for the workspace.

## 9. Out of scope

Stage vocabulary and rules, client portal, billing/rates/margins, credential expiry alerts, e-signature/offers/contracts,
dialer/SMS, email packets to facilities, job-board feeds, reporting.

## 10. Decisions required before build

- **D1** Confirm `JobOrder` as a new object separate from public `Vacancy`.
- **D2** Job-order fields that must exist at launch (rates, shift, duration, perm vs travel, bill/pay).
- **D3** Recruiter data scope: client-based only (inherited) or also "own submissions only".
- **D4** Is the facility a legal entity under a `Client`, or are Client and Facility the same thing for Blackhill?
- **D5** Can a recruiter submit an incomplete-profile Candidate?
- **D6** Who may approve duplicate exceptions, and is a reason code list needed.
- **D7** What the facility receives (a packet, a portal, nothing yet) because it fixes what the snapshot must contain.
