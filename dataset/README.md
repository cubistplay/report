# Best Reviewer Dataset

This directory contains sequential, repository-grounded examples for the
**Best Reviewer** project. The source project is BRAINWASH, and the initial
repository baseline is the current `main` commit recorded in each sample.

The dataset has two tracks:

- `implement/`: an AI acts as the code author. It uses TDD, removes code smells,
  refactors, explains design choices, opens a PR, and participates in review.
- `review/`: an AI acts as the reviewer. It finds material problems, shares
  Clean Code and TDD knowledge, and conducts constructive peer-level discussion.

The target is 28 PR samples across the two tracks. The final Implement/Review
split has not yet been specified. The first ten are produced incrementally:

1. Create and calibrate `implement-01`.
2. Complete `implement-02` through `implement-05`.
3. Complete `review-01` through `review-05`.
4. Decide the remaining 18-sample allocation after this initial batch, then
   expand one natural, roadmap-grounded PR at a time until the experiment
   contains 28 accepted samples.

Read [PROJECT_MEMORY.md](PROJECT_MEMORY.md) for user decisions and
[DATASET_SPEC.md](DATASET_SPEC.md) for the artifact contract.
