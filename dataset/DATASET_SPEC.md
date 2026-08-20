# Dataset Artifact Contract

## 1. Dataset unit

A sample represents one coherent pull-request lifecycle grounded in a specific
BRAINWASH commit. It is an ordered trace, not a bag of messages.

Every sample directory uses this layout:

```text
implement/implement-01/        # or review/review-01/
├── manifest.json
├── 00_task.md
├── 01_prompt_history.md
├── 02_pr_description.md
├── 04_pr_conversation.md
├── 05_final_report.md
├── 06_capture_list.md
└── 07_checklist.md
```

Numeric prefixes define canonical reading order. Do not add a review JSONL
duplicate; `04_pr_conversation.md` is the canonical review record.

## 2. Manifest

`manifest.json` records at least:

```json
{
  "schema_version": "1.0",
  "sample_id": "implement-01",
  "track": "implement",
  "status": "draft",
  "language": "ko",
  "repository": {
    "name": "BrainWash",
    "base_branch": "main",
    "base_commit": "<full sha>",
    "head_branch": "feature/example",
    "initial_review_head": "<full sha at first PR snapshot>",
    "final_head_commit": "<full sha after review responses>",
    "commits": [
      {"role": "red_tests", "sha": "<full sha>"},
      {"role": "implementation", "sha": "<full sha>"}
    ]
  },
  "perspective": {
    "primary_actor": "implementer",
    "counterpart": "reviewer"
  },
  "change_budget": {
    "minimum_changed_lines": 200,
    "preferred_upper_bound": 300,
    "measured_changed_lines": null,
    "counting_rule": "product code and tests; additions plus deletions"
  },
  "artifacts": []
}
```

The status lifecycle is `draft -> calibrated -> accepted`.

## 3. Task record

`00_task.md` contains:

- user-visible task request;
- business and technical context;
- acceptance criteria;
- constraints and explicit non-goals;
- expected testing level;
- the reason the task meets its track-specific workload guidance while
  remaining one coherent review unit.

The task must permit more than mechanical editing. It should expose at least one
meaningful design choice and one plausible review discussion.

## 4. Prompt history

`01_prompt_history.md` is a concise, human-readable **virtual development
prompt scenario**: the natural prompts a developer would give to produce the
sample PR. It is not an export of the current dataset-authoring conversation.
Keep the scenario grounded in the committed code and observed verification
outcomes; label non-dialogue steps as work stages. Never include private
chain-of-thought, secrets, or irrelevant terminal output.

## 5. Git commit contract

The sibling BRAINWASH repository's Git history is the source of truth. Do not
duplicate the change as a dataset diff artifact.

Implement samples require at least two ordered commits:

```text
base commit
└── test commit            # Red: tests fail for the intended missing behavior
    └── implementation     # Green: targeted and regression tests pass
```

The manifest records full commit SHAs, roles, branch, and ancestry. The prompt
scenario records the Red and Green commands and their summarized results.
Tests and implementation must not be combined into one commit merely to make the
final checkout green.

### BRAINWASH path allowlist

The complete `base_commit..head_commit` range, including intermediate commits,
may change only:

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

This is strict. A sample is invalid if any commit touches another path, even when
the final tree later removes that change.

For Implement samples, the total change size must be at least 200 added plus
deleted lines across the sample's commit range. The preferred band is 200-300
lines; exceeding 300 is valid when the change remains one coherent, reviewable
unit. For Review samples, the initial PR under review should normally contain
about 50-200 changed lines, with feature completeness taking priority. The final
report must state:

- additions and deletions;
- files changed;
- whether generated or formatting-only lines were excluded;
- reviewability justification when the patch exceeds 300 lines.

The 200-line minimum applies to Implement samples. The upper bounds are
reviewability heuristics, not rejection boundaries. Never introduce churn to
reach a threshold, truncate a complete feature, or reduce readability merely to
remain inside a preferred band.

Implement samples must make the TDD sequence observable: failing test, minimal
implementation, refactoring, and final regression run. A final diff alone is
not sufficient evidence of TDD.

Review samples do not require a TDD history in their initial PR. The initial
review head must nevertheless be complete and pass the existing full test suite.
When review identifies a regression risk or requests a behavior change, add the
focused regression test and response implementation as new post-review commits.

## 6. Pull request

`02_pr_description.md` follows the repository's assumed pull-request template.
It includes:

- a short summary;
- major changes;
- concrete **Review Points**;
- explicit PR type;
- executed tests at the initial PR head;
- Todos for review feedback.

Detailed design reasoning, tool chronology, review outcomes, risks, and optional
diagrams belong in the development activity report or review record when they
materially help comprehension. They are not mandatory initial-PR sections.

For Implement samples that materially restructure control flow or responsibility,
the initial PR description must name the pattern actually used, explain the
responsibility boundary, and include before/after Mermaid diagrams. Do not claim
a pattern that the code does not implement.

The PR description represents only the initial PR snapshot. It must not narrate
future review dialogue, Change Requests, or later fixes as if they were already
known. Those events belong in commits, threads, and the final report.

Local PR artifacts must include `Synthetic GitHub artifact: true` near the top.

Review Points are scored. High-quality points reduce reviewer search cost by
identifying the most consequential decisions without pre-empting disagreement.

## 7. Reviews and threads

`04_pr_conversation.md` is the canonical readable review conversation, including
replies, Change Request commits, verification, and resolution. Do not create a
parallel `03_reviews.jsonl` record.

Every material review comment should contain:

- file and line anchor, when applicable;
- severity: `blocking`, `important`, `suggestion`, or `question`;
- observation grounded in the code;
- impact or risk;
- actionable direction without unnecessarily prescribing implementation;
- Clean Code or TDD principle when it adds explanatory value.

For Review-track samples, use authoritative links when they materially support a
finding and make the thread useful as information sharing. Implement-track
reviews evaluate a strong completed implementation: keep them focused on real
design and behavior checks, not reviewer teaching or reference dumping. Friendly
emoji markers are encouraged in the human-readable conversation to make
investigation, response, fix, and approval easy to scan, but they never
substitute for evidence or guidance.

Threads preserve disagreement and revision. They must not manufacture easy
agreement merely to make an actor look good.

Implement samples target 3-5 material threads with about 3-5 exchanges each;
Review samples normally have more. These are review-quality targets, not quotas.
Every thread must arise from a real finding. Friendly emoji markers are
encouraged in the readable conversation but cannot replace technical content.

Implement samples normally have 0-1 Change Request. Use other material threads
for evidence-based questions and behavior/design confirmation. Do not turn every
review observation into a required code change merely to increase the
response-commit count.

Review samples start from an authentically reviewable implementation with
material shortcomings such as missing TDD evidence, Clean Code issues, or SOLID
responsibility violations. Do not invent arbitrary faults in code that is
already clean merely to increase thread count.

Implement initial PRs must be complete and tested. Review PR source snapshots
must be authentically reviewable rather than arbitrarily defective. After review
begins, do not rewrite reviewed history; implement
Change Requests as new commits and record the resulting test run. The manifest
and prompt history must distinguish the initial review head from the final head
when a review-response commit exists.

After each review-response commit, the corresponding thread response records the
commit SHA, focused test result, and full-suite result. After the first review
event, rebase, squash, force push, and force-with-lease are prohibited.

The repository history remains linear from `main`: test-first Red, initial
implementation Green, then any review-response commits. After a sample's local
review completes, fast-forward `main` to its final head and start the next
sample from that updated mainline.

## 8. Track-specific perspective

### Implement

The final report focuses on requirement interpretation, TDD chronology, code
quality, refactoring choices, executed test events, PR clarity, and how review
feedback was handled. Reviewer messages are supporting context.

### Review

The final report focuses on code comprehension, issue discovery quality,
severity calibration, false-positive avoidance, knowledge sharing, discussion
quality, and whether the review changed the implementation. Author messages are
supporting context.

## 9. Final report

`05_final_report.md` synthesizes the complete ordered trace. It includes:

- executive summary;
- timeline;
- code and design assessment;
- TDD/Clean Code assessment;
- communication assessment;
- PR description and Review Points assessment;
- tool-use assessment;
- unresolved risks;
- self-critique from the primary actor's perspective;
- rubric scores with evidence links;
- dataset-quality notes and any synthetic elements.

The report must not infer success from polished prose alone. Claims should link
to a code hunk, test result, review event, or thread event in the sample.

## 10. Quality gates

A sample is not accepted unless:

- all required artifacts exist and parse;
- sequence numbers are unique and increasing;
- recorded commits exist and form the declared ancestry from the base commit;
- every changed path in the complete commit range matches the strict BRAINWASH
  allowlist;
- for Implement samples, the Red commit fails for the intended behavior and the
  implementation commit passes the recorded Green checks;
- for Review samples, the initial review head passes the existing full suite,
  and review-driven regression tests are added only when the finding requires
  them;
- the initial review head and final head are recorded, including intervening
  review-response commits when present;
- executed-test claims match the prompt history, commits, and review record;
- the change-size count is present;
- simulated GitHub content is labeled;
- no secret or hidden chain-of-thought is stored;
- the primary perspective matches the track;
- the final report cites concrete evidence;
- the user has completed the required calibration step.

## 11. Track-specific grading target

All samples target at least A (85 or above) and preferably A+ (95 or above).
Artifact completeness alone does not establish the grade.

### Review PR grading

- **A+ (95 and above):** active horizontal discussion, questions, replies, and
  exchange of multiple perspectives.
- **A (85-95):** considerate communication with clear evidence and concrete
  guidance.
- **B+ (75-85):** clear evidence or concrete guidance.
- **B-C:** comments are mainly formal or cosmetic corrections.

### Implement PR grading

- **A+ (95 and above):** tests plus structure-improving refactoring.
- **A (85-95):** structure-improving refactoring, including an appropriate
  abstraction or abstract class where useful.
- **B+ (75-85):** readability-oriented reorganization of functions, classes, or
  files.
- **B-C (50-75):** simple code-smell cleanup such as naming consistency or
  comment removal.

The user-provided ranges currently share endpoints. Preserve that notation and
do not assign exact-boundary grades until calibration resolves inclusivity.

## 12. Behavior-preserving refactoring proof

For a behavior-preserving refactor, record all applicable evidence:

- unchanged existing safety-net tests across the base-to-head range;
- a separate new specification test file when an allowlisted path is available;
- the full `python3 -m unittest discover -s tests` result;
- a before/after CLI-output diff demonstrating identical observable behavior.

The separate-test guidance never overrides the strict path allowlist. If no
separate allowlisted path exists, document the constraint instead of creating an
out-of-scope file.

## 13. Six deliverable categories

Each sample exposes six user-facing deliverable categories:

1. source changes and auditable commits;
2. initial-snapshot PR description;
3. lifecycle report;
4. Markdown prompt history derived from actual session evidence;
5. capture list and stable capture names;
6. publication/merge checklist.

These are categories rather than a requirement to have exactly six files. The
manifest, task record, prompt history, and readable PR conversation are the
required narrative evidence. Keep review JSONL only when it contributes distinct
structured evidence. Never add a capture, prompt, tool identity, review event,
or test result that was not actually observed.
