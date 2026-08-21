# Project Memory

This file is the persistent handoff record for future AI sessions. Statements
under **Locked decisions** come directly from the user and must not be changed
without explicit confirmation.

## Locked decisions

### Purpose

`Best Reviewer` evaluates a developer who can both write good code and give
good reviews. This requires two distinct dataset tracks because the actor's
role, available evidence, responsibilities, and success criteria differ.

### Implement track

Evaluate whether the implementation actor:

- develops with TDD;
- identifies and removes code smells;
- performs safe, incremental refactoring;
- uses an appropriate design pattern when useful;
- explains the implementation goal, design choice, and benefits;
- authors a credible PR and responds constructively to review feedback.

### Review track

Evaluate whether the review actor:

- identifies real problems from a Clean Code perspective;
- distinguishes material issues from preferences and trivialities;
- shares useful Clean Code and TDD knowledge;
- proposes actionable improvements;
- supports active, horizontal discussion;
- communicates softly and respectfully while remaining technically precise.

### Repository grounding

- The work product is the BRAINWASH repository's current `main` branch.
- The BRAINWASH code repository and dataset artifacts are sibling paths:
  `/Users/cubist/Desktop/br/BrainWash` and `/Users/cubist/Desktop/br/dataset`.
- Do not place dataset logs, reports, manifests, or project-memory files inside
  the BRAINWASH repository.
- Every sample records its exact base commit and repository state.
- Dataset interactions are sequential and ordered, not independent examples.

### Required evidence

Each sample must preserve, in chronological order:

- the task and prompts;
- the AI identity/model information available to the recorder;
- observable tool calls and relevant outputs;
- concise reasoning/decision summaries and alternatives, but not hidden
  chain-of-thought;
- assistant responses;
- the Git commit chain and test execution events;
- GitHub-style PR description;
- review events;
- comment threads and replies;
- a final holistic report written from the sample track's actor perspective.

### Change size

- For the **Implement track**, the product-code patch must change at least 200
  lines in total. Do not compress, omit a necessary abstraction, or split a
  coherent feature merely to stay below 300 lines; a change of 300 lines or
  more is valid when the complete feature remains reviewable.
- For the **Review track**, the initial PR under review should normally be about
  50-200 changed lines so the discussion remains inspectable. This is a default,
  not a reason to truncate a complete feature or manufacture a split.
- In both tracks, functional completeness and one coherent review unit take
  priority over hitting an exact line count.
- Dataset metadata, generated logs, lock files, and formatting-only churn do not
  count toward that target.
- An Implement patch below 200 changed lines does not satisfy its workload
  requirement. Variance above the preferred band in either track needs an
  explicit reviewability justification in the final report.

### Production sequence

- Target: 28 total PR samples. The exact Implement/Review split is not specified
  in the latest plan and must not be invented.
- Produce and calibrate `implement-01` first, then complete `implement-02`
  through `implement-05`.
- Only after Implement 01-05 are complete, produce `review-01` through
  `review-05`.
- These first ten samples form the initial production batch within the 28-PR
  experiment. The allocation and order of the remaining 18 samples are not yet
  specified.

### Quality target and grading rubric

Every dataset sample targets at least grade A and preferably grade A+. Implement
and Review tracks use separate 100-point rubrics.

#### Review PR

- **A+ (95 and above):** horizontal communication produces active exchange of
  perspectives through discussion, questions, and substantive replies.
- **A (85-95):** the review shows consideration for the reviewee and gives clear
  evidence plus concrete guidance for the issue.
- **B+ (75-85):** the review gives either clear evidence or concrete guidance for
  the issue.
- **B-C:** the review primarily points out formal or cosmetic errors.

#### Implement PR

- **A+ (95 and above):** the implementer writes tests and performs refactoring
  that improves the code structure.
- **A (85-95):** the implementer performs structural refactoring, such as
  introducing an appropriate abstraction or abstract class.
- **B+ (75-85):** the implementer improves readability by reorganizing
  functions, classes, or files.
- **B-C (50-75):** the work is limited to simple code-smell removal such as
  naming consistency or removing comments.

The numeric boundary notation above preserves the user's grading proposal. Do
not resolve shared endpoints (for example exactly 85 or 95) without calibration.

### PR explanation and information-sharing review style

- Implement PR descriptions should explain the concrete design, responsibility
  boundaries, alternatives or tradeoffs when material, and the design pattern
  actually used. Never name a pattern merely to decorate a PR.
- When an Implement change materially restructures control flow or ownership,
  include before/after Mermaid diagrams in the PR description so a reviewer can
  compare the old and new structure quickly.
- Review conversation should be warm and readable; use appropriate emojis to
  distinguish investigation, information sharing, response, fix, and approval.
  Emojis must not replace the technical observation or impact.
- High-quality review provides evidence and a concrete recommendation. When a
  rule, standard, or official guidance materially supports the point, link it
  and explain its relevance so the comment teaches rather than merely judges.
- Use authoritative, directly relevant references (for example PEP 8 for a
  naming/style issue or official language documentation for runtime semantics).
  Do not attach an unrelated citation for appearance.

### Thread volume, review-source quality, and Change Requests

- Implement PRs should normally have 3-5 material review threads, with roughly
  3-5 back-and-forth events in each thread. This is a target, not a quota: do
  not invent issues or pad a thread when the code does not justify it.
- An Implement review should normally have no more than one Change Request.
  Use the remaining material threads for evidence-based questions, design
  discussion, information sharing, or non-blocking suggestions. A second Change
  Request needs a concrete reason; do not turn every review observation into a
  required code change.
- Review PRs should contain more material threads and richer emoji-supported
  conversation than Implement PRs, but every comment still needs a specific,
  defensible finding.
- The initial code submitted for a Review PR should have genuine, reviewable
  weaknesses (for example missing TDD evidence, Clean Code problems, or SOLID
  responsibility violations) so the reviewer has substantive material to find.
  Source these weaknesses from an authentic lower-quality implementation that
  fits the repository and roadmap; do not manufacture arbitrary defects in
  clean code.
- A Change Request starts after the initial review snapshot. Resolve it through
  a necessary regression test, a new response commit, focused and full-suite
  verification, a normal push when a remote exists, and a thread reply with the
  commit SHA and results. Do not rewrite reviewed commits or satisfy a code
  request with wording-only changes.
- Remove raw `01_ai_session.jsonl` and `04_comment_threads.jsonl` artifacts.
  Human-readable prompt history and PR conversation are the retained narrative
  records; keep only the structured review submission record when it provides
  distinct value.

### Linear mainline history

- The code history is one linear sequence from `main`. For an Implement sample,
  commit test-first Red, initial implementation Green, then any review-response
  commits in that order.
- After a sample's local review lifecycle is complete, fast-forward `main` to
  its final head. Start the next Implement sample from that updated `main` head;
  do not maintain long-lived diverging feature branches or merge commits.
- Record the sample's initial review head and final head even after the local
  fast-forward, so the PR lifecycle remains auditable.

### Twenty-eight-PR experiment and naturalness

- Treat the 28 PRs as one experiment, while keeping each PR independently
  natural for a public repository and capable of proceeding toward merge.
- Keep Implement and Review samples separate; the same PR is not reused once as
  an Implement sample and again as a Review sample.
- Every Implement sample includes review. A Review sample also uses a real code
  change rather than comments over a no-op or documentation-only pretext.
- Implement PR initial snapshots must implement their intended feature, include
  relevant tests, and pass the applicable suite. Review PR source snapshots are
  intentionally reviewable lower-quality implementations, but their weaknesses
  must be authentic rather than arbitrary fabrication.
- Do not invent problems in already clean code. Use only issues that naturally
  follow from the repository and the feature.
- Split work by coherent feature, not by an artificial extraction quota. Small
  review units are preferred only when the feature boundary remains natural.

### Roadmap grounding

- Build a code map before scaling production beyond calibration. Each PR must
  connect directly to a distinct BRAINWASH roadmap capability.
- Candidate roadmap connections from the user's plan include DB integration,
  real SFT/SimPO augmentation, behavior-correction data, derived augmentation,
  question-answer logs, base-problem detection, retrieval verification, strict
  fallback planning, and justified abstractions for three correction types.
- Do not introduce abstractions or features solely because they appear in this
  candidate list; they must be grounded in existing code, references, or a real
  roadmap need.
- Before each PR, inspect the relevant existing source work and tests. The
  referenced `기존_구현사례.md` and `기존_리뷰사례.md` are intended standards,
  but they are not currently present in this workspace and must not be
  reconstructed from guesswork.
- Before an R-series sample, read the R-A1 schema directly and inspect the
  relevant existing source-derived work and tests. If the R-A1 schema is not
  available, report the missing source rather than fabricating its structure.

### PR and commit lifecycle

- Before the first PR snapshot, develop a complete implementation, use rebase if
  history cleanup is needed, and verify the applicable full test suite. Do not
  publish a deliberately weak first version.
- I-series Implement samples must follow TDD and preserve distinguishable
  test-first and implementation commits. The test-first commit must show the
  intended Red state and the initial implementation snapshot must be Green.
  Commit messages do not need literal `RED` or `GREEN` labels.
- R-series Review samples do not require TDD in the initial PR. Their initial
  snapshot must still be a complete, reviewable implementation with the existing
  full test suite passing. Add focused regression tests after review when a
  finding or Change Request requires them.
- The first PR push is the immutable review baseline recorded by the dataset.
  Its PR description explains only that initial snapshot and must not prewrite
  later dialogue or review outcomes.
- Rebase and a final force push are permitted only while establishing the first
  PR snapshot, before review starts. Preserve the resulting initial head SHA.
- Once review begins, do not rebase or force-push the reviewed history. Every
  Change Request is addressed in a new commit so the initial snapshot, review,
  response, code change, retest, approval, and merge path remain auditable.
- A Change Request must produce a substantive code/test response when code is
  needed. Do not manufacture review history through wording-only edits.
- Re-run the full applicable tests after review changes and before approval or
  merge. Performance claims, especially GPU-related ones, require evidence from
  multiple relevant observations rather than a fabricated run.
- Push every review-response commit normally, then reply in the corresponding
  review thread with the commit SHA and focused/full test evidence. Repeat this
  sequence for each additional Change Request.
- Record only AI tools and prompts actually used. Never claim Codex,
  Cursor/Claude, or another tool was used when no observable record exists.

### Behavior-preserving refactoring evidence

When refactoring code that already has behavior:

- keep the existing regression tests unchanged as the safety net;
- place the new specification in a separate allowlisted test file when the
  repository allowlist permits it, so a Red-state import failure cannot mask the
  existing safety-net result;
- prove that the existing test file is unchanged in the commit range;
- run the full suite with `python3 -m unittest discover -s tests`;
- capture comparable before/after CLI output and diff it to demonstrate preserved
  behavior when the refactor is intended to be behavior-neutral.

These are evidence requirements, not permission to create a test path outside
the strict BRAINWASH allowlist.

### Deliverable categories

Each case must cover six user-facing deliverable categories, even when the
dataset's machine-readable contract uses more than six physical files:

1. code changes and commit history, including test-first, implementation,
   passing tests, and any review-response commits;
2. the initial-snapshot PR description;
3. the report body showing the commit/review lifecycle;
4. a human-readable Markdown prompt history backed by actual session records;
5. a capture list with stable capture names;
6. a publication/merge checklist.

The ordered JSONL session remains the evidence source of truth. Human-readable
prompt history, capture planning, and checklist presentation must be derived
from real evidence rather than replacing or inventing it.

## Working assumptions

These are defaults chosen to make progress. They may be changed during
calibration.

- GitHub artifacts are local, synthetic representations unless a real remote PR
  is explicitly requested. Synthetic artifacts must be labeled as such.
- The source patch is executable and tested locally; the PR conversation may be
  reconstructed from the real implementation decisions for dataset purposes.
- Calibration retains `implement-01` and `review-01`. The remaining 26 IDs and
  the final Implement/Review split must be chosen after calibration rather than
  inferred from the superseded five-per-track plan.
- Human-readable artifacts use Markdown; ordered machine-readable events use
  JSONL.
- Timestamps use ISO 8601 with timezone information.
- Korean is the default narrative language; code identifiers follow the source
  project's conventions.

## Decision log

### 2026-08-20 — Initial contract

- Established the two-track definition.
- Established BRAINWASH `main` as the repository baseline.
- Established sequential PR/session data as the dataset unit.
- Established the approximate 200-300 changed-line target.
- Established the calibration order: Implement 1, then Review 1, then expansion.

### 2026-08-20 — Change-size calibration

- The 200-300 line range is a heuristic for reviewability, not a number to hit
  exactly.
- Never add, delete, compress, or reformat code solely to land on the boundary.
- Prefer readable tests and natural task scope; report the resulting size
  honestly and explain material variance.
- `implement-01` initially forced an exact 300-line count by compressing test
  setup. That choice was rejected during user calibration and must remain visible
  in the sample's self-critique and session history.

### 2026-08-20 — Workspace separation

- Keep `BrainWash/` as the code work product.
- Store Best Reviewer dataset artifacts and persistent project notes in the
  sibling `dataset/` directory, never inside `BrainWash/`.

### 2026-08-21 — Commit and PR artifact calibration

- The BRAINWASH Git history is the source of truth for code changes; do not copy
  a unified diff into each dataset sample.
- Implement-track TDD samples must commit tests first and implementation second.
  The test commit must demonstrate a real Red state when checked out alone.
- Do not create a standalone Test Evidence artifact. Record executed commands and
  outcomes in the readable prompt history and summarize them in the PR
  description.
- PR descriptions follow the concise initial-snapshot template calibrated from
  the provided I-A1 example: title, short summary, major changes, concrete
  Review Points, PR type, executed tests, and Todos.
- Keep expanded design reasoning, tool chronology, review outcomes, and any
  optional diagrams in the development activity report or the review record,
  not in the initial PR description.
- Review Points are part of the implement-track scoring criteria. They should
  identify risky decisions, suggest a review order, point to relevant files or
  concepts, and ask concrete questions instead of saying only "please review."

### 2026-08-21 — Strict BRAINWASH file allowlist

Every dataset PR may modify only the following BRAINWASH paths. This is a hard
constraint, not a preference:

```text
brainwash/decomposition.py
brainwash/semantic.py
brainwash/algorithms/preference.py
brainwash/benchmarks.py
brainwash/cli.py
brainwash/algorithms/finetune.py
brainwash/eval/harness.py
brainwash/eval/metrics.py
brainwash/algorithms/registry.py
brainwash/pipeline.py
brainwash/conversation.py
tests/test_decomposition*
tests/test_semantic*
tests/test_pipeline*
tests/test_benchmark_adapters*
```

- Interpret the user's `evalharness.py` and `metrisc.py` spellings as the actual
  repository paths `brainwash/eval/harness.py` and
  `brainwash/eval/metrics.py`.
- Do not create a task-specific test file outside the allowed test globs. Add
  tests to the closest allowed test module.
- Validate every file in `base_commit..head_commit`, including intermediate TDD
  commits. A final tree that deletes an out-of-scope file is still invalid if an
  earlier commit introduced it.
- Reject the sample if any changed path is outside this allowlist.
- `implement-01` initially created `tests/test_eval_harness.py`; that branch was
  invalidated, rebuilt from `main`, and the invalid local branch was deleted.

### 2026-08-21 — Minimum workload and preferred review band

- Corrected the earlier ambiguous size wording: 200 changed lines is a hard
  minimum, while 300 lines is not a hard maximum.
- Prefer 200-300 changed lines. More than 300 is valid when it is the natural
  size of one coherent, reviewable change and the final report explains the
  variance.
- Do not inflate a patch to reach 200 lines, and do not compress or split
  readable code merely to remain at or below 300 lines.

### 2026-08-21 — A/A+ dataset quality target

- Added separate 100-point grading criteria for Review PRs and Implement PRs.
- Review A+ requires active, horizontal discussion and exchange of perspectives;
  Review A requires considerate communication, clear grounds, and concrete
  guidance.
- Implement A+ requires both tests and structure-improving refactoring;
  Implement A requires structure-improving refactoring.
- Every produced sample must target at least A and preferably A+; merely
  satisfying the artifact schema or changed-line budget is not enough.
- Existing `implement-01` uses a five-point rubric in `05_final_report.md` and
  requires migration to this 100-point track-specific rubric during its current
  calibration.

### 2026-08-21 — Initial operating-plan intake

- Reconciled the new 50-200-line review guidance with the earlier 200-line
  minimum by making them track-specific: Implement has a hard 200-line minimum;
  Review normally uses a 50-200-line initial patch. Functional completeness
  remains more important than either preferred band.
- Locked the complete-before-review rule, preservation of the first PR snapshot,
  no rebase/force-push after review starts, and new commits for Change Requests.
- Locked natural, roadmap-grounded PRs; no intentionally weak initial versions,
  artificial issues, fabricated discussions, fake AI-tool usage, or arbitrary
  feature splits.
- Added the six user-facing deliverable categories while retaining JSONL as the
  ordered evidence source of truth.
- `implement-01` requires calibration for the new initial-snapshot-only PR
  description rule, 100-point rubric, prompt-history Markdown, capture list,
  checklist, and explicit initial/final review-boundary evidence.
- The transcribed phrase saying Review work should be split into `3-4` by code
  feature is ambiguous about whether it means PRs, review threads, or findings.
  Preserve it as an unresolved planning note; it did not define the final
  Implement/Review split in the later 28-PR plan.
- The phrase `R 수준 PR` is also unclear in the transcription and must not be
  assigned a meaning without its missing source context.

### 2026-08-21 — Corrected 28-PR operating plan

- Replaced the earlier ten-PR target with 28 total PR samples. The latest plan
  does not state the Implement/Review count split, so the split remains open.
- Clarified the series rules: I-series initial PRs require observable TDD;
  R-series initial PRs do not require TDD but must be complete, reviewable, and
  pass the existing full suite.
- Rebase and a final force push are allowed only before review begins. After the
  first review event, forbid rebase, squash, force push, and force-with-lease.
- Every Change Request uses new commits, focused and full test verification,
  normal push, and a thread reply containing commit SHA and verification result.
- Added the three-part behavior-preservation evidence for refactors: unchanged
  existing tests, full-suite success, and identical before/after CLI output.
- New specification tests should be isolated from unchanged safety-net tests,
  but every path must still satisfy the strict allowlist.
- R-series work must read the R-A1 schema directly before starting; its schema is
  not currently present in this workspace.
- The repeated `3-4` feature-split phrase is retained as a planning constraint,
  but its exact unit remains ambiguous in the supplied text and must not be
  guessed.
- `implement-01` already has a valid Red test commit and Green implementation
  commit, but its current synthetic question/approval sequence has no actual
  Change Request response commit. It remains a draft calibration sample and must
  not be labeled compliant with the 28-PR lifecycle until a real review event
  justifies a substantive response, or the user explicitly accepts a no-change
  review outcome. Never invent a Change Request merely to fill the pattern.

### 2026-08-21 — Initial ten-sample production order

- Within the 28-PR experiment, complete `implement-01` through `implement-05`
  first, followed by `review-01` through `review-05`.
- Finish and calibrate `implement-01` before starting `implement-02`.
- The remaining 18 samples are intentionally unscheduled until the first ten
  establish a calibrated pattern.

### 2026-08-21 — Implement-01 observed review response

- Froze `ee7837e` as the initial review head.
- The observed review found that evaluator registry validation happened after
  generator invocation and that `evaluators or defaults` erased the meaning of
  an explicit empty registry.
- Added Red review regression commit `8d63b61` and Green structural response
  commit `c21f195`; no reviewed commits were rewritten.
- Final targeted tests are 10/10 and the full suite is 73/73. The strict global
  allowlist passes with the new `tests/test_pipeline_eval_harness.py` path.
- The PR description remains the initial-snapshot artifact. Canonical review and
  thread files now contain the observed local Change Request and response rather
  than the earlier synthetic discussion.
- `implement-01` remains `draft` pending user calibration and honest completion
  or waiver of remote PR, push/merge, and capture items.

### 2026-08-21 — Implement-01 three-thread completion

- Added two further material review findings after code-level reproduction:
  locality incorrectly rejected a matching baseline that contained target text,
  and behavior requests never reached the registered behavior evaluator.
- Added `b7c6fc3` → `45462d6` and `fc72101` → `ffce1a2` as separate Red and
  response commit pairs without rewriting earlier review history.
- Final validation is targeted 12/12 and full suite 75/75; the readable PR
  conversation now has three four-event material threads.
- The earlier raw-session/thread files have been deleted. Prompt history,
  readable conversation, and review JSON retain the useful evidence.

### 2026-08-21 — I-A1 artifact-style calibration

- The provided BrainWash I-A1 example is the style reference for Implement
  samples: a concise first-snapshot PR description, readable PR conversation,
  Codex-export-style prompt history, and a factual development activity report.
- Do not invent intervening user prompts to make the history look conversational.
  Label source-derived work as a work stage; retain only actual user messages as
  user instructions.
- Mermaid diagrams and long design essays are optional supporting material, not
  mandatory content in every initial PR description.
- Keep structured JSONL as the evidence source; add a readable PR-conversation
  Markdown artifact derived from it.

### 2026-08-21 — Design and information-sharing calibration

- Implement PR descriptions should now be concrete about the real design pattern
  and include before/after Mermaid diagrams when the change has a meaningful
  structural comparison.
- Review records should use friendly, readable emoji markers while retaining
  observation, impact, actionable direction, and a relevant official reference
  when one exists.
- `implement-01` documents its actual Strategy Pattern and includes an official
  Python truth-value reference for the empty-registry review finding.

### 2026-08-21 — Thread and review-source calibration

- Implement thread target: 3-5 real threads, each with about 3-5 exchanges;
  variance is acceptable when further issues would be fabricated.
- Review PRs should begin with authentic, materially reviewable weaknesses and
  should support more threads and richer information-sharing discussion.
- Confirmed Change Request lifecycle: preserve initial review history; add new
  test and response commits; rerun focused and full checks; report SHA and
  results in the reply.
- Raw AI-session and comment-thread JSONL artifacts are no longer part of the
  deliverable. Prompt history and readable PR conversation replace them.

### 2026-08-21 — Implement CR density and linear mainline calibration

- Implement reviews should usually have 0-1 Change Request. The target thread
  count is achieved primarily through substantive questions, information sharing,
  and non-blocking suggestions, never manufactured issues.
- `implement-01` has three material Change Requests because it was completed
  before this density calibration. Preserve its evidence; do not rewrite history
  to make it appear to have fewer. It is an exception, not the future template.
- Fast-forward each completed local sample to `main`; begin the next sample from
  that head, creating a single linear sequence `test -> implementation -> review
  response` from the original mainline.
- Applied this rule to `implement-01`: `main` now fast-forwards through
  `ffce1a2` with no merge commit. `implement-02` starts from that exact main
  head.

Future user corrections should be appended here with date, affected artifacts,
and whether existing samples require migration.

### 2026-08-21 — Implement-01 full reconstruction

- The user explicitly requested that `implement-01` be rebuilt from the start.
  The previous eight-commit version ending at `ffce1a2` is not the active sample;
  it is retained only on `archive/implement-01-overcommitted` for recovery.
- Reset the active `main` to base `87abaa3`, then rebuilt the sample as exactly
  two linear commits: `d30cae7` (Red tests) followed by `fc2c304` (Green
  implementation). `fc2c304` is both the initial-review and final main head.
- The rebuilt sample has 300 changed product/test lines (257 additions,
  43 deletions) in three allowed paths. Validation is targeted 9/9 and full
  suite 72/72.
- Its review record has three evidence-based design/contract discussions and
  zero Change Requests. This is an accepted no-change review outcome: do not
  manufacture a response commit when the review finds no code defect.
- Replaced the old artifact names and content with the canonical ordered layout:
  `01_prompt_history.md`, `06_capture_list.md`, and `07_checklist.md`; retained
  no raw AI-session or raw comment-thread JSONL files.

### 2026-08-21 — Commit-message convention

- Follow the repository's visible Conventional Commit style: lowercase type,
  colon, then a concise imperative action phrase, for example
  `feat: record memory fallback events` or
  `refactor: decompose dynamic evidence loop into stage objects`.
- Use the type that reflects the change (`feat`, `refactor`, `fix`, `test`,
  `docs`, and so on). Prefer no parenthesized scope unless the user later
  explicitly asks for one.
- Do not put `(#PR-number)` in ordinary local commits. It is a squash-merge
  presentation detail and should appear only when an actual remote merge adds it.

### 2026-08-21 — Prompt and Implement-review correction

- Prompt history is a virtual, natural developer-prompt scenario showing what
  prompts would yield the PR. It is not the live conversation used to author the
  dataset. The scenario must remain grounded in the committed implementation and
  observed test outcomes.
- Avoid awkward Korean translations or transliterations for established code and
  ML terms. Use clear English terms such as `locality`, `baseline`, `evaluator`,
  `harness`, `Strategy`, and `model edit` in Korean narrative where appropriate.
- Implement-track source PRs represent strong implementation work. Their review
  threads should inspect real design/behavior decisions and should not contain
  reviewer information-sharing, tutorial citations, or performative teaching.
  Information-sharing review with references belongs to Review-track samples.
- Final reports must use respectful Korean honorific prose.
- Remove `03_reviews.jsonl`; `04_pr_conversation.md` is the only canonical
  review record for the sample.

### 2026-08-21 — Prior classification removed

- A classification system carried over from unrelated prior work is out of
  scope. Do not use, infer, preserve, or mention it in planning, samples,
  review rationale, or future dataset decisions.

### 2026-08-21 — Implement-01 A+ reconstruction

- The user requested another from-base reconstruction focused on the Implement
  A+ criteria: substantive test-first work plus a structural refactoring, not
  superficial documentation or test-count inflation.
- The active sample now starts from `87abaa3` and has exactly two commits:
  `dda9a7c` (`test(implement-01): specify evaluator registry contracts`) and
  `dcd46fd` (`feat(implement-01): preserve locality with evaluation strategies`).
  The earlier A-version
  is recoverable only through `archive/implement-01-a-version`.
- The implementation separates `CaseEvaluator` Strategy, `EvaluatorRegistry`,
  and `EvaluationReportBuilder`. Tests cover baseline preservation, unscored
  handling, independent kind scores, no generator call on incomplete registry,
  custom evaluator context, and behavior-kind construction.
- The active range changes 299 lines in four allowlisted files and passes 6
  contract tests, 5 pipeline tests, and the 74-test full suite. Its implement
  review remains three design/behavior checks with zero Change Requests.

### 2026-08-21 — Dataset ID in commit scope

- Include the active sample ID as the Conventional Commit scope. For example,
  `test(implement-01): ...` and `feat(implement-01): ...`.
- This explicitly overrides the prior default of omitting scopes. Keep the
  subject concise and do not append a PR number unless a real squash merge does so.

### 2026-08-21 — Korean narrative terminology

- Dataset narratives use natural Korean for widely established concepts such as
  validation, configuration, aggregation, execution, result, failure, loop,
  and parameter.
- Keep English only where it is a code identifier, established specialized
  term, or direct API contract: for example `locality`, `baseline`, evaluator,
  registry, `Strategy`, `EvaluationContext`, and `unscored`.
- Do not translate terms mechanically; prefer the wording a Korean developer
  would naturally use in a PR.

### 2026-08-21 — Implement-02 completion

- Implement-02 begins linearly from Implement-01 head `dcd46fd` and ends at
  `29bb11e` after `ae1de36` (Red test) then a behavior-preserving Factory
  refactor.
- `BenchmarkRequestFactory` centralizes `CorrectionRequest` construction and
  shared metadata rules. Each benchmark loader retains ownership of its raw
  schema interpretation and passes normalized values through
  `BenchmarkRequestSpec`.
- Evidence includes three new Factory tests, six existing benchmark adapter
  tests, 77 full-suite tests, and identical JSON output before/after for
  CounterFact, KnowEdit, MQuAKE, and RippleEdits.
- The patch changes 229 lines in two allowlisted paths. The synthetic review
  contains three design/behavior checks and zero Change Requests.

### 2026-08-21 — Implement size guidance clarified

- The 200-line threshold is a minimum, not a target range to optimize toward.
  Prefer a complete, natural abstraction and its necessary tests even when the
  resulting Implement patch exceeds 300 lines. Reviewability and functional
  completeness decide the boundary.

### 2026-08-21 — Implement-03 completion and review correction

- Implement-03 continues linearly from Implement-02 head `29bb11e` with
  `5655dde` (Red test), `3cb00d9` (Registry refactor), and `3c11c02`
  (review-response regression test).
- `AlgorithmRegistry` is a mapping-compatible Registry-pattern abstraction for
  adapter registration, explicit replacement, mapping conversion, and required
  lookup. `BrainwashPipeline` delegates missing-adapter handling to it.
- The algorithm override uses `dataclasses.replace()` so only `algorithm` and
  `reason` change while all other routed `AlgorithmPlan` fields remain intact.
- The initial PR evidence includes five registry/pipeline tests, five existing
  pipeline tests, an 82-test full suite, and identical before/after JSON for
  both the default fact route and an explicit DPO override.
- The Registry refactor also corrects the RAG adapter's inherited default name
  by registering it as `RAG_CONTEXT_PATCH`. This is a deliberate identity fix,
  not part of the fact/DPO parity claim.
- Review raised one legitimate Change Request for a direct RAG identity
  regression test. Commit `3c11c02` adds it; final evidence is six
  registry/pipeline tests, five existing pipeline tests, and an 83-test suite.
- The final patch changes 242 lines across three allowlisted paths. Its
  synthetic implement review contains five design/behavior threads, not a
  mechanically fixed three-thread template.

### 2026-08-21 — Implement-04 completion

- Implement-04 continues linearly from Implement-03 final head `3c11c02` with
  `5e08fba` (Red test) followed by `f3795e2` (refactor).
- `DynamicActionStageRegistry` dispatches dynamic planner `answer`, `ask`, and
  unsupported actions to distinct action Strategies. `DynamicEvidenceState`
  owns the action log, evidence steps, answer lookup, memory dependencies, and
  memory-used state for one execution.
- The patch preserves planner protocol, memory recovery, output schema, and a
  representative scripted dynamic execution JSON. Evidence includes three new
  stage/state tests, 22 existing decomposition tests, and an 86-test suite.
- The patch changes 491 lines across two allowlisted paths; exceeding 300 lines
  is justified by one coherent loop-to-Strategy/State refactor. Its PR has two
  concise Review Points and three material review threads with no Change Request.

### 2026-08-21 — Implement-05 completion

- Implement-05 continues linearly from Implement-04 head `f3795e2` with
  `4fa7b66` (Red test) followed by `3819dbe` (refactor).
- `PreferenceRecordStrategy` separates paired chosen/rejected rows from binary
  labelled rows. `PreferenceExportAdapter` owns common JSONL output, while
  `PairedPreferenceExportAdapter` shares the SimPO/DPO dataset and asset flow.
- Evidence includes four new preference export tests, five existing pipeline
  tests, a 90-test suite, and identical normalized SimPO/DPO/KTO manifest plus
  JSON/JSONL artifact output before and after the refactor.
- The patch changes 282 lines across two allowlisted paths. Its PR uses two
  concise Review Points and three material review threads with no Change Request.

### 2026-08-21 — Review-track initial PR quality

- A Review-track initial PR may deliberately include reviewable design, test,
  or maintainability issues so there is meaningful material for the reviewer.
- The issues may be synthetic, but the initial implementation must still look
  like plausible project code: it must have coherent intent, runnable context,
  and a bounded scope. Do not make it so obviously poor or toy-like that a
  real reviewer would reject it without substantive discussion.
- The goal is credible review depth, not manufactured nitpicks or a deliberately
  broken submission. Material Change Requests still require independent code or
  test commits after the initial PR.
- Missing tests or missing coverage for a meaningful boundary are valid Review
  PR issues. A Change Request may ask for a regression or contract test when it
  identifies the concrete risk, the scenario to cover, and the expected result;
  the response is then an independent test or implementation commit.
- Review-track conversations use a friendly, horizontal peer-review voice with
  natural GitHub short-code emoji where appropriate (for example `:thinking:`,
  `:+1:`, and `:smile:`). They should invite discussion, acknowledge tradeoffs,
  and avoid a commanding or tutorial-like tone.
- Strong review comments provide concrete evidence: the affected code path,
  a reproducing or missing test scenario, observable behavior, and—when it
  materially helps—an authoritative reference such as official documentation,
  PEP, or library documentation. References are information-sharing evidence,
  not decorative links.
- The same strict source/test allowlist applies without exception to every
  Review-track sample, just as it did to Implement-01 through Implement-05.
- Review-track samples may contain more threads and longer, natural back-and-
  forth discussion than Implement samples when there are enough material issues.
  Do not set a fixed count or manufacture exchanges merely to make a review look
  active.

### 2026-08-21 — Review-track information-sharing enforcement

- Every Review-track sample must visibly include information sharing, not only a
  defect report and proposed patch. The sharing must explain a directly relevant
  behavior, rule, or tradeoff through a short code example, a concrete before/
  after case, or an authoritative source when one materially helps.
- Minimum requirement: each Review PR includes at least one directly relevant
  official document, PEP, standard, or primary library reference, plus a short
  explanation or code example that connects that material to the finding.
- A bare link or vague reference is insufficient: the review comment must make
  the point understandable without opening the link. Official documentation is
  useful when it directly supports language or library semantics.
- Record this evidence in the PR conversation, final report, and publication
  checklist. Existing Review-01 through Review-05 records require this same
  audit; this is not optional for later samples only.

### 2026-08-21 — Memory-domain Git allowlist

- The earlier BRAINWASH code allowlist is superseded for future Git code work.
  Modify only `brainwash/memory/update_db.py`,
  `brainwash/algorithms/memory_edit.py`, `brainwash/memory/ledger.py`,
  `brainwash/router.py`, `brainwash/schema.py`, `brainwash/memory/schema.sql`,
  `brainwash/memory/promotion.py`, `brainwash/io.py`, and tests matching
  `tests/test_router*`, `tests/test_memory_*`, or `tests/test_update_db*`.
- User-supplied spellings `memroy`, `schmea.sql`, and `schema.ph` refer to the
  repository paths `memory`, `schema.sql`, and `schema.py`. Treat the actual
  paths above as the strict list; no other code or test file may be affected.

### 2026-08-21 — Black formatting and comment-cleanup reconstruction

- The user requested that Implement and Review 01–10 code history be rebuilt
  from the oldest active code commit, rather than adding a final formatting
  commit.
- Every rewritten code commit must apply Black before it is recorded. Do not
  commit a pre-Black source snapshot.
- Remove `#` comments and inline comments from the Python files changed by the
  01–10 code history. Do not add docstrings during this cleanup; existing
  docstrings remain API documentation rather than newly introduced comments.
- Because each commit SHA, line anchor, raw change count, initial review head,
  and final head changes during such a rewrite, synchronize every affected
  sample manifest and readable artifact, then repoint local review branches to
  their rewritten initial review heads.
