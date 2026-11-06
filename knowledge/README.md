# knowledge plugin

A PM Bridge plugin that gives personas a knowledge base: hybrid search (BM25 keywords + Ollama
embeddings) over YouTrack articles and local folders, as **one cross-project pool**. It is a standard Claude
Code plugin: one MCP stdio server plus one skill. It was originally part of the PM Bridge app.

Pure JavaScript: the vector search is a brute-force scan over a `Float32Array` (no vector database,
no native module). Embeddings are optional: without Ollama every tool still works, keyword-only.

```
.claude-plugin/plugin.json   name, version, description, author, userConfig
.mcp.json                    server "knowledge": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js, env from ${user_config.KEY}
skills/kb-search/SKILL.md    the skill knowledge:kb-search
src/ test/                   the server and its tests
```

## Build and run

```sh
npm install
npm run build   # tsc -> dist/server.js (not committed)
npm test        # builds, then runs unit tests and the stdio end-to-end test
```

The end-to-end test spawns the built server exactly as `.mcp.json` describes it, filling the
`${user_config.KEY}` placeholders like the host (`${CLAUDE_PLUGIN_DATA}` is a temporary folder), against local stub
YouTrack and Ollama servers; unit tests start the server in memory with a fake environment.

The host runs the server from `.mcp.json`. `dist/` is not committed: run
`npm install && npm run build` once after checking out.

## Configuration

Read once at startup from the process environment. `.mcp.json` maps each variable to a
`${user_config.KEY}` placeholder that the host (or Claude Code) fills from the owner's settings;
`plugin.json` `userConfig` declares each KEY (title, description, `required`, `sensitive`). No owner variable is required.

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `ollama_url` → `OLLAMA_URL` | text | no | default `http://127.0.0.1:11434` |
| `embed_model` → `KNOWLEDGE_EMBED_MODEL` | text | no | default `bge-m3` (the old app's default) |
| `folders` → `KNOWLEDGE_FOLDERS` | text | no | absolute folder (or file) paths, comma or newline separated |
| `extensions` → `KNOWLEDGE_EXTENSIONS` | text | no | file extensions indexed in folders, e.g. `md,txt`; default: common text and code files |
| `youtrack_base_url` → `YOUTRACK_BASE_URL` | text | no | YouTrack server for articles; needs `YOUTRACK_TOKEN` too |
| `youtrack_token` → `YOUTRACK_TOKEN` | secret | no | YouTrack permanent token, sent only to `YOUTRACK_BASE_URL` as `Authorization: Bearer` |

Invalid config does not stop the server: every tool call returns an error that names the variable
and never echoes a secret. A placeholder left unexpanded (`${user_config.youtrack_token}`) counts as unset.

The index is kept in the plugin's own data folder: `.mcp.json` sets
`KNOWLEDGE_INDEX_DIR` to `${CLAUDE_PLUGIN_DATA}/index` (the official plugin data folder, which the
host creates 0700 and Claude Code keeps under `~/.claude/plugins/data/`; not an
owner setting). It must be an absolute path and is created on reindex. Without it every tool fails
with an error that names `KNOWLEDGE_INDEX_DIR`. An index in an old `indexDir` folder is not
migrated or read: run `reindex` once (the old folder can be deleted by hand).

## Tools

| tool | read/write | what it does |
|---|---|---|
| `search(query, limit?, source?)` | read | ranked chunks: `ref`, source, title, section, project, location (article URL or file path), score, snippet, `why` (rank per path, matched terms); `mode` is `hybrid` or `lexical`, `note` explains a fallback or an empty result |
| `get_document(ref, maxChars?)` | read | one document's full indexed text (redacted) by a search `ref` (`<docId>#<n>`) or doc id, plus the referenced chunk |
| `index_status()` | read | counts per source, build time, embedding model and dims, whether Ollama is available now, notes on what to do |
| `reindex(source?)` | write | rebuild from the configured sources (`youtrack`, `folder`, or both); incremental, idempotent, writes only `KNOWLEDGE_INDEX_DIR` |

Read tools set `annotations.readOnlyHint = true`; `reindex` sets `readOnlyHint: false` and
`idempotentHint: true`. Endpoints used for articles (same as the youtrack plugin's
`search_articles` / `get_article`): `GET articles?fields=...&$top=-1` and `GET articles/{id}`.

## Rules (enforced in the plugin)

- **One pool.** Every article and file is searchable from any project; ranking decides relevance.
  The project is display metadata only.
- **Keyword fallback, said out loud.** If Ollama is unreachable or the model is not pulled, reindex
  writes a keyword-only index and search answers with BM25 only; the result says so in `note`. The
  query is embedded with the model the index was built with, never a different one.
- **Nothing found is an answer.** The dense path has a similarity floor (0.45, measured for bge-m3)
  so an unrelated query returns no hits instead of confident noise.
- **Secrets are not indexed.** Document text is redacted (tokens, keys, JWTs, PEM blocks,
  `password=`-style assignments in English and Chinese, basic auth in URLs) before it is stored,
  chunked or embedded, and again on output. The folder walker never reads `.env`, key material,
  credential files, `settings.json`, `.git`, `node_modules` or build output, never follows a
  symlink out of a configured folder, and never indexes the index folder itself.
- **A failing source keeps its documents.** If YouTrack cannot list articles, the previous articles
  stay; a body that cannot be fetched keeps its previous text. A source that is no longer configured
  is removed from the index.
- **No credentials in output.** Errors are `YouTrack HTTP <status>: <server message>` or a network
  error code; URLs with credentials, headers and the token never appear, and the token is scrubbed
  from every result.

## Index folder

Everything in `KNOWLEDGE_INDEX_DIR` is disposable (deleting it costs a reindex): `manifest.json` (written last),
`docs.json`, `chunks.jsonl`, `texts.jsonl` (redacted full text per document), `vectors.f32` +
`vectors.map.json` (when embedded), `postings.json` (only above 30 000 chunks).

## Skill

- `skills/kb-search/SKILL.md` (`knowledge:kb-search`) - when and how to search, citing sources, never
  answering beyond what was retrieved.
