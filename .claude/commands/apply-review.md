---
description: Apply accepted ChatGPT review findings — only the ones Ahmad accepted
argument-hint: <paste accepted findings>
---
These review findings were ACCEPTED by me:

$ARGUMENTS

Rules:
- Fix only these. Do not act on anything else in the review.
- If a finding would change scope or a FROZEN decision, do not implement it — add it as a PROPOSED row in docs/DECISIONS.md and tell me.
- If you believe a finding is wrong, say so with file:line evidence instead of complying.
- Add or adjust tests so each fix is covered, run them via test-runner, then /close-session.
