# Model roster

Tier → concrete model, per provider. Scout verifies this against the live runtime before dispatch (SKILL.md §1).

⚠ marks a row carried from prior config and **not** verified. Confirm it before passing it as a model string, then drop the mark.

## Dispatch mechanism

**Spawn headless processes.** One process per subagent, model and effort set per process, output redirected to the receipt path. This is what makes the roster below meaningful — an in-session subagent tool takes a model but inherits the session's reasoning effort, so half the tiering is unavailable there.

Headless costs you three things, all manageable: you own concurrency (no built-in cap — batch with `&` and `wait`), each process pays a full cold start, and auth is per-process.

## Anthropic — Claude

```sh
claude -p "<brief>" \
  --model sonnet --effort high \
  --output-format json \
  --session-id "$UUID" \
  > "$RECEIPT_PATH"
```

| Tier | Model | Alias | $/1M in → out | Context |
| --- | --- | --- | --- | --- |
| Mechanical | Haiku 4.5 | `haiku` | 1 → 5 | 200K |
| Judgment | Sonnet 5 | `sonnet` | 3 → 15 | 1M |
| Synthesis | Opus 5 | `opus` | 5 → 25 | 1M |
| Escalation | Fable 5 | `fable` | 10 → 50 | 1M |

`--effort`: `low` `medium` `high` `xhigh` `max`. Effort is the second tiering axis — a judgment-tier model at `low` is often cheaper and better than a mechanical-tier model at `high`, because the smaller model burns turns instead of thinking.

Flags that matter here:

- `--session-id <uuid>` — assign it at dispatch, and retry can `--resume <uuid>` into the same warm context instead of paying a fresh cold start.
- `--output-format json` — the receipt arrives parseable rather than as prose you have to read.
- `--append-system-prompt` — where the scope and return contract go, keeping the brief itself about the task.
- `--max-turns` — a hard stop on a subagent that starts thrashing.
- `--fallback-model <list>` — survives an overloaded tier without failing the unit.
- `--permission-mode` — `acceptEdits` for subagents that must write unattended.
- `--allowedTools` — MCP tools reach headless subagents, but they are **deferred**: allow `ToolSearch` alongside the MCP tool name or the subagent loads no schema and reports the tool as unavailable. Verified — `--allowedTools "mcp__…__whoami"` alone returned "no such tool"; adding `ToolSearch` returned the live answer. That false negative is indistinguishable from an auth failure, so check the allowlist before concluding a server is unreachable.

**`--bare` needs an API key.** It would shrink a subagent's starting context by skipping hooks, auto-memory, and CLAUDE.md discovery, but it forces `ANTHROPIC_API_KEY` and never reads OAuth or keychain — tested on an OAuth login, where it fails with `api_error` before spending a token. It becomes available the moment an API key is exported.

Budget for the cold start instead: a trivial haiku unit measured 26K tokens and ~$0.019 before doing any work, almost all of it CLAUDE.md, skills, and system prompt. Your own floor scales with how much CLAUDE.md and how many skills your setup loads — measure it once. That floor is what the gate in SKILL.md §2 is protecting against.

Pricing cached 2026-06-24 and goes stale without warning — treat the table as ratios for tier decisions, not as a bill. Sonnet 5 carries an introductory 2 → 10 through 2026-08-31.

## OpenAI — Codex

Only usable if `codex` is installed and logged in — check in §1 before you brief anything against these rows. Everything in this skill works with the Claude section alone.

```sh
codex exec "<brief>" \
  -m gpt-5.6-sol \
  -c model_reasoning_effort="high" \
  -C "$WORKDIR" \
  -s workspace-write \
  > "$RECEIPT_PATH"
```

| Tier | Model | Effort |
| --- | --- | --- |
| Synthesis / coordination | `gpt-5.6-sol` | `high` |
| Judgment | `gpt-5.6-luna` ⚠ | `max` ⚠ |

`gpt-5.6-sol` and the effort key are confirmed against `~/.codex/config.toml` (default effort there is `medium`). The luna row is carried from a prior config and unverified. No mechanical-tier entry — add one if a cheap model exists here.

Effort levels: `low` `medium` `high` `xhigh` `ultra` `max`. Note `ultra` sits between `xhigh` and `max` and has no Claude equivalent.

Flags that matter here:

- `-s <read-only|workspace-write|danger-full-access>` — sandbox policy. It restricts **shell commands the model runs**, not the model's own tools: web search works under `read-only`, verified here. What the sandbox does block is a shell command reaching the network or binding a port, so scope shell-based verification accordingly and run anything broader yourself.
- `--skip-git-repo-check` — required outside a git repo, which a scratch run directory is.
- Redirect stdin (`< /dev/null`) or `codex exec` waits on it even with a prompt argument.
- `--output-schema <FILE>` — a JSON Schema for the final response, which enforces the receipt shape rather than requesting it.
- `codex exec resume` — the analogue of `--resume` for retry rounds.
- `--ephemeral` — no session files on disk, for throwaway units.

## Scout checklist

1. `which claude codex` — which mechanisms exist here.
2. For each, confirm the model strings the roster claims. An invalid string fails at dispatch, after the brief is written.
3. Confirm the effort flag and its accepted levels.
4. Fix any row that turned out wrong and clear its ⚠.

## Adding a provider

Append a section in the same shape. Three things are required:

- **The invocation** — the exact command, with prompt, model, effort, and output redirect.
- **A tier mapping** — every tier the provider covers, named by the exact string the CLI accepts. Leave a tier out rather than guessing at it.
- **Sandbox and capability limits** that change what a subagent can verify on its own.

Mark every new row ⚠ until a scout run has dispatched against it. Cost and context are useful for tier decisions but not load-bearing — omit them rather than inventing them.
