# Blackhill — Healthcare Staffing Platform

US healthcare staffing operating platform for Blackhill International: clinicians, facilities,
job orders, applications, client submissions, credentials, offers, placements.
Completely isolated from PMS ATS. Never read, copy from, or connect to ~/pms-ats unless the
current slice spec explicitly names a file to port.

## Source of truth (in this order)
1. docs/DECISIONS.md — only rows marked FROZEN are rules. PROPOSED/OPEN are not rules.
2. docs/architecture/ — approved documents only. Anything in docs/drafts/ is NOT approved.
3. docs/slices/<current slice>.md — scope for this session. Out-of-scope = do not build.
4. docs/STATE.md — what exists right now.
If a request conflicts with a FROZEN decision, stop and say so. Do not resolve it yourself.

## Commands
- App:        docker compose up
- Tests:      python manage.py test <narrowest label>   (never the full suite unless asked)
- Full suite: only when asked, or before a release tag
- Lint:       ruff check . && ruff format .
- Sync pack:  bash tools/sync_pack.sh            (writes docs/handoff/SYNC_PACK.md)

## Hard rules
- Candidate.id (UUID) is the person. Email is a login method, never the identity.
- Application, ClientSubmission, Offer, Placement are separate records with separate status
  sets and history tables. Never one shared status field.
- An uploaded document never makes a credential "verified". Verification is its own record.
- Submitted snapshots and issued documents are immutable. Corrections create new versions.
- No full SSN anywhere. SSN-last-4 only after DECISIONS.md marks it FROZEN.
- Staff permission checks go through the capability + data-scope service, never ad-hoc.
- No real candidate data in the repo, fixtures, tests or screenshots.

## Session discipline
- One slice (or one sub-task of a slice) per session.
- Plan first (plan mode). Write tests that encode acceptance criteria before implementation.
- Do NOT create new docs, refactor unrelated code, rename things, or "clean up" unless the
  slice spec says so. List anything you noticed under "Spotted, not done" in STATE.md instead.
- Never state a number (test count, commit, line count) you did not get from a command
  in this session. Say "not checked" instead.
- After two failed attempts at the same bug: stop, summarise what you learned, ask me.

## Definition of done for every session
1. Narrow tests for the slice pass (paste the command + result summary).
2. docs/STATE.md updated (use /close-session).
3. Committed with a message that names the slice ID, e.g. "BH-AUTH-01: email OTP login".
4. bash tools/sync_pack.sh run, so ChatGPT can review.

## Compact instructions
When compacting keep: current slice ID, files modified, failing test names, and any
decision that was raised but not yet recorded in DECISIONS.md.
