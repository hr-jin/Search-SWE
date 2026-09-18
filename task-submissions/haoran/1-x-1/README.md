# Compact Conversational Memory Question Answering

Build a memory, a retriever, and a grounded answerer over long meeting transcripts.

**Task:** `task-1-x-1` · **Mode:** Implementation · **Metric:** MemoryAnswerAccuracy

## Overview

Long meetings contain the information a question needs, but not where a keyword
search looks for it: participants interrupt each other, refer back to things by
pronoun, revise earlier proposals, and leave questions open. Recovering an
answer means deciding what to keep, finding it again, and reporting its status
accurately.

This task supplies 67 meetings from three longitudinal ICSI series: Bmr (29),
Bro (23), and Bed (15). The 53,600 utterances contain 719,788
whitespace-delimited words, with interruptions, unclear references, corrections,
and evolving proposals. No starting system is provided; the agent builds the
whole pipeline, starting from the transcripts themselves.

## What This Task Tests

- Choosing what to keep when the transcripts cannot be retained in full, and
  representing it in a form that survives compression.
- Retrieving the passage that answers a question when the question and the
  transcript use different words for it.
- Reporting what was decided, proposed, or left open, and resolving the
  references that a meeting leaves implicit.
- Fitting all of this into a fixed memory and storage budget.

## Task Setup

### Provided Assets

| Asset | Purpose |
| --- | --- |
| `data/history.jsonl` | The complete transcripts, visible throughout development |
| `data/validation/queries.jsonl` | 30 public questions |
| `data/validation/golden_answers.jsonl` | Public references and required factual points |
| `data/validation/evidence.jsonl` | Transcript excerpts supporting the public answers |

Public assets are mounted read-only under `/task/data`. All 67 meetings have
questions across the two splits. Only the questions and reference answers are
split; the transcripts are shared.

### Fixed Components and Allowed Changes

The transcripts, the internal retrieval-to-answering interfaces, the output
format, and the resource budgets are fixed. The memory format, retrieval
strategy, prompts, and the answerer's internal design are implementation
choices. Only the submitted answerer may call the configured answering API;
memory construction and retrieval must run on local computation alone.

### Environment and Resource Limits

The CPU Python 3.12 environment provides 8 CPUs, 8 GiB memory, 8 GiB storage,
and no GPU. The agent has two hours and the Harbor verifier one hour. Retrieval
has a combined 60-second budget for a full question set and answering a combined
600-second budget, with at most two answering API calls per question.

The memory may retain at most 20% of the dialogue text; the verifier measures
what the memory expands to, so compression does not enlarge that budget. Every
submitted file, the memory included, also counts toward a 183,981-byte disk
budget.

## Submission Contract

The deliverable is a finished `/app/memory/` artifact plus executable
`/app/run.sh` and `/app/answerer/answer.sh`. The verifier runs the retrieval
entry point once per question against the submitted memory, then hands the
top ten records to the answering entry point. Evaluation never runs a
memory-construction program.

The full interface and output schema live in [instruction.md](instruction.md).

## Evaluation

### Answer Quality

An answer scores one only when it satisfies the reference answer's required
facts without contradicting them. Paraphrases are accepted; missing required
facts and unanswered questions are incorrect. MemoryAnswerAccuracy is the mean
of these binary outcomes over 118 hidden questions, and the reported 0–100
score is 100 times that mean. Three judge calls per question vote by majority.

### Correctness and Resource Gates

Both entry points must exist, be executable, and honour the internal
interfaces; the submission runs read-only and must behave the same across
repeated questions. The retained-dialogue and disk budgets are enforced by
measurement. Invalid output or a failed executable gate scores zero.

### Integrity Checks and Final Reward

The answer judge and the trajectory audit are separate: the first grades answer
equivalence, the second checks task compliance and can set the whole reward to
zero. Their credentials are isolated from submission processes and from each
other. Retrieved records are not scored separately.

## Running This Task

From the repository root, follow the [quick start](../../docs/quickstart.md)
to install the pinned Harbor dependencies and prepare Docker. Use the
[evaluation guide](../../docs/evaluation.md) to configure the selected agent and
task credential profile. The
[asset guide](../../docs/assets.md) covers downloads, checksums, and cache options.

This task needs `ANSWER_API_*` for the submitted answerer, `ANSWER_JUDGE_*` for
answer scoring, and `VERIFIER_OPENAI_*` for the trajectory audit.

```bash
python scripts/download_assets.py --task-path task-submissions/haoran/1-x-1
bash scripts/run_task.sh --task-path task-submissions/haoran/1-x-1 --model "YOUR_AGENT_MODEL"
```

Replace `YOUR_AGENT_MODEL` with your configured model. Add `--dry-run` to inspect
command construction without starting an evaluation; this does not validate
assets, credentials, or hardware.

## Task Files

| File or directory | What to read it for |
| --- | --- |
| [instruction.md](instruction.md) | Complete agent-facing specification and executable contract |
| [task.toml](task.toml) | Task identity, artifact collection, and phase budgets |
| [assets.json](assets.json) | Fixed asset paths, immutable revisions, and checksums |
| [Environment guide](environment/docs/environment.md) | Installed runtime and task environment |
| [Environment configuration](environment/docker-compose.yaml) | Read-only mounts and hardware requests |
| [Verifier](tests/) | Execution, output validation, and scoring implementation |
| [Resource policy](environment/docs/available_resources.md) | Allowed submission APIs and model restrictions |

## Asset Publication Status

The four fixed inputs are staged locally under `data/`. They are not published yet: `assets.json` points at a personal temporary dataset and leaves `revision` empty, so this package is review-stage only. Publishing the files and pinning the resulting immutable commit is required before the manifests validate and a reviewer can restore the inputs.

## Structural Precedents

Authoring rules require consulting one or two current formal packages as bounded
structural precedents and recording why they match. The exact paths, the files
consulted, the patterns reused, and the intentional deviations are recorded in
the PR description for this submission. In summary, the three references were:
an Implementation-mode task graded by a model judge over free-text answers with
a separate verifier and a private hidden split; an Implementation-mode CPU task
with a continuous held-out metric; and a CPU task with a pinned Hugging Face
asset manifest. No task-specific id, dataset, threshold, pin, licence or access
policy was inherited from them; every value here was established independently,
and the current skill and validators were treated as authoritative over any
older example.

## Provenance and Limitations

Source: [ICSI Meeting Corpus](https://groups.inf.ed.ac.uk/ami/icsi/download/), Janin et al., *The ICSI Meeting Corpus*, ICASSP 2003. Preserve the supplied [license notice](ICSI_LICENSE.html) and source attribution.

The conversations are human transcripts. The 148 question-answer labels were authored later and are not original ICSI QA annotations. They have transcript-linked evidence, independent model-call reconstruction, full-transcript audits, and editorial corrections, but have not received independent human certification. The model judge can also vary despite majority voting.
