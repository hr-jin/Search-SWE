# Available Resources

## Answering API configuration

Read the following runtime environment variables when implementing and testing your answerer:

| Variable | Meaning |
| --- | --- |
| `ANSWER_API_KEY` | Credential for the answering API |
| `ANSWER_API_BASE_URL` | OpenAI-compatible API base URL; the chat endpoint is this URL plus `/chat/completions` |
| `ANSWER_MODEL` | The only model your answerer may call |

These fields are blank in the published configuration template. The person running the benchmark fills them in before starting Harbor, which injects them into your development environment and the answering stage. Do not hard-code a key, URL, or model. You can check that the variables are set without printing the key:

```python
import os
for name in ("ANSWER_API_KEY", "ANSWER_API_BASE_URL", "ANSWER_MODEL"):
    if not os.environ.get(name):
        raise RuntimeError(f"Missing runtime setting: {name}")
```

Only the configured endpoint and model are allowed. Do not use other providers, external search or answer services, or benchmark-answer datasets. Credentials must not be written into submitted code, memory, indexes, prompts, or logs. The model running your coding session is separate from this API resource.

## Where API calls are allowed

Your system has three components:

1. **Finished memory library:** prepare it locally from the supplied history and submit it under `/app/memory/`. Choose its internal format yourself. The memory may retain at most 20% of the dialogue text (735,924 bytes); the verifier expands the containers the runtime can read and counts the result. No API calls for preparation.
2. **Retriever:** accept a query and return its Top-10 memory records. No API calls, including hosted embedding or reranking APIs.
3. **Answerer:** combine the current query with those retrieved records to produce an answer. This is the only component allowed to call the configured API.

These restrictions also apply while you develop and test the system. Do not call APIs to interpret the raw corpus, resolve references for memory construction, summarize history into memory, create indexes, or prepare retrieval results. Local computation and installed local libraries are allowed. Any answerer test must provide only its current query and the records returned by your retriever.

During evaluation, the submitted memory is used as-is; no builder is invoked. Retrieval runs without network access or API credentials. Each answerer invocation receives only one query and its retrieved records, plus its corpus-independent code, runtime tools, and answering API configuration. It cannot read the original history, full memory/index, other queries, their retrieved records, or prior answers.

## Calling the API from your answerer

Use the task-provided Chat Completions transport in your submitted answerer. It forwards your request to `ANSWER_API_BASE_URL` using `ANSWER_API_KEY`; you supply the prompts and parse the response. It does not provide answering logic, memory search, or reference labels. Direct network access is disabled during evaluation, so this transport is the permitted API route.

```python
import importlib.util
import os

spec = importlib.util.spec_from_file_location("task_llm", os.environ["TASK_LLM_CLIENT"])
api = importlib.util.module_from_spec(spec)
spec.loader.exec_module(api)
response = api.chat_completion(
    messages=[{"role": "user", "content": prompt_from_current_query_and_retrieved_memories}],
    model=os.environ["ANSWER_MODEL"],
    temperature=0,
    max_tokens=1000,
)
text = response["choices"][0]["message"]["content"]
```

During evaluation, the runtime supplies `TASK_LLM_CLIENT` and `TASK_LLM_FD` to `answer.sh`. Preserve `TASK_LLM_FD` if launching another process to implement the answerer. The API key and base URL remain available under the configuration names above; the retrieval process does not receive them.

Allowed request options are `model`, `messages`, `temperature`, `top_p`, `max_tokens`, `response_format`, and `thinking`. Each request may contain 1–32 messages, each with string `role` and `content` fields; roles are `system`, `user`, or `assistant`. Serialized requests must fit within 128 KiB including the newline. Each question permits up to two API calls and each call up to 2,000 output tokens. Streaming, tool calls, and alternate endpoints are unsupported. Provider-specific options should be used only if supported by the configured endpoint.

The answerer must use only the current query and retrieved evidence. Its prompts and static dependencies must not contain corpus-specific facts. Do not reuse information from previous questions. When the evidence is insufficient, return the documented abstention output.

Using an API outside the answering component, accessing unprovided evidence, or otherwise bypassing the memory and retrieval pipeline is a task violation and sets the entire score to zero.
