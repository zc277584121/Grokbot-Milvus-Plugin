# Milvus document knowledge base

Ask questions about your own documents in **Grok Bot or Cursor chat**, and get answers that identify the supporting sources. Import Markdown, TXT, PDF, and DOCX files, search across them, and refresh the knowledge base when your files change.

Once set up, you can ask:

> Import the documents in `/absolute/path/to/handbook` into `team_docs`.

> According to our handbook, how do we handle a failed deployment? Cite the source file and section.

> I updated the handbook. Re-import the same folder into `team_docs`, then summarize the updated procedure with sources.

Your chat app is the interface. It follows the plugin's instructions, runs its Python scripts, and writes an answer from the retrieved passages. There is no separate plugin window or installation wizard. You can also ask it to check the Milvus connection or list your knowledge bases.

## What you need

Two services do different jobs:

- **Milvus stores your searchable knowledge base.** It holds extracted text, source information, and numerical representations of the text called vectors. A *collection* is a named knowledge base, such as `team_docs`. Your original files stay where they are. The beginner setup uses **Milvus Lite**, which stores data on the execution computer without a separate database server or an extra SQLite/PostgreSQL service.
- **An embedding service makes those vectors.** This lets search match meaning as well as words. This guide uses OpenAI's `text-embedding-3-small`: document passages and search questions are sent to its API. You need your own API key and API access; usage can incur charges. Your Grok Bot or Cursor chat model and subscription are separate from this service. The plugin includes no model.

Have **Python 3.10+, [uv](https://docs.astral.sh/uv/), and Git** available on the computer that will execute commands, and allow your host to run shell commands and read your documents. The commands below use Bash or Zsh.

The beginner path is local Milvus Lite with OpenAI embeddings. For an already installed Ollama model or a remote Milvus/Zilliz Cloud database, see [alternative services](docs/reference.md#alternative-services).

**Validation status:** Linux script tests have passed with Milvus Lite 3.2.2 and Milvus Server 3.0.0, using PyMilvus 3.0.2 and real OpenAI embeddings. macOS, Grok Bot/Cursor GUI workflows, and Zilliz Cloud have not been verified end to end.

## 1. Put the plugin where your host can use it

### Cursor on your computer

For a new local source installation, run the following on the computer where Cursor executes commands. If `~/.cursor/plugins/local/milvus` already exists, check that it is this repository and reuse it; skip the clone and do not overwrite it.

```bash
mkdir -p "$HOME/.cursor/plugins/local"
git clone https://github.com/zilliztech/Grokbot-Milvus-Plugin.git "$HOME/.cursor/plugins/local/milvus"
uv sync --locked --project "$HOME/.cursor/plugins/local/milvus"
```

In Cursor, run **Developer: Reload Window** from the command palette. Open **Customize** and confirm that `milvus-knowledge-base` and `milvus-inspect` appear. The repository includes the required `.cursor-plugin/plugin.json`. Local imports can be restricted by your administrator; symlinks pointing outside the local plugin folder are skipped. This follows [Cursor's local plugin instructions](https://cursor.com/docs/plugins#test-plugins-locally); it does not require a marketplace listing.

### Grok Bot

Use the [manual source setup](docs/reference.md#grok-bot-manual-source-setup) to clone this repository on the computer that actually runs Grok Bot commands, then ask it to read `skills/milvus-knowledge-base/SKILL.md` and run the scripts. No arbitrary-repository import flow or marketplace listing has been verified for this plugin.

Grok Bot normally uses a cloud computer with persistent storage under `/workspace`; local execution is a separate option. A path or `localhost` on your laptop refers to a different computer from the cloud host. Follow the path adjustments in the reference before continuing. See [Grok Bot's computer documentation](https://docs.x.ai/grok-bot/computer-and-apps).

## 2. Configure storage and embeddings once

Keep your database and private configuration **outside the plugin installation**. On a local computer, this guide uses `~/.local/share/milvus-plugin/knowledge.db` and `~/.config/milvus-plugin/env.sh`. `$HOME` expands to an absolute path, so the database stays the same when the working directory changes.

Create the directory and private file:

```bash
mkdir -p "$HOME/.local/share/milvus-plugin" "$HOME/.config/milvus-plugin"
touch "$HOME/.config/milvus-plugin/env.sh"
chmod 600 "$HOME/.config/milvus-plugin/env.sh"
```

Open `env.sh` yourself in a private text editor and add the following, replacing the key placeholder there. If the file already has settings, edit those entries. **Never paste API keys into chat or commit this file.** This is a user-managed shell file, not a plugin secret store.

```bash
export PLUGIN_ROOT="$HOME/.cursor/plugins/local/milvus"
export MILVUS_URI="$HOME/.local/share/milvus-plugin/knowledge.db"
export MILVUS_DB=default
export EMBED_PROVIDER=openai
export EMBED_MODEL=text-embedding-3-small
export EMBED_ENDPOINT=https://api.openai.com/v1
export EMBED_API_KEY='REPLACE_WITH_YOUR_OPENAI_API_KEY'
```

The scripts read command-line options and environment variables only. They do **not** automatically load this file or a `.env` file. Source it in the **same shell invocation as each command**, as shown below. An export in another terminal, or an earlier isolated host command, does not configure the host's next command.

```bash
source "$HOME/.config/milvus-plugin/env.sh"
uv run --project "$PLUGIN_ROOT" "$PLUGIN_ROOT/scripts/milvus_inspect.py" health
```

Look for `"connected": true` and the intended absolute database path. Opening a new Lite path initializes storage there. This checks the database connection; the first import below checks your embedding configuration.

## 3. Import the samples and ask a first question

The included `samples/kb` folder contains an English explanation of hybrid search and reciprocal rank fusion (RRF), plus a Chinese introduction to Milvus. First preview what will be imported:

```bash
source "$HOME/.config/milvus-plugin/env.sh"
uv run --project "$PLUGIN_ROOT" "$PLUGIN_ROOT/scripts/milvus_kb.py" ingest "$PLUGIN_ROOT/samples/kb" --collection team_docs --dry-run
```

You should see a `[dry-run]` line for each of the two files with its title and chunk count. This preview does not call the embedding service or create a collection. Now import and query, waiting for each command to finish:

```bash
source "$HOME/.config/milvus-plugin/env.sh"
uv run --project "$PLUGIN_ROOT" "$PLUGIN_ROOT/scripts/milvus_kb.py" ingest "$PLUGIN_ROOT/samples/kb" --collection team_docs
uv run --project "$PLUGIN_ROOT" "$PLUGIN_ROOT/scripts/milvus_kb.py" query "How does hybrid search combine semantic and keyword matches, and what does RRF do?" --collection team_docs --json
```

On a fresh `team_docs` collection, the import prints `[created]`, an `[added]` line for each file, and a summary with `added: 2` and a positive `chunks_written` count. A successful query returns JSON with `found: true` and a nonempty `results` list. Read the returned `text`: the English sample explains dense semantic matching, BM25 keyword matching, and RRF combining ranked results. Each result includes a citation number and source path; rankings and scores can vary.

**The command returns passages. Your chat host writes the final answer.** In a new conversation, give it this setup instruction, using your actual configuration path:

> Use the Milvus knowledge-base skill and collection `team_docs`. Before every plugin command, source `~/.config/milvus-plugin/env.sh` in the same shell invocation as the command. Do not print the file, keys, or environment, or enable shell tracing. Use the configured `PLUGIN_ROOT` with `uv run --project` and absolute script paths. Run commands sequentially.

Then ask:

> Search `team_docs`: how does hybrid search combine semantic and keyword matches, and what does RRF do? Answer from the retrieved text and cite the source files.

The host should explain the retrieved evidence, use citations such as `[1]`, and identify the corresponding source file and section or PDF page when available. If the passages do not support an answer, it should say so.

## Use it in everyday chat

Give the host an **absolute path accessible on its execution computer**. Keep your own documents outside the plugin folder, and repeat the configuration instruction above in each new conversation.

| What you want | What to ask |
| --- | --- |
| Add your documents | “Preview `/absolute/path/to/handbook`, then import it into `team_docs`.” |
| Find an answer | “Search `team_docs` for our release checklist. Explain the steps and cite the relevant sections.” |
| Check what is stored | “List the documents in `team_docs`.” |
| Check the connection | “Use the Milvus inspection skill to check the connection and list collections.” |
| Update documents | “Re-import `/absolute/path/to/handbook` into `team_docs`; I edited the files in place.” |
| Remove a document | “List the document IDs in `team_docs` and preview removal of the old handbook. Show me the affected document before deleting it.” |

Updates happen only when you **explicitly re-import**; there is no folder watcher. Reusing the same file paths preserves document identity, skips complete unchanged documents, and replaces changed content. For files that move between folders or computers, use the [portable path options](docs/reference.md#document-identity-and-moved-files).

Deleting a source file does not remove its imported text. The host must use `delete-doc`, preview the affected chunks, then apply an authorized deletion with `--execute --confirm COLLECTION`. See [deletion commands](docs/reference.md#inspect-and-delete-documents). There is no collection-drop UI.

Run imports into one collection one at a time, including on Server. All commands sharing one Lite database path must run sequentially. If an import is interrupted, retry it with the same inputs; a multi-batch update can remain partially visible until the retry finishes.

## If something goes wrong

| Symptom | What to check |
| --- | --- |
| The skill is missing in Cursor | Check the local installation path and manifest, reload the window, and check Customize and your administrator's local-import policy. |
| It works in your terminal but fails in chat | Ask the host to source your private config **in the same shell invocation** as every command. Check that it is running on the intended computer. |
| “No embedding model,” authentication error, or HTTP 429 | Check the private file's provider/model, API key, and API quota. `EMBED_API_KEY` takes precedence over `OPENAI_API_KEY`. Do not share the key in chat. |
| The collection is missing or looks empty | Check the absolute `MILVUS_URI` and collection name. A different Lite path opens a different database. Run the import before querying. |
| A file is missing or a PDF has no extractable text | Check the path on the execution computer and the file format. Scanned PDFs need OCR before import. |
| An answer uses old content | Re-import the same source paths. Removing or editing a local file alone does not update Milvus. |
| “Embedding provider/model differs” | Restore the original model settings, or import into a new collection for the new model. |
| Database busy or import interrupted | Wait for other commands to finish, then retry sequentially with the same inputs. |

For Ollama, remote databases, filters, and other command options, see the [reference](docs/reference.md). To update the plugin itself, see [source-install updates](docs/reference.md#update-a-source-installation). Contributors can use the separate [development guide](docs/development.md).
