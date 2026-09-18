# CPU Docker Environment

This image provides a Conda-managed Python 3.12 environment with CPU-only PyTorch and the common search, embedding, indexing, document-processing, media, HTTP, and service packages used by the tasks. Installed Python packages include `torch`, `torchvision`, `torchcodec`, `numpy`, `transformers`, `sentence-transformers`, `FlagEmbedding`, `deepspeed`, `faiss-cpu`, `bm25s`, `rank-bm25`, `pyserini`, `hnswlib`, `qdrant-client`, `docling`, `marker-pdf`, `pdf2image`, `pypdfium2`, `CairoSVG`, `av`, `imageio`, `imageio-ffmpeg`, `tiktoken`, `fastapi`, `uvicorn`, `python-multipart`, `requests`, `aiohttp`, `openai`, and `pydantic-settings`.

The task Python interpreter and its installed packages are available at `/opt/conda/bin/python`. Use `/opt/conda/bin/python` and `/opt/conda/bin/pip` when invoking Python or installing packages.

The image also includes JDK 21, FFmpeg, Poppler utilities, Cairo, Git, curl, `jq`, `build-essential`, `ca-certificates`, `libffi`, `libgomp`, `netbase`, `netcat`, `procps`, `tzdata`, `unzip`, and the related system runtime libraries.

## Task-specific execution

You have 8 CPUs, 8 GiB RAM, 8 GiB storage, and no GPU. Development runs as `agentdev`; `/app` and your home directory are writable. Task data and system libraries are read-only. You can consult the complete history while building and testing your system.

`/app` starts empty. You build the memory, the retrieval entry point, and the answerer there. Evaluation uses the finished memory as a data artifact and does not execute a memory generator, so the memory must already be complete and self-contained when you finish. The memory may retain at most 20% of the dialogue text; the verifier expands it before counting, so compressing retained dialogue does not enlarge that budget. Your retriever runs offline against this memory and the provided questions; it has no original-history access or API credentials. Use installed local packages or include required dependencies in the counted submission. Development home files and extra installed packages are not transferred. `/opt/models` is not a provided model asset directory.

Your answerer runs separately for each query, with only its corpus-independent code, that query's recalled memories, runtime tools, and the answering API configuration. It may call the configured API through the provided helper. It cannot read the full memory, original history, other questions or prior answers.

Memory preparation and retrieval may not use APIs, including during development. Only the answerer may use the runtime-configured API.
