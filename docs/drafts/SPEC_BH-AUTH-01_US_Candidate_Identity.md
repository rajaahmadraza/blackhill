# BH-AUTH-01 — US Candidate Identity

Status: **DRAFT for approval. Not implemented.** Nothing here is Blackhill-approved business behaviour until the
"Decisions required" section is answered. Evidence references (`path:line`) are to the inherited PMS-derived code at
the recorded upstream commit and may move.

## 1. Goal

A candidate is identified by a **verified email address** and a **persistent Candidate UUID**. CNIC leaves
Blackhill's required identity flow. Duplicate candidates are prevented explicitly, account recovery is a defined
flow, profile completeness reflects a US healthcare profile, and every historical Application snapshot is preserved
untouched.

## 2. Current behaviour being replaced (inherited)

| Concern | Inherited fact |
|---|---|
| Identity key | `Candidate.normalized_cnic` is `unique=True` (`apps/candidate_portal/models.py:24`); ~30 lookup sites |
| Sign-in | One form takes CNIC + Pakistan mobile + email (`forms.py:25-79`); mobile validator accepts only `+923…` (`identity.py:44-57`) |
| Account creation | After the first correct OTP, `_candidate_after_correct_otp` creates `Candidate` + `CandidateAccount` (`services.py:499-545`) |
| Returning user | Refused if the email or mobile differs from the stored one (`services.py:231-246`), a dead end |
| Email | `Candidate.verified_email` is a plain, **non-unique** `EmailField` (`models.py:33`) |
| OTP / throttles | Keyed on CNIC, mobile, email, client (`services.py:79-340`; `OTPChallenge` `models.py:237-279`) |
| Snapshot | `ApplicationSubmissionSnapshot.submitted_cnic`, `submitted_mobile` (`applications/models.py:512-514`) |
| Completeness | Hard-requires father/husband name, domicile, CNIC, +92 mobile (`profile_completeness.py:24-30,129`) |

Kept as-is: the UUID primary key on `Candidate` (no FK is rewritten), the separate candidate/staff session keys, the
email-OTP delivery mechanism, `CandidateSecurityEvent`.

## 3. Scope

1. **Email identity.** One `Candidate` per normalized email. Normalization: trim, Unicode NFKC, lower-case the whole
   address (no provider-specific folding such as Gmail dots or `+tags`: those would merge distinct people).
   Enforced by a database unique constraint on the lower-cased value (functional `UniqueConstraint`), not only in forms.
2. **Persistent UUID.** `Candidate.id` stays the immutable internal identity. It is never derived from, or changed by,
   an email change. Email is an attribute that can change through the flow in §5.
3. **Sign-in.** Email → OTP → session. Create-or-reuse happens only after a correct OTP (preserves the existing
   "identity is created after verification" rule). The response to an unknown vs known address is identical
   ("If this address can receive a code, we sent one") to prevent account enumeration.
4. **Remove CNIC from the required flow.** No form, OTP, throttle, completeness rule or snapshot *requires* it.
   The column and historical data are **not dropped in this slice** (expand/contract): `normalized_cnic` becomes
   nullable, with the unique constraint kept only for non-null values. Final removal is a later, separate migration
   once nothing reads it.
5. **Phone.** Optional on first sign-in, E.164 format, **not verified** (no SMS provider). International numbers
   accepted. Replaces `normalize_mobile`'s Pakistan-only rule.
6. **US profile completeness.** One centralized rule (extend `profile_completeness.py`, keep it the single
   definition used by Databank searchability, as the inherited amendment requires). Proposed required set, pending
   decision D3: full legal name, verified email, phone, date of birth, US address (street, city, state, ZIP),
   primary profession/specialty. Father/husband name and domicile become irrelevant to completeness.
7. **Ownership and security.** Every candidate-facing object is resolved through the session's candidate (existing
   ownership tests stay green and are extended). Session ID is rotated on login. OTP codes stored hashed with
   attempt/send throttles keyed on email and client IP. No email address in URLs or logs.
8. **Historical snapshots preserved.** No rewrite of any existing snapshot, `EligibilityAttempt` or Application.
   `submitted_cnic`/`submitted_mobile` become nullable and stay populated on old rows. New snapshots add
   `submitted_email` and `submitted_phone` and a `snapshot_schema_version`. Snapshot JSON for new rows carries the
   US address.

## 4. Duplicate prevention

- **Hard rule:** one Candidate per normalized email (DB constraint). Staff-created Candidates (BH-STAFF-01, intake)
  must call the same `get_or_create_candidate_by_email` service; no second creation path.
- **Soft signals, no auto-merge:** on staff create, warn on same phone, or same name + date of birth. A warning
  never blocks creation; it shows the existing record for the staff member to reuse.
- **Merge is out of scope** and would be its own capability-gated, audited slice (it touches Applications, documents
  and talent-pool membership).

## 5. Account recovery and email change

| Case | Flow |
|---|---|
| Lost session, still has mailbox | Normal sign-in. |
| Wants to change email, still signed in | Signed-in candidate enters new email; OTP to the **new** address; on success email is updated; a notice goes to the old address; `CandidateSecurityEvent` written. |
| Lost mailbox access | Staff-assisted change: requires a new capability `candidate.change_login_email` (sensitive), a reason, and an identity-verification note; OTP is still sent to the new address before the switch; event written. No self-service bypass. |
| New email collides with another Candidate | Refused with a neutral message; staff see the collision and decide (merge slice later). |

## 6. Data/migration notes

- Blackhill has no production data. **Recommended:** keep the inherited migration history and add forward
  migrations, so upstream fixes can still be diffed against the recorded base commit. (Alternative: squash and
  reset, simpler but loses that property. Decision D5.)
- Candidate rows from the inherited UAT/seed tooling are not carried.
- `OTPChallenge`: add `normalized_email` (indexed), make `normalized_cnic`/mobile nullable; stale unexpired challenges
  are invalidated by the migration.

## 7. Acceptance criteria

1. A new email completes OTP and gets exactly one Candidate with a UUID; the same email later (any letter case,
   surrounding spaces) signs into that same Candidate.
2. Two concurrent first sign-ins for one email produce one Candidate (constraint + retry test).
3. Known and unknown emails get indistinguishable responses and equivalent timing class.
4. No candidate screen asks for CNIC, father/husband name or domicile; a US-only profile reaches "complete" and
   becomes Databank-searchable; an incomplete one does not.
5. Candidate A cannot read or modify Candidate B's profile, documents or Applications (URL tampering returns 404).
6. Email change requires OTP at the new address, notifies the old, and writes a security event; a colliding address is
   refused.
7. A pre-existing snapshot, EligibilityAttempt and Application are byte-identical after the migration.
8. The staff and candidate security domains remain separate (a candidate session grants no staff access and vice
   versa), including the shared-cookie behaviour noted in the audit (see D6).
9. No Pakistan-only validator is reachable from the Blackhill sign-in path.

## 8. Test plan (risk-based, per the inherited fast-lane rule)

Heavy: email normalization and constraint races, OTP throttle/expiry/attempts, enumeration parity, ownership, email
change and recovery, snapshot immutability across the migration, completeness gating of the Databank, session
rotation. Light: copy and layout. Reuse and adapt `test_candidate_identity.py` (36) and `test_email_otp.py` (75).

## 9. Out of scope

SSN (separate sensitive-identity spec), passwords/MFA/social login, SMS verification, candidate merge, background
checks, document model changes, any submission logic.

## 10. Decisions required before build

- **D1** Is email the *sole* identity? (Audit open question 1.) Is a verified phone ever required?
- **D2** Shared-mailbox policy: may two people legitimately share one email (e.g. a family inbox)? Default here: no.
- **D3** The exact US profile-completeness field list, and whether work-authorization status is collected (legal review).
- **D4** Candidate self-registration open to the public, or invite-only at launch?
- **D5** Keep inherited migration history (recommended) or reset.
- **D6** Move the candidate session to its own cookie name/store, separate from staff (the shared cookie noted in the audit
  is INFERRED, to be confirmed in code before deciding).
- **D7** Data retention and deletion of candidate records (conflicts with the inherited "withdrawn Applications are
  retained" invariant under some US state privacy laws).
