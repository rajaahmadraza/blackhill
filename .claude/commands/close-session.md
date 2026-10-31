---
description: End the session cleanly — verify, record state, commit, build the review pack
---
Close this session:

1. Run the slice's narrow tests via the test-runner subagent. Report the exact command and counts.
2. Update docs/STATE.md: snapshot (branch, short commit from git, current/next slice), "Last session" (what was done, tests command → result, what is carried over), Blockers, and "Spotted, not done". Facts only; any number must come from a command you ran now.
3. If any question came up that needs my decision, add it as an OPEN row in docs/DECISIONS.md. Never write FROZEN.
4. Show me `git status --short` and a one-paragraph summary, then commit with message "<SLICE-ID>: <summary>" once I confirm.
5. Run `bash tools/sync_pack.sh` and tell me the two output paths so I can upload them to ChatGPT.
