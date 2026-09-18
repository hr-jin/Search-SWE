# Internal runtime interface

The external task is question text in, answer text out. These entry points let the evaluator connect the submitted retriever and answerer while enforcing their access permissions. Intermediate memories are internal context, not part of the final answer and not independently scored.

## Retrieval

```bash
/app/run.sh --memory-dir /app/memory --question /path/to/question.txt --output /path/to/memories.json
```

`question.txt` contains one UTF-8 natural-language question, with no ID or meeting metadata. Write a JSON array of up to ten records in relevance order to `memories.json`. Each record has nonempty string `id` and `text` fields; IDs are distinct within a result and additional metadata is allowed. This handoff format does not constrain the stored memory format. Records must come from the submitted memory.

## Answering

```bash
/app/answerer/answer.sh --context /path/to/context.json --output /path/to/answer.txt
```

The evaluator passes one JSON object containing `question` (the same question text) and `memories` (the array returned by the retriever). Write only the final UTF-8 answer to `answer.txt`, without a JSON wrapper, record list, or memory-ID citations. Both programs must return exit code 0.

Each answering process is fresh and can read only the current context and its own code and runtime dependencies. Submission files are read-only. The verifier enforces the same contract. Paths can vary; use the supplied command-line arguments.
