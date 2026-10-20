# Development and validation

For installation and daily chat use, start with the [README](../README.md). Detailed configuration and commands are in the [reference](reference.md).

## Layout

The plugin contains two skills and ordinary Python scripts. There is no background MCP server or custom UI.

| Location | Responsibility |
| --- | --- |
| `skills/milvus-knowledge-base/SKILL.md` | Host instructions for document import, updates, retrieval, citations, and deletion |
| `skills/milvus-inspect/SKILL.md` | Host instructions for connection and collection inspection |
| `scripts/milvus_kb.py` | Knowledge-base command entry point |
| `scripts/milvus_inspect.py` | Read-only inspection command entry point |
| `scripts/mvbot_common.py` | Shared connection, embedding, validation, and write guards |
| `scripts/kb_documents.py` | Document parsing and chunking |

Both entry points use PyMilvus `MilvusClient` directly and close the connection on exit. Milvus stores document metadata, text, and vectors; original files remain at their source. BM25, filtering, and hybrid fusion run natively in Milvus. Import verification and retrieval request Strong consistency; ordinary writes do not flush.

## Local checks

Run these from the repository root, using its `.venv`, `pyproject.toml`, and `uv.lock`:

```bash
uv sync --locked
uv run scripts/validate_plugin.py
uv run ruff check scripts tests
uv run ruff format --check scripts tests
uv run pytest -m 'not e2e'
git diff --check
```

The validator checks the manifest, skills, command references in the README/skills, and Python syntax. It does not validate host installation or commands in `docs/`. For documentation-only changes, run the validator and `git diff --check`, and check documented options against the parsers; no new tests are needed.

## Backend and embedding tests

Tests use a fresh temporary Lite path unless `TEST_MILVUS_URI` selects a server/Cloud backend. They create unique collections and clean up only their own collections. Set `TEST_MILVUS_TOKEN` privately in the executing process environment if authentication is required.

```bash
TEST_MILVUS_URI=http://localhost:19530 uv run pytest -m 'not e2e'
```

Use a reachable test server; `localhost` refers to the computer running pytest. Serialize commands sharing a Lite path and imports into the same collection on every backend.

Real embedding E2E is opt-in and can incur API charges. Supply an API key through the process environment, without printing it. To test with OpenAI:

```bash
RUN_EMBEDDING_E2E=1 EMBED_PROVIDER=openai EMBED_MODEL=text-embedding-3-small uv run pytest
```

The E2E uses all four file formats and covers retrieval, source headings/pages, updates, and deletion. Regression cases use deterministic vectors for interrupted imports, document identity, filters, and citations. CLI tests cover Lite close/reopen and execution outside the repository.

Recorded Linux/Python 3.12 validation passed 14 tests per backend with Lite 3.2.2 and Server 3.0.0 (`milvusdb/milvus:v3.0.0`), using PyMilvus 3.0.2 and real OpenAI `text-embedding-3-small` embeddings. This is prior script-test evidence. macOS, Zilliz Cloud, and Grok Bot/Cursor GUI installation, secret propagation, file access, and citation rendering remain unverified end to end. Report the actual backend versions and scope of each new test run.
