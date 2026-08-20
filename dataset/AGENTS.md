# Best Reviewer dataset instructions

When working on this dataset, read these files first:

1. `PROJECT_MEMORY.md`
2. `DATASET_SPEC.md`
3. The target sample's `manifest.json` and numbered artifacts, if it exists

Treat `PROJECT_MEMORY.md` as the persistent record of user decisions.
Append new user corrections to its decision log instead of relying on chat
history. Do not silently change a locked decision.

Dataset samples are ordered interaction records, not isolated prompt/answer
pairs. Keep event order, actor perspective, repository base commit, tool
observations, PR artifacts, and the final report traceable.

Keep this dataset directory outside the sibling `BrainWash/` repository.
`BrainWash/` is the code work product; dataset records and project-memory files
belong here.

Do not store hidden chain-of-thought. Store concise decision summaries,
evidence, alternatives considered, and observable tool calls/results.
