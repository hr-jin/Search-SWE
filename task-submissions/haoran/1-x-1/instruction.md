# Task: Task-1-x-1

## Task Description

Build an executable conversational-memory question-answering system over the supplied meeting transcripts. Given one natural-language question, the system should return one concise answer, drawing on a compact memory library that you build from the transcripts during this task.

The objective is to maximize answer correctness on held-out questions. You may use any method that satisfies the internal interfaces, the memory and storage budgets, and the resource restrictions.

## Requirements

- Create or modify submission files only under `/app`. Treat `/task` as read-only. `/app` starts empty: the memory, the retrieval entry point, the answerer, and every index are yours to create.
- Build the memory library yourself and leave it complete under `/app/memory/`. Evaluation runs the finished memory as a data artifact and never invokes a memory-construction program, so the memory must stand on its own; the original transcripts are not available to the retrieval or answering programs at evaluation time.
- Provide executable `/app/run.sh` and `/app/answerer/answer.sh`. Keep answering code and its static dependencies inside `/app/answerer/`; keep retrieval code, indexes, and the memory under `/app`.
- The memory may retain at most **735,924 bytes** of dialogue — 20% of the dialogue text's 3,679,621 UTF-8 bytes, counting one newline per utterance and excluding JSON metadata. The verifier expands the memory with every container format the runtime can read (xz/lzma, gzip/zlib, bzip2, zstd, lz4, brotli), follows nested containers, and counts what it expands to, so compressing retained dialogue does not buy extra room.
- All submitted files, including memory, indexes, and code, must together fit within **183,981 bytes** on disk. This is separate from the retained-dialogue budget above and is 5% of the same dialogue text. File contents and relative path bytes count once. Links and special files are forbidden.
- Keep scratch files, caches, and experiment output outside `/app`: every submitted file is transferred and counted.
- Memory construction and retrieval must use local computation only. Do not call helper APIs for summarization, memory creation, indexing, embedding, or ranking, including during development. The externally configured coding Agent itself is exempt from this helper-API restriction.
- Read `ANSWER_API_KEY`, `ANSWER_API_BASE_URL`, and `ANSWER_MODEL` from the runtime environment to configure the answering API. The benchmark runner supplies these initially blank settings. Only the submitted answerer may use this API, as documented in `/task/docs/available_resources.md`.
- Do not retain uncounted dialogue copies, hard-code public or hidden question-to-answer tables, depend on dataset query IDs, access hidden labels, or modify the evaluator.
- The evaluator allows up to 120 minutes for the Agent to complete this task; plan implementation, validation, and debugging within this time budget.

### Retrieval interface

The verifier invokes:

```bash
/app/run.sh \
  --memory-dir /app/memory \
  --question /path/to/question.txt \
  --output /path/to/memories.json
```

`question.txt` contains one UTF-8 natural-language question, with no ID or meeting metadata. `run.sh` must write a JSON array of up to ten records in relevance order to `memories.json`, and return exit status `0`. Each record must have nonempty string `id` and `text` fields; IDs must be distinct within a result, and additional metadata is allowed. This handoff format does not constrain the stored memory format, but the records must come from the submitted memory.

### Answering interface

The verifier invokes:

```bash
/app/answerer/answer.sh \
  --context /path/to/context.json \
  --output /path/to/answer.txt
```

`context.json` is one JSON object containing `question` (the same question text) and `memories` (the array returned by the retrieval step). `answer.sh` must write only the final UTF-8 answer to `answer.txt`, without a JSON wrapper, record list, or memory-ID citations, and return exit status `0`. Each answering process starts fresh and can read only the current context, its own code, and its runtime dependencies.

### Output contract

For every request the system receives one natural-language question and returns one plain-text answer:

- the answer must be a nonempty UTF-8 string of at most **2,000 Unicode characters**;
- it must not contain recalled records, citations, or memory identifiers;
- if the evidence is insufficient, the answer should say so in plain text.

## Available Validation Data

The following files are available in the task environment:

- `/task/data/history.jsonl` — the complete transcripts, with `meeting_id`, `series_id`, `date`, `speaker`, `start_seconds`, `end_seconds`, `text`, and `source_ids`. A null speaker is unidentified. Speech offsets locate an utterance within a meeting.
- `/task/data/validation/queries.jsonl` — 30 public development questions. Each row includes bookkeeping fields for analysis, but the runtime interface supplies only the natural-language question.
- `/task/data/validation/golden_answers.jsonl` — public reference answers and required factual points, keyed by `query_id`.
- `/task/data/validation/evidence.jsonl` — transcript excerpts supporting the public reference answers.

Use these files to develop and check the system. Hidden questions concern the same transcripts and are disjoint from the public questions.

A practical self-check before you finish: expand your own memory with the container formats listed above and confirm the total is inside the retained-dialogue budget; confirm every submitted file together is inside the disk budget; run both entry points by hand on the public questions and read the answers; and confirm that a second run of the same question returns the same result, because the verifier repeats every question.

## Expected Artifacts

The finalized submission must contain an executable `/app/run.sh`, an executable `/app/answerer/answer.sh`, the finished memory under `/app/memory/`, and whatever code they need.

```text
/app/
├── memory/               # finished compact memory and indexes
├── run.sh                # executable retrieval entry point
├── answerer/
│   ├── answer.sh         # executable answering entry point
│   └── ...               # corpus-independent answering code and dependencies
└── ...                   # other retrieval code and dependencies
```

## Verification

After the Agent phase, Harbor transfers `/app` to a separate verifier with the same runtime and the private question set. The verifier checks the following items:

1. **Memory and retrieval integrity.** The submission must answer from the supplied transcripts through the memory it submits. It must not keep uncounted copies of the dialogue, obtain answers or judgments from hidden labels, hard-code question-to-answer or question-to-result tables, serve a fixed answer file instead of running the submitted answerer, or reach any service other than the configured answering API.
2. **Resource compliance.** Follow `/task/docs/available_resources.md`. The retained-dialogue and disk budgets above are enforced by measurement, not by declaration, and no helper API may be used for memory construction, indexing, embedding, retrieval, or ranking.
3. **Executable and output validity.** Both entry points must exist and be executable, run read-only and across repeated requests, and honour the internal interfaces above. Retrieval has a combined 60-second budget for a full question set and answering a combined 600-second budget, with at most two answering API calls per question and at most 2,000 output tokens per call. Invalid output or a failed executable gate receives a score of `0`.
4. **Final answer score.** An answer is correct when it satisfies the required reference facts without contradictions. Paraphrases are accepted; missing required facts and unanswered questions are incorrect. Retrieved memories are not scored separately. The metric is MemoryAnswerAccuracy, averaged equally across all hidden questions.

```text
score = 100 * MemoryAnswerAccuracy
```

An independent trajectory audit checks compliance with the task and resource restrictions. A confirmed violation sets the entire submission's score to `0`.

## Hidden Test Overview

The hidden evaluation contains 118 held-out questions over the same 67 meetings, with private reference answers and required factual points. The hidden questions are disjoint from the public development examples and are not copied into the Agent-visible environment. At evaluation time each program receives only the inputs documented above.

## Environment and available resources

Before implementing the system, read the following task-provided documents:

- [`/task/docs/environment.md`](/task/docs/environment.md) — the installed Python environment, system runtime, and resource limits.
- [`/task/docs/available_resources.md`](/task/docs/available_resources.md) — the answering API resources, runtime credential handling, model and endpoint restrictions, and jailbreak penalty rules.
- [`/task/docs/runtime_interface.md`](/task/docs/runtime_interface.md) — the internal retrieval-to-answering contract enforced by the evaluator.

These documents are part of the Agent-visible task data and should be treated as read-only. Follow the resource and model restrictions in `available_resources.md`; do not infer permission to use an unlisted provider, model, or endpoint. Credentials are injected by Harbor at runtime and must not be placed in the task package or Docker image.
