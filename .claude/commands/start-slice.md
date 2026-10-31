---
description: Start work on a slice — load only what it needs and plan before coding
argument-hint: <slice-id e.g. BH-AUTH-01>
---
We are starting slice $ARGUMENTS.

1. Read docs/STATE.md, the FROZEN rows of docs/DECISIONS.md, and docs/slices/$ARGUMENTS*.md. Read nothing else yet.
2. If the slice status is not READY, or it depends on any decision that is not FROZEN, stop and list exactly what is missing. Do not plan around missing decisions.
3. Otherwise read only the code files the slice touches, then produce a plan: files to create/change, models and constraints, migrations, and the test names that encode each acceptance criterion.
4. Stop and wait for my approval of the plan. Do not write code yet.
