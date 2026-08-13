# Repository maintenance instructions

This repository is an append-only engineering solution log for Dataverse and pyDataverse work.

After completing a relevant issue or pull-request task, ask the user whether the work should be documented here. Only update and push after the user confirms.

For every confirmed update:

1. Append a new numbered solution to `Pydataverse-solution.md`.
2. Follow `SOLUTION_TEMPLATE.md` and update the table of contents.
3. Explain both the technical failure and what customers actually experience.
4. Identify affected customer roles, blocked workflows, data/compliance risks, workaround cost, scope, and severity where evidence supports them.
5. Include reproduction, investigation, operations, obstacles, communication, final code solution, validation, post-fix customer experience, and reusable lessons.
6. Clearly distinguish implemented, tested, CI-passed, approved, merged, and released. Never describe an open PR as merged or released.
7. Add small screenshots or diagrams under `assets/` when useful.
8. Never commit credentials, API tokens, passwords, cookies, private customer data, machine-specific private paths, or oversized raw artifacts.
9. Use evidence-based wording. Do not invent customer impact; label risks and limitations accurately.
10. Link public issues, PRs, commits, CI runs, and releases whenever available.
