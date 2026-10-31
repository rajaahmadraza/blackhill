# Blackhill ATS — Decision Register v0.1

Date: 2026-11-01 · Status: **DRAFT.** Nothing under PROPOSED or REQUIRES AHMAD'S DECISION is approved.
Companion to `docs/architecture/ARCHITECTURE_AND_INVARIANTS_v1.0_DRAFT.md` ("Arch") and `DOMAIN_MODEL_v0.1.md`.

How to read: section 3 is ordered by **what it blocks**. Tier 0 blocks *any* code; Tier 1 blocks the first deliverable
(candidate sign-in, clinician profile, public applications); Tier 2 blocks facility submissions; Tier 3 blocks the second
release. "Before real data" items do not block building but must be closed before any real candidate data is loaded.
Every item has a recommendation; I have marked the ones where I think reasonable people could disagree.

---

## 1. AGREED / FROZEN

Source: the brief (items A-1…A-8 were stated as AGREED) and `AGENTS.md`. These are not re-opened here.

| ID | Decision |
|---|---|
| A-1 | Support permanent/direct-hire **and** travel/contract healthcare staffing |
| A-2 | Initially 1–5 internal staff users |
| A-3 | Candidate authentication is email-based |
| A-4 | Intended careers domain `careers.blackhillinternational.com` |
| A-5 | First operational release supports **both** public candidate applications **and** recruiter-to-facility submissions |
| A-6 | Blackhill stays operationally and technically isolated from PMS production |
| A-7 | Build a healthcare staffing operating platform, not only an applicant portal |
| A-8 | Reuse appropriate PMS architecture without sacrificing integrity, security or maintainability |
| A-9 | Hard separation from PMS: no PMS remote/data/containers; `blackhill*` DB-name guard is not weakened; synthetic data only (`AGENTS.md`) |
| A-10 | Inherited invariants in `AGENTS.md` stand until formally changed (Candidate ≠ Application; immutable Candidate ID; Vacancy ≠ Application status; profile ≠ historical truth; immutable published form versions; separate candidate/staff domains; capability-based server-side authorization; withdrawn Applications retained **subject to D-20**; sensitive-data access control) |
| A-11 | Stack: Django modular monolith + PostgreSQL, server-rendered templates; no microservices/SPA/event sourcing/message broker without measured need |
| A-12 | **No SSN policy is approved.** Preference only: none collected at recruitment, possibly last-four, full SSN possibly later at onboarding. Nothing may assume a policy |
| A-13 | Architect for the whole lifecycle; implement only what each approved release needs |
| A-14 | No implementation until Ahmad reviews the architecture |

## 2. PROPOSED (my recommendations; low controversy; Ahmad may veto but need not decide)

| ID | Proposal | Where |
|---|---|---|
| P-01 | Candidate login by **email one-time code** (not magic link) in V1; magic link may be added later as another delivery of the same challenge | Arch §6.2 |
| P-02 | **Separate `ClinicianProfile`** (1:1 Candidate) for clinical scalars/preferences; collections are separate tables | Arch §4.2 |
| P-03 | Login email lives on `CandidateAccount` (unique, normalised: trim → NFKC → lower-case, **no** provider-specific folding); staff-entered `contact_email` lives on `Candidate`, unverified, non-unique; phone stored E.164, unverified, never unique | Arch §6.1 |
| P-04 | Duplicate signals (phone, name+DOB, **credential identifier**) only warn; never auto-merge. Candidate merge deferred; a `MERGE_RELATIONS` registry test lands early | Arch §6.3 |
| P-05 | **Code-defined status sets** with `status` + `is_open` CHECK and one transition map per lifecycle, instead of BH-STAFF-01's data-driven `SubmissionStage` rows | Arch §4.3 |
| P-06 | Credentials = claim + evidence link + **immutable verification**; effective status is *derived* via the L0–L3 ladder; an upload never exceeds "document attached" | Arch §8 |
| P-07 | One `Activity` table with explicit nullable anchor FKs (no generic FKs); manual logging in MVP | Arch §10 |
| P-08 | Constraint-first concurrency: partial unique indexes arbitrate; lock order Candidate → JobOrder → Consideration → ClientSubmission → Offer → Placement; real-thread test as in `tests/test_lock_order.py` | Arch §12.4 |
| P-09 | **Standalone deployments from one codebase**; no tenant column/manager/admin | Arch §11 |
| P-10 | Scheduled work = idempotent management commands run by host cron; no Celery/worker until measured need | Arch §8.4, §12.7 |
| P-11 | Sensitive identifiers (when/if approved): separate `onboarding` app, field-level AEAD with external key, audited reveal with step-up re-auth; never on Candidate/Profile/snapshots/packets/exports | Arch §9.3 |
| P-12 | Dates (job start/end, credential expiry, availability) are `DateField`s interpreted in the **facility's** IANA timezone; instants are UTC | Arch §12.6 |
| P-13 | Offer generalised from `ApplicationOffer` (parent Consideration; schema-versioned frozen `terms`), not a second offer system; Release 2 | Domain §3.8 |
| P-14 | Pay/bill rates live in a separate restricted table, never on the `JobOrder` row | Arch §9.1 |
| P-15 | Licence **expiry** has one source of truth (`ProfessionalCredential.expires_on`); document-version `expiry_date` is ignored for credentials | Arch §8.4 |
| P-16 | `Vacancy` keeps its name in code and is the public *posting*; nullable `job_order` link | Arch §4.2 |

---

## 3. REQUIRES AHMAD'S DECISION

### Tier 0 — blocks any implementation

#### D-06 · Migration history: reset once, or keep PMS history? **(FOUNDATIONAL)**
BH-AUTH-01 §6 recommends keeping inherited migrations and using expand/contract. Blackhill has no data and no deployment.

| Option | Trade-offs |
|---|---|
| **1. Reset once at slice 0 (recommended)** | Clean schema with no CNIC/Urdu/PKR columns, no nullable-forever fields, no legacy-row tests. Upstream PMS fixes are then ported by diff against the recorded base SHA (migrations couldn't merge anyway once identity diverges). Cost: must be done *before* any environment holds data worth keeping; the existing `test_ph08_migration`-style legacy tests are deleted |
| 2. Keep history, forward migrations | Retains the ability to `diff` migrations vs upstream, but every PMS migration (62) ships, Pakistan columns linger until a later contract step, and AUTH-01's "snapshots byte-identical" tests protect rows that cannot exist |

Rule either way: from the first moment real/UAT data exists, history is append-only. **Also decide here:** initialise a local git repo now (the directory has none) with **no remote**, and create the missing `docs/blackhill/PROVENANCE.md` (upstream base `7cec319`).

#### D-29 · What to do with inherited PMS modules and features
| Item | Recommendation |
|---|---|
| `intake`, `interview_import` (CNIC-keyed, ~5.5k LOC) | **Remove** at slice 0 |
| Urdu locale, Urdu publish rule, bilingual eligibility | **Remove** (English only; keep Django i18n machinery) |
| `recommendations`, `screening` (queue mode), `help` (PMS/UR copy) | Keep **disabled/unlinked** until a Blackhill need is stated; rewrite or delete `help` |
| Interviews/scorecards (Application-bound) | Keep as is, unlinked; Blackhill interview process unknown |
| Talent pools | Keep as recruiter lists; **must not** gate searchability (D-12) |
| `OGDCL` seeds, PMS branding, UAT seed tooling with CNIC literals | Remove / replace |
| Permission presets | Redefine for Blackhill (data change) |
The alternative (leave everything and "hide") keeps ~57 CNIC-referencing files alive and every test red. Removal is not an "unrelated refactor": it is the first slice.

#### D-27 · Company default timezone
Deployment setting (default currently Karachi, wrong). Which US timezone is Blackhill's office default for staff UI? Facilities carry their own. **Recommendation:** ask; set via env; no code branch.

### Tier 1 — blocks first deliverable (sign-in, profile, public applications)

#### D-01 · Is email the sole candidate identity? Is a verified phone ever required?
| Option | Trade-offs |
|---|---|
| **Email sole; phone optional, unverified (recommended)** | Simplest; no SMS vendor/TCPA exposure at launch; recruiters still hold a phone number for calling |
| Email + verified phone | Stronger identity for fraud-prone staffing, but requires an SMS provider, consent handling and international delivery issues |

#### D-02 · Public self-registration vs invite-only at launch
**Recommendation: open** — A-5 requires public applications. Mitigate with throttles (per email, per IP, global), uniform responses, optional bot challenge only if abuse appears.

#### D-03 · Shared mailbox
**Recommendation: not supported for login** (one account per email); staff `contact_email` may repeat. Edge: spouses/families sharing an inbox must use separate addresses.

#### D-04 · Claiming a staff-created/imported profile
| Option | Trade-offs |
|---|---|
| A. Auto-attach on matching verified email | Smoothest, but a recruiter typo would expose a stranger's profile to the wrong person |
| **B. Attach on matching verified email **plus** DOB/name confirmation; mismatches go to a staff queue (recommended)** | Small friction; limits typo exposure; no knowledge-based secrets beyond data the candidate already knows |
| C. Never attach; create new account and let staff merge | Safest, but guarantees duplicates and defers work to staff |

#### D-05 / D-28 · Separate candidate and staff cookies via hostnames
Today both domains share one cookie (verified; Arch §2). **Recommendation:** staff on a different hostname than `careers.blackhillinternational.com` (host-only cookies do not cross), plus a path-prefix guard per host. Ahmad to choose the staff hostname (e.g. `staff.` or `ats.` under the company domain) and confirm DNS/TLS ownership. Alternative (custom session middleware with two cookie names) is more code and more risk.

#### D-12 · Searchable vs complete vs submission-ready
Inherited rule: a candidate is invisible in the Databank until the profile is "complete" (PK fields). That would hide recruiter-imported candidates.
| Option | Trade-offs |
|---|---|
| **Three concepts (recommended):** *searchable* = any active candidate; *complete* = defined profile fields; *submission-ready* = per-job-order check (profile + consent + credentials per D-19) | Matches how recruiters work; "ready" can differ for perm vs travel; more explicit rules to write |
| Keep a single "complete" gate | Simple, but blocks Scenario B and forces fake data to appear in search |
Ahmad to supply the **field list for "complete"** (BH-AUTH-01 D3 proposed: legal name, verified email, phone, DOB, US address, primary profession/specialty).

#### D-21 · Work-authorisation / I-9 data
Collect at all before offer? **Recommendation: do not collect in recruitment**; revisit with counsel. Blocks only the profile field list.

#### D-22 · Pay transparency on public postings
Several states require pay ranges in job ads. Does Blackhill post in those states? **Recommendation:** add optional `pay_range_min/max`/`pay_basis` display fields on the *posting* (not the rates table) and confirm with counsel. Blocks posting design only if answer is yes.

#### D-23 · Public disclosure of facility/client identity
Staffing ads often hide the facility. Existing `Client` carries a public name/logo and the careers URL `careers/<client_slug>/`.
**Recommendation:** default to **not** showing client/facility; disclosure is an explicit per-posting choice; the public URL must not expose the client slug unless disclosed. Ahmad to confirm Blackhill's practice.

#### D-24 · Staff MFA (blocks real data, not slice work)
**Recommendation:** TOTP for all staff before any real candidate data; recovery by another admin; no SMS. Alternative: SSO via the company's identity provider if one exists (ask).

#### D-19 (part 1) · Credential types and alerting — decide before slice 2
Which licenses/certifications to seed (RN/LPN/CNA state licences, BLS, ACLS, PALS, NRP, TNCC…), whether a **compact** flag is needed, expiry alert windows (90/60/30/0?) and recipients (candidate, recruiter, compliance). Also: are references collected in MVP? **Recommendation:** seed a small list, make it editable, alerts at 60/30/0 to recruiter + candidate, references deferred.

### Tier 2 — blocks facility submissions

#### D-07 · JobOrder vs public Vacancy (BH-STAFF-01 D1)
| Option | Trade-offs |
|---|---|
| **A. JobOrder separate; Vacancy = optional posting, nullable link (recommended)** | Matches reality (not every job order is advertised; one order may have several postings; a "talent community" posting has no order); keeps immutable form versions untouched; two objects to explain |
| B. Extend Vacancy to be the job order | Fewer tables, but forces facility-confidential data and employment-type rules onto a public object, and form-version immutability collides with job-order edits |
| C. Fully unrelated | Loses applicant → consideration traceability |

#### D-08 · Consideration entity
| Option | Trade-offs |
|---|---|
| **A. Consideration + ClientSubmission (recommended)** | Clean "submission date", consent checked at the right moment, Scenario-H arbitration earlier; one extra small table |
| B. Single `Submission` with `presented_at` and two independent status columns | Fewer tables; acceptable fallback; conflates internal and client-facing lifecycles over time |
| C. Single `Submission` with `PRE_SUBMISSION` stage | Not recommended; semantics and reports degrade; costly to split later |

#### D-09 · Employment types and minimum job-order fields (BH-STAFF-01 D2; conflicts with the brief)
**Recommendation (minimum, no rates):** employment type (`PERMANENT`, `CONTRACT`; extend only if Blackhill names more, e.g. per-diem), profession, specialty, facility, openings, start date/window, contract duration (required for CONTRACT), shift (structured + text), requirements text, `client_reference`. Rates/margins later in a restricted table. Ahmad to confirm the employment-type list and whether bill/pay is needed at release one.

#### D-10 · Recruiter data scope (BH-STAFF-01 D3)
**Recommendation:** keep client-based scope (`ALL` for all 1–5 users at launch); record an `owner` on Considerations; do **not** restrict recruiters to their own records now. Alternative "own records only" adds a second filter dimension to every list with little benefit at this size.

#### D-11 · Client vs Facility (BH-STAFF-01 D4)
**Recommendation:** Client = contracting organisation and staff scope boundary; Facility = site (address, timezone) under it; job orders attach to a Facility. If every Blackhill customer is a single site, the Facility row is simply 1:1 — the cost of keeping both is small, the cost of adding Facility later is large.

#### D-13 · Duplicate-exception approver (BH-STAFF-01 D6)
With 1–5 staff a strict "different person" rule can deadlock. **Recommendation:** capability `submission.override_duplicate` held by a manager; approver must differ from requester **unless** only one holder exists, in which case self-approval is allowed with a mandatory reason and is flagged in the audit view. Reason-code list: small and editable.

#### D-14 · What the facility receives and how (BH-STAFF-01 D7)
| Option | Trade-offs |
|---|---|
| **A. System generates a frozen packet (rendered file + hash); recruiter sends it by email outside the system; send is logged as an Activity (recommended for MVP)** | Smallest build; the exact disclosed content is still frozen and auditable; delivery isn't tracked automatically |
| B. System emails the packet to the contact | Delivery status/idempotency needed; email deliverability work |
| C. Client portal with secure links | Highest value later; needs client authentication domain, a third security domain |
Also fixes what `SubmissionPacket` must contain.

#### D-15 · Status vocabularies, transitions, who may make them, visibility
Needed for: Application (review-only subset), Consideration, ClientSubmission (and reason codes for rejection/withdrawal/close). The illustrative sets in Arch §7 are placeholders. **Ahmad must approve:** the values; legal transitions; the capability for each; which are terminal; whether `CLIENT_SELECTED` auto-opens an offer; "ready to submit" prerequisites. Blocks the *submission* slice only; the mechanism can be built and tested with fixture statuses.

#### D-16 · Representation consent
Required before every submission? Scope (job order / facility / client / general)? Evidence (portal checkbox, email reply, recorded verbal)? What may be shown to the facility (name, contact details, employer names)? Is cross-facility submission allowed while another is open? **Recommendation:** consent record is **required** before `ClientSubmission`; scope default = job order; evidence free (portal/email/verbal by staff with Activity reference); disclosure level starts at "name withheld until client selects" **only if** Ahmad confirms Blackhill's practice; wording is a legal artifact.

#### D-17 · Does the candidate see submissions/offers?
**Recommendation (MVP):** no candidate-facing view of submissions or facilities (consistent with BH-STAFF-01 AC#8); communication happens through the recruiter. Revisit for offers (inherited CHANGE_004 could not be verified here).

### Tier 3 — second release and beyond

#### D-18 · SSN (A-12 stays binding)
Options to choose from when the time comes:
| Option | Notes |
|---|---|
| **A. Collect none in recruitment; full SSN only via a background-check/onboarding provider's hosted form post-offer (recommended)** | Blackhill never stores it; minimal liability; depends on vendor choice |
| B. Last-four at recruitment, encrypted, restricted | Only if a facility/vendor needs it that early; still personal data; not an auth factor |
| C. Store full SSN post-offer | Highest risk; needs field encryption, key management, retention, breach readiness; avoid |
Not required for any slice before offers.

#### D-25 · Offers: scope and candidate visibility (Release 2)
Re-parent offers to Consideration; terms schema for perm vs travel; e-sign vendor (DocuSign?) and signers (candidate, client, Blackhill); candidate-visible or not.

#### D-26 · Placements
Extension = same placement with a new term (recommended) vs new placement; permanent-placement guarantee/rebate period handling; can one candidate hold overlapping active placements; early-termination reason codes.

### Before real data (do not block building)

| ID | Decision | Recommendation |
|---|---|---|
| D-20 | **Retention and deletion** — the inherited "withdrawn Applications are retained" invariant versus US privacy/deletion rights and EEO retention minimums | Counsel to define retention periods per record class; until then do not promise deletion or permanence to candidates; design for *anonymise-in-place* of person fields while keeping opportunity history |
| D-30 | Backups, restore test, HSTS ≥ 1 year, AV scanning hook, CSP | Required before launch; slices after the first |
| D-31 | Production deployment rewrite (compose/nginx/scripts/runbook are PMS-named) | Rewrite before first deployment; today they exist only because inherited hardening tests read them |

### Legal / compliance review items (not Ahmad's to answer; need counsel)
HIPAA applicability per data class · FCRA/state background-check rules · EEO/OFCCP recordkeeping · state applicant-privacy laws (e.g. California) and SSN laws · I-9/E-Verify timing · ESIGN/UETA and staffing-agency licensing · TCPA/SMS consent and two-party call-recording consent · exclusion-list screening duties · pay-transparency · consent language for client disclosure. Full list: Arch §9.4.

---

## 4. Decisions blocking each slice (summary)

| Slice (see sequence doc) | Must be decided first |
|---|---|
| S0 Foundation clean-up | D-06, D-29, D-27 |
| S1 Email identity | D-01, D-02, D-03, D-04, D-05/D-28 |
| S2 Clinician profile + credentials | D-12 (field list), D-19 part 1, D-21 |
| S3 Public postings + applications | D-22, D-23, (D-15 for the Application subset) |
| S4 Facilities, job orders, staff-created candidates | D-07, D-09, D-10, D-11 |
| S5 Consideration + consent + activities | D-08, D-15 (Consideration), D-16 |
| S6 Client submission | D-13, D-14, D-15 (submission), D-17 |
| S7 Credential verification + expiry alerts | D-19 |
| Real data (any slice) | D-24, D-20, D-30, D-31 |
| R2: Offers, placements, onboarding | D-18, D-25, D-26 |
