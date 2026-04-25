# Tool surface

Why these specific tools, in this groupings, and how each one is meant to be
chosen over the available shell equivalent. Companion to `crates/tui/src/prompts/agent.txt`.

## Design stance

- **Dedicated tools over `exec_shell` whenever the dedicated tool returns
  structured output.** Bash escaping is error-prone and platform behavior
  varies (GNU vs BSD `grep`, `rg` is not always installed). Structured
  output also frees the model from re-parsing free-form text.
- **`exec_shell` for everything else.** Build, test, format, lint, ad-hoc
  commands, anything platform-specific. We don't try to wrap the long tail.
- **Drop tools that don't beat their shell equivalent.** Two-tool aliases
  for the same backing operation are a model trap — the LLM will alternate
  between them and the cache hit rate suffers.

## Final surface (v0.5.1)

### File operations

| Tool | Niche |
|---|---|
| `read_file` | Read a UTF-8 file. PDFs auto-extracted via `pdftotext` (poppler) when available; `pages: "1-5"` slices large docs. |
| `list_dir` | Structured, gitignore-aware listing. Preferred over `exec_shell("ls")`. |
| `write_file` | Create or overwrite a file. |
| `edit_file` | Search-and-replace inside a single file. Cheaper than a full rewrite. |
| `apply_patch` | Apply a unified diff. The right tool for multi-hunk edits. |

### Search

| Tool | Niche |
|---|---|
| `grep_files` | Regex search file contents within the workspace; structured matches + context lines. Pure-Rust (`regex` crate), no `rg`/`grep` shell-out. |
| `file_search` | Fuzzy-match filenames (not contents). Use when you know roughly the name. |
| `web_search` | DuckDuckGo (with Bing fallback); ranked snippets + `ref_id` for citation. |
| `fetch_url` | Direct HTTP GET on a known URL. Faster than `web_search` when the link is already known. HTML stripped to text by default. |

### Shell

| Tool | Niche |
|---|---|
| `exec_shell` | Run a shell command. Foreground or background (`background: true` returns a `task_id`). |
| `exec_shell_wait` | Poll a background task for incremental output. |
| `exec_shell_interact` | Send stdin to a running background task and read incremental output. |

### Git / diagnostics / testing

| Tool | Niche |
|---|---|
| `git_status` | Inspect repo status without running shell. |
| `git_diff` | Inspect working-tree or staged diffs. |
| `diagnostics` | Workspace, git, sandbox, and toolchain info in one call. |
| `run_tests` | `cargo test` with optional args. |

### Task management

| Tool | Niche |
|---|---|
| `todo_write` | Granular per-item progress. |
| `update_plan` | Structured checklist for complex multi-step work. |
| `note` | One-off important fact for later. |

### Sub-agents

`agent_spawn`, `agent_swarm`, `spawn_agents_on_csv`, plus the supporting
tools (`agent_result` / `swarm_result` / `wait` / `send_input` /
`agent_assign` / `agent_cancel` / `resume_agent` / `agent_list` /
`report_agent_job_result` / `swarm_status`). See `agent.txt` for the
delegation protocol.

## Recently consolidated (v0.5.1)

Removed from the prompt as duplicates of equivalent tools (the underlying
dispatchers still resolve them, so existing sessions don't break — they just
no longer pollute the model's tool list):

- `spawn_agent` → use `agent_spawn`.
- `close_agent` → use `agent_cancel`.
- `assign_agent` → use `agent_assign`.

## Why we don't ship a single `bash` tool

Single-`bash` agents (Claude Code's design) are powerful but hand the model
all the foot-guns of shell scripting: quoting, platform divergence,
side-effects from misread cwd, `cd` not persisting between calls, etc. Our
file tools are also significantly cheaper to render in the transcript
(structured JSON-shaped output collapses better than `ls -la` walls of text).

The model can always fall back to `exec_shell` when something is missing.
The dedicated tools just take the common 80% off the shell escape-hatch.
