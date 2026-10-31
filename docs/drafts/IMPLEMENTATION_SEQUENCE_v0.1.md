# Blackhill ATS — Implementation Sequence v0.1

Date: 2026-11-01 · Status: **DRAFT. Plan only. Nothing here has been run or built.**
Prerequisite: Ahmad reviews `ARCHITECTURE_AND_INVARIANTS_v1.0_DRAFT.md` and closes the Tier-0 decisions in
`DECISION_REGISTER_v0.1.md`.

Rules for every slice (from `AGENTS.md`, restated): read the spec first; narrowest tests first; the real frontend is part of
done (shared design system; candidate flows mobile-first; staff screens dense); no security weakened to pass a test; stop and
report if an approved invariant would have to change; tests run only against the `blackhill-ats-db` container.

Tests below are **described, not executed**. "Heavy" = risk-based must-have; light items are listed only where useful.

---

## Sequence overview

```
S0 Foundation clean-up ─▶ S1 Email identity ─▶ S2 Clinician profile + credentials ─▶ S3 Postings + applications
                                                                                       │
              ═══ MILESTONE M1: "first deliverable" (candidate side complete) ═════════╝
                                                                                       ▼
S4 Facilities, contacts, job orders, staff-created candidates ─▶ S5 Consideration + consent + activities ─▶ S6 Client submission
              ═══ MILESTONE M2: "first operational release candidate" (needs MFA, backups, retention decision) ═══
S7 Verification workflow + expiry alerts ─▶ S8 Merge + claim hardening ─▶ R2: offers → placements → clearance → e-sign
```

Rough sizing assumes solo AI-assisted work; treat as relative, not a commitment: S0 M · S1 L · S2 L · S3 M-L · S4 M · S5 M · S6 L · S7 M · S8 M.

---

## S0 — Foundation clean-up (no Blackhill business behaviour)

**Depends on:** D-06, D-29, D-27.
**Scope:** `git init` locally with **no remote**; write `docs/blackhill/PROVENANCE.md` (upstream base `7cec319`, what was excluded);
remove `intake`, `interview_import`, Urdu locale/rule, OGDCL seeds, CNIC-literal fixtures and PMS UAT seeds; set `TIME_ZONE`
via env to the Blackhill choice; strip hard-coded "PMS" from shells/emails/branding into settings; run the migration reset
(if D-06 = 1); get the whole remaining suite green **on the Blackhill container** and record the baseline pass/fail (currently
unknown; the suite was never run in this repo).
**Not in scope:** any identity or domain change.
**Acceptance:** `manage.py check` and `makemigrations --check` clean; full suite result recorded in `PROVENANCE.md`; no tracked file
contains `CNIC`-shaped literals from PMS tests; no template names PMS or OGDCL; no remote configured.
**Security tests (heavy):** `settings` still refuses non-`blackhill*` DB names; no secret in the tree (grep gate); dev `SECRET_KEY`
fallback removed.
**Risk:** deleting tests that protect behaviour we keep. Mitigation: delete only tests of removed modules; every other failing test is
fixed or consciously rewritten, never skipped.

## S1 — BH-AUTH-01: US email identity

**Depends on:** S0; D-01–D-05, D-28.
**Scope:** `CandidateAccount.login_email_normalized` (unique), `Candidate.contact_email_normalized`, `phone_e164` (+ `phonenumbers`),
remove CNIC/Pakistan mobile from forms, OTP, throttles, completeness, snapshots (schema v2 with email/phone); email OTP flow with
uniform responses; claim flow (D-04 option B); email change with step-up OTP and old-address notice; staff-assisted email change
capability; `MERGE_RELATIONS` registry test; hostname-split middleware (D-05); session rotation retained.
**Acceptance:** BH-AUTH-01 §7 criteria 1–6, 8, 9 (criterion 7 is moot after a reset); plus claim and hostname criteria below.
**Security/acceptance tests (heavy):**
- normalisation: whitespace, case, Unicode forms resolve to one account; Gmail dots/`+tags` do **not** merge.
- concurrency: two simultaneous first sign-ins for one email yield one `Candidate` (real threads, constraint + retry).
- enumeration: known vs unknown email responses identical in body/status; timing in the same class (threshold test, not microsecond).
- OTP: expiry, single use, max attempts, resend cooldown, daily cap per email, per-IP throttle, hashed at rest, invalidated on new issue,
  not valid across emails.
- ownership: Candidate A gets 404 for B's profile, credentials, documents, applications by URL tampering.
- email change: needs fresh OTP at the **new** address; old address notified; collision refused with neutral message; security event written; stale session after change.
- claim: matching email + DOB confirms; mismatch → staff queue, no data revealed; two account-less candidates with the same contact email → no auto-attach.
- domain separation: candidate session cookie not sent to staff host and vice versa; staff URL on careers host → 404; a candidate session grants no staff capability; staff logout does not clear candidate session.
- no Pakistan-only validator reachable from any sign-in path (grep + behavioural).
- international phone numbers accepted; garbage rejected; phone never used as lookup.
**Light:** copy, layout, one 390 px overflow check on sign-in.
**Done also means:** the sign-in and code-entry pages look like a Blackhill product on mobile.

## S2 — Clinician profile and self-reported credentials

**Depends on:** S1; D-12 (field list), D-19 part 1, D-21.
**Scope:** `Profession`/`Specialty` (seeded, editable), `ClinicianProfile` (preferences, `preferred_states` array + GIN),
`ClinicalSpecialty`, `Employment` extensions, structured US address on `CandidateProfile`; `CredentialType`,
`ProfessionalCredential` (claim, event log), `CredentialEvidence` → existing wallet versions; one derived-level function
(L0–L3 + expired); new completeness rule replacing the PK rule; candidate profile screens (mobile-first) and staff profile view.
**Acceptance:** a candidate completes a US clinician profile and adds a license with a document; the derived level reads "Self-reported"
or "Document attached", never "Verified"; staff see the same data read-only; searchable per D-12.
**Tests (heavy):**
- derived level: upload alone never above L1; document review never above L2; only primary-source verification reaches L3; expired overrides;
  freshness demotion; boundary dates around midnight in **facility/local timezone**, not UTC.
- uniqueness: same (type, jurisdiction, identifier) for one candidate refused; retire then re-add allowed.
- ownership: evidence can only link a version belonging to the same candidate (cross-candidate version ID rejected).
- immutability: evidence link and verification rows cannot be updated/deleted (single and bulk).
- completeness gating per D-12; no father/husband/domicile anywhere.
- privacy: license identifier masked by default; reveal requires capability and writes a sensitive-access event (extend CHECK).
- search: profession+state+availability filters use the intended indexes (assert `EXPLAIN` uses index scan on a seeded set); no N+1 on list (query-count assertion).

## S3 — Public postings and applications (US adaptation)

**Depends on:** S2; D-22, D-23, D-15 (Application subset).
**Scope:** `Vacancy` adjustments (client hidden by default, optional pay-range display, remove Urdu rule, nullable `job_order` FK
added now but unused); US profile-field requirements; snapshot schema v2 content; reduced Application review vocabulary; staff applicant
screening works end to end; public careers pages restyled/branded for Blackhill.
**Acceptance:** public visitor browses postings, signs in, completes profile, applies, and receives confirmation; staff sees the
application with an immutable snapshot; withdraw retained.
**Tests (heavy):** duplicate application handling (existing unique key and double-submit); snapshot immutability after later profile edits;
form-version preservation; client/facility name never in public HTML/JSON/URL unless the posting discloses it; submitted documents
immutable; staff scope/capability matrix for applicants; withdrawn retained (subject to D-20 wording).
**Milestone M1 (first deliverable): S0+S1+S2+S3 = US candidate authentication, clinician profile, job applications.** It can be demoed
to Ahmad on synthetic data. It is **not** production-ready (no staff MFA, no backups, PMS deployment files).

## S4 — Facilities, contacts, job orders, staff-created candidates

**Depends on:** S3; D-07, D-09, D-10, D-11.
**Scope:** `Facility` (timezone), `ClientContact`, `JobOrder` (minimal fields, perm + contract), posting→job-order link in UI, scope
extension (`StaffScope.facilities/job_orders`), new capabilities, staff "add candidate" via the single service with duplicate
warnings, recruiter candidate search over structured columns (Scenario B).
**Tests (heavy):** capability × scope × object matrix with 404s on crafted URLs and fail-closed with no profile; contact PII hidden without
`client_contact.view`; staff-created candidate: no account, duplicate warnings (phone, name+DOB, credential identifier) never block and never merge;
creating a candidate by a second path is impossible (single-service test); job-order `employment_type=CONTRACT ⇒ duration` constraint;
reference uniqueness per client; rates absent from all reads/exports.

## S5 — Consideration, representation consent, activities

**Depends on:** S4; D-08, D-15 (Consideration), D-16.
**Scope:** `Consideration` + events + transition service; "consider this applicant" from an Application; `RepresentationConsent`
(record/revoke); `Activity` (notes, calls, follow-ups) with anchors; recruiter workspace list/detail.
**Tests (heavy):** Scenario H at consideration level (real threads: exactly one open row; loser redirected); open-pair uniqueness
allows the same candidate across different job orders (Scenario C); event table immutability; consent revoke effect on future
submissions; activity CHECK (≥1 anchor); scope on every anchor; ownership-reassignment audit.

## S6 — Client submission

**Depends on:** S5; D-13, D-14, D-15 (submission), D-17.
**Scope:** `ClientSubmission`, immutable `SubmissionPacket` (+ rendered artifact with sha256), transition service and events, duplicate
exception flow, packet open/download with `SensitiveAccessEvent` (extend types/CHECK), facility feedback recording, consent-required gate,
credential level shown *as of date* in packet.
**Tests (heavy):** Scenarios A, B, C, D, G, H end to end; one open submission per consideration (service, UI, DB, double-click, two tabs);
exception requires different approver (or the D-13 fallback is flagged); packet unchanged after profile/credential edits; packet shows no field
outside `disclosed_fields`; revoked consent blocks new submissions and flags open ones; status transitions only via service (direct
`update()` refused); lock-order deadlock test with real threads; export capability separation; candidate-facing views never expose submissions;
audit event written **before** bytes are served and failure to write blocks access.
**Milestone M2 (first operational release candidate):** S4–S6 + the production blockers listed below.

### Production blockers that gate M2 (not slices of domain work)
Staff TOTP MFA (D-24) · backups + tested restore · retention decision (D-20) · HSTS ≥ 1 year, CSP · production deployment files rewritten
(D-31) · AV scanning hook decision · uploads pen-test pass · third-party review of authz matrix.

## S7 — Verification workflow and expiry alerts

**Depends on:** S6 (not strictly); D-19.
**Scope:** staff verification recording UI (`credential.verify`), discrepancy handling, expiry dashboard, `send_credential_expiry_alerts`
management command (idempotent per credential/window) run by host cron.
**Tests (heavy):** idempotency (run twice → one alert per window); window boundaries in local timezone; renewal resets alert state; verification
immutability; discrepancy blocks L3; alerts respect `do_not_contact`; command fails safe on partial SMTP outage.

## S8 — Merge and claim hardening

**Scope:** `candidate.merge` capability, merge preview, blocking rules (same Application attempt / open Consideration), `CandidateMergeEvent`, tombstone redirect.
**Tests (heavy):** every FK to Candidate is in `MERGE_RELATIONS` (fails when a new FK is added unregistered); merge is atomic (forced mid-merge failure leaves both intact);
snapshots keep original `candidate_reference`; loser account cannot sign in; blocked merges explain why; audit and reversal-by-record (not undo).

## Release 2 and later (design only; do not start without approval)

Offers re-parented to Consideration with schema-versioned terms (D-25) → placements/terms (D-26) → requirement sets + `ClearanceCheck` →
e-signature adapter → provider-based credential verification → onboarding/SSN module (D-18) → dialer/SMS → client portal → reporting.

---

## What the first deliverable (M1) must demonstrate to Ahmad

1. A clinician on a phone signs in with email, builds a clinical profile, adds a license with a photo of the card.
2. The same clinician applies to a public posting; staff sees an immutable snapshot and a license shown as "Document attached", not "Verified".
3. Repeating sign-in with `Foo@Example.com ` lands in the same account; a typo address gives the same-looking response as a real one.
4. Staff and candidate sessions are visibly separate (two hostnames).
5. No screen anywhere mentions CNIC, Urdu, PKR, OGDCL or PMS.

## Cross-cutting test inventory (applies to every slice that touches the area)

| Area | Always test |
|---|---|
| Authorization | capability present/absent × scope in/out × object exists/doesn't; 404 vs 403 consistency; direct POST without UI |
| Ownership | candidate A vs B; staff scope A vs B |
| Concurrency | real-thread tests for any partial-unique or lock-order rule |
| Immutability | single save, bulk `update()`, `delete()`, queryset delete |
| Snapshots | later edits do not change frozen records |
| Documents | version ownership, private serving, audit-before-bytes, fail-closed |
| Sensitive data | masked by default, reveal audited, absent from logs/exports/packets unless intended |
| Time | local-timezone date boundaries; DST; expiry-day inclusivity |
