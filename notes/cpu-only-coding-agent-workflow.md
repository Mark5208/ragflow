# Coding assistance on a CPU-only PC

Work entirely on the second PC. Nothing on this Mac has to reach it.

A GPU is not required. RAGFlow's default device is CPU, and the quickstart starts the CPU image. An agent on that machine can answer from documents you upload, generate code, and, if you enable the sandbox, run isolated Python or JavaScript. It does not edit a git checkout or apply patches to a project on disk.

## What the PC must already have

Source: `docs/quickstart.mdx`, "Prerequisites" and the IMPORTANT note above it.

- x86 CPU, 4 cores or more
- 16 GB RAM or more
- 50 GB disk or more
- Docker 24.0.0 or newer, Docker Compose v2.26.1 or newer
- Python 3.13 or newer, only if you run from source
- gVisor, only if the agent will execute code

Official Docker images are for x86. ARM64 is tested, but there is no maintained ARM image. On ARM, build an image from `docs/develop/build_docker_image.mdx` instead of pulling the published one.

`vm.max_map_count` must be at least 262144 before Elasticsearch will stay up. On Linux that is `sudo sysctl -w vm.max_map_count=262144`, and the same line in `/etc/sysctl.conf` so it survives reboot. Source: `docs/quickstart.mdx`, "Start up the server".

## 1. Start RAGFlow on CPU

Source: `docs/quickstart.mdx`, steps 2–6. Source: `docker/.env`, `DEVICE=${DEVICE:-cpu}` and `COMPOSE_PROFILES=${DOC_ENGINE},${DEVICE},...`.

On that PC:

```bash
git clone https://github.com/infiniflow/ragflow.git
cd ragflow
git checkout -f v0.27.2
cd docker
docker compose -f docker-compose.yml up -d
docker logs -f docker-ragflow-cpu-1
```

Leave `DEVICE` at `cpu`. Do not switch `COMPOSE_PROFILES` to `gpu`, and do not enable the `tei-gpu` profile. The `ragflow-gpu` service is the one that reserves an NVIDIA device (`docker/docker-compose.yml`, service `ragflow-gpu`).

Wait until the log shows the service listening, then open `http://localhost` in a browser on that same PC. The published web port is 80, so no port is required in the URL.

## 2. Give it a chat model and an embedding model

An agent cannot run until both defaults exist. Source: `docs/guides/models/llm_api_key_setup.md`, "Set Default Models".

On a 16 GB machine, use a hosted API. That keeps the weights off this PC. RAGFlow's 16 GB figure is the floor for its own services (the FAQ says parsing, embedding, indexing, and task processing already share that machine). Source: `docs/faq.mdx` on resource usage, and `docs/quickstart.mdx` prerequisites.

In the UI: avatar, **Model providers**, add a provider, save API key and base URL, verify the connection, add an LLM and an embedding model, then set them as the defaults. If the agent should call tools, add a chat model that supports tool calling and enable **Tool call** on that model. Source: `docs/guides/models/llm_api_key_setup.md`, "Add a Custom Model".

Use a local model only when the API is unavailable. The documented CPU-capable path is Ollama. Its own startup log lists `cpu`, `cpu_avx`, and `cpu_avx2` runners next to the CUDA ones, and the guide's first models are `llama3.2` (3B chat) and `bge-m3` (567M embedding). Source: `docs/guides/models/deploy_local_llm.mdx`, "Deploy Local Models Using Ollama".

```bash
sudo docker run --name ollama -p 11434:11434 ollama/ollama
sudo docker exec ollama ollama pull llama3.2
sudo docker exec ollama ollama pull bge-m3
```

RAGFlow in Docker reaches Ollama on the same host at `http://host.docker.internal:11434`. Check from inside the RAGFlow container:

```bash
sudo docker exec -it docker-ragflow-cpu-1 bash
curl http://host.docker.internal:11434/
```

Then add Ollama under **Model providers** with those exact model names and types (`llama3.2` / chat, `bge-m3` / embedding). That Ollama section does not document a rerank model. If loading the model times out, the FAQ says to check Ollama logs and free memory, then try a smaller model. Source: `docs/faq.mdx`, "Fail to access model(Ollama/xxxxx)".

Leave the bundled embedding service off. `tei-cpu` and `tei-gpu` are commented out in `docker/.env`, and images from v0.22 on do not ship embedding models. The default TEI model, `Qwen/Qwen3-Embedding-0.6B`, needs 25 GB. `BAAI/bge-m3` on TEI needs 21 GB. Both are above this PC's 16 GB floor. `BAAI/bge-small-en-v1.5` is the TEI option the file lists at 1.2 GB, and only if you choose to turn TEI on. Source: `docker/.env`, "The embedding service image, model and port".

IPEX-LLM's introduction says it can run on Intel CPUs or Intel GPUs, but the steps in that section set `OLLAMA_NUM_GPU=999` so layers stay on an Intel GPU. That is not the CPU procedure. vLLM, SGLang, and GPUStack are GPU paths. Skip them here. Source: `docs/guides/models/deploy_local_llm.mdx`.

## 3. Put the material the agent should know into a dataset

Source: `docs/quickstart.mdx`, "Create your first dataset" and "Set up an AI chat".

1. **Dataset** → **Create dataset**.
2. Pick the embedding model before parsing anything. It cannot be changed after a file has been parsed.
3. For plain text, Markdown, and text PDFs, choose the **Naive** parser. DeepDoc's OCR, table structure, and layout analysis are the GPU-heavy parsers, and Naive is the documented way to skip them. Source: `docs/guides/dataset/notes_and_faqs.md`, "Best Practices: Index Acceleration".
4. Upload files, parse them, and run one retrieval test.
5. Turn off RAPTOR, knowledge-graph extraction, auto-keyword, and auto-question on a small machine. Those call the LLM or do extra indexing. Same section.

If a PDF parse stalls near the end with no error in the log, the FAQ says the process was likely killed for lack of RAM. `MEM_LIMIT` in `docker/.env` defaults to 8073741824 bytes for the container. Raising `DOC_BULK_SIZE` (default 4) or `EMBEDDING_BATCH_SIZE` (default 16) also raises memory use. Source: `docs/faq.mdx`.

Quickstart's upload list is documents (PDF, DOC, DOCX, TXT, MD, MDX), tables, pictures, and slides. Put source you want retrieved into Markdown or text if the dataset will not take the original extension.

For a one-off question about a single file, the Agent `Begin` component can take an upload for that run only. Those files are not parsed into a knowledge base and are limited by the model context. Source: `docs/guides/agent/agent_workflow/basic_component.md`, note under the Begin section.

## 4. Use an agent, not only chat

Chat is enough for question answering over a dataset. An agent is the workflow canvas: branches, tool calls, and code execution. Source: `docs/guides/agent/agent_overview.md`.

On that PC, open **Agent** → **Create agent**. Source: `docs/guides/agent/creation_and_management.md`.

| Goal | What to create |
| --- | --- |
| Explain or answer from your docs and notes | Template **Knowledge Base Q&A**. Point it at the dataset. |
| Generate and run Python over a dataset | Template **Data Analysis** (`agent/templates/data_analysis_beginner_assistant.json`). Its prompt tells the model to use `CodeExec` for calculations. This needs the sandbox in the next section. It analyzes data. It does not edit a repository. |
| A smaller custom helper | Blank agent. Keep `Begin`. Add an **Agent** component, select the chat model, and add **Retrieval** as a tool only if it must look up the dataset by itself. |

The Agent component is the LLM node: it reasons, calls tools, and returns text. Add tools only when the prompt needs them. Tool calls, sub-agents, and extra reflection rounds make each reply slower, which matters more on a CPU model. Source: `docs/guides/agent/agent_workflow/basic_component.md`, "Agent Component" and "Tools and Sub-Agents".

When the agent follows a retrieval node, the user prompt should cite the retrieved text, for example answering `/sys.query` from `/Retrieval_0.formalized_content` and saying when the dataset does not contain the answer. Same file, "Prompt Configuration".

Save, then on the canvas click **Run**, enter a test question, and check each component's result. Source: `docs/guides/agent/understand_the_canvas.md`, "Save & Run".

The canvas picker calls the code node **Code**. The sandbox admin guide and the Data Analysis template call the same capability **CodeExec**. The **GitHub** tool searches public repositories by popularity. It does not change a local checkout. The **Compiler** template is an ingestion step that builds knowledge artifacts, not a source-code compiler. Sources: `docs/guides/agent/agent_workflow/tool_components.md`; `docs/guides/knowledge_compilation/overview.md`.

## 5. Enable code execution only if the agent must run code

Q&A and code generation do not need this. The **Code** component and the Data Analysis template do.

The Code component runs Python or JavaScript for processing, conversion, calculation, and file generation. It requires a sandbox. Source: `docs/guides/agent/agent_workflow/data_manipulation_components.md`, "Code Component". Source: `docs/administrator/configurations/sandbox_quickstart.md`.

On that Linux PC:

1. Install gVisor from https://gvisor.dev/docs/user_guide/install/.
2. In `docker/.env`, set `SANDBOX_ENABLED=1` and append `sandbox` to `COMPOSE_PROFILES`. Set `SANDBOX_EXECUTOR_MANAGER_API_TOKEN` to a shared secret (`openssl rand -hex 32`). The same value is injected into RAGFlow and the executor. If it is left empty, `/run` answers HTTP 503. Source: `agent/sandbox/executor_manager/services/auth.py`; `docker/docker-compose-base.yml`.
3. Add `127.0.0.1 es01 infinity mysql minio redis sandbox-executor-manager` to `/etc/hosts`.
4. Pull or build the sandbox base images, then start Compose again. Pull path:

```bash
docker pull infiniflow/sandbox-base-python:latest
docker pull infiniflow/sandbox-base-nodejs:latest
docker tag infiniflow/sandbox-base-python:latest sandbox-base-python:latest
docker tag infiniflow/sandbox-base-nodejs:latest sandbox-base-nodejs:latest
```

5. Open **Admin > Sandbox Settings**, choose `self_managed`, save, and test the connection.

`self_managed` is the default and runs code in Docker containers under gVisor. No GPU is involved. `local` runs code as a process on the same machine and is documented for trusted development only. Cloud providers (`tenki`, `e2b`, `aliyun_codeinterpreter`) need outbound network and an API key, not a GPU.

Sandbox limits, from `agent/sandbox/README.md`:

- Runner containers use `--network none` unless `SANDBOX_CONTAINER_NETWORK=bridge`.
- Python is AST-checked before it runs. File operations and subprocess calls are rejected.
- Each container's default memory cap in `.env` is `SANDBOX_MAX_MEMORY=256m`, and the default timeout is 10 seconds.

So the agent can run a snippet and return stdout or an artifact. It cannot clone a repo, edit project files, or run the test suite on the host.

## What to skip

- `DEVICE=gpu`, the `tei-gpu` profile, and `tei-cpu` with the default 25 GB embedding model.
- IPEX-LLM's published steps, which pin the model to an Intel GPU.
- DeepDoc for plain-text sources.
- A local 7B-or-larger chat model on a 16 GB box that is already running Elasticsearch, MySQL, MinIO, and RAGFlow. The docs do not give a combined RAM budget; they only set 16 GB as the RAGFlow floor and recommend the 3B `llama3.2` as the local starting chat model.
- Treating the agent as an IDE. Cursor on this Mac, and the RAGFlow agent on that PC, are separate tools and do not need a connection.
