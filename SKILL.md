---
name: marauder-cli
description: Use whenever a task calls the Marauder CLI (`python -m app.cli`) or the MCP server at /api/mcp, uses or mentions an Agent key, or wires an agent or VM to Marauder - login and whoami, notebooks and files, portfolios and pinned notebooks, revx-book and revx-account, order proposals and approvals, FMP catalog calls, plugin runs - or builds an MCP or ChatGPT app on Marauder tools. Triggers include /marauder-cli, /chatgpt-apps, "marauder" commands, "notebook list", "revx-book", "portfolio --source", "Agent key", "MCP tool".
---

# Marauder CLI

The account CLI acts as the user who owns the Agent key. It is the same key as MCP. The FMP key stays on the server. The CLI does not place broker orders.

Invoke it from `backend/` so `app` imports:

```bash
python -m app.cli --help
python -m app.cli <command> --help
```

A local `marauder` on `PATH` is that module. If it fails with a missing interpreter, point it at this checkout's `backend/.venv/bin/python`, not an iCloud Desktop copy.

Prove the session before you write:

```bash
python -m app.cli whoami
```

`auth` must be `ok`. Then run one read (`notebook list` or `portfolio list`). A 503, `rate_limited`, or `ok: false` is a failure. Do not invent the missing payload.

## Auth

New logins use the Notebook origin:

```bash
printf '%s\n' "$MARAUDER_AGENT_KEY" | python -m app.cli login --base-url https://notebook-marauder.com --key-stdin
```

Do not pass the key in argv. `--key` is refused. Use `--key-stdin` or the prompt.

Credential order:

1. `MARAUDER_WORKSPACE_KEY` plus `MARAUDER_API_URL` (agent VM). This wins over the file.
2. Both `MARAUDER_BASE_URL` and `MARAUDER_API_KEY`, or neither.
3. `~/.config/marauder/credentials` (or `$XDG_CONFIG_HOME/marauder/credentials`), mode `0600`.

Login and logout refuse while those env vars are set. Unset them first.

Old hosts still answer until the user logs in again. `whoami` then sets `host_hint`. Re-login to `https://notebook-marauder.com`. Do not keep using `backend-production-cb99.up.railway.app` or the retired connect host for a new login.

An Agent key cannot read `GET /api/auth/me` (cookie only). `user: null` with `auth: ok` is expected. It is not a failed login.

`mcp-url` prints `{base}/api/mcp`. The `/mcp` alias is not the URL you configure.

## Output

Default is one key-sorted JSON object on stdout. `--pretty` indents. It works before or after the subcommand. `--full` keeps the untrimmed payload. Slim is the default on purpose.

- Exit 0: ok.
- Exit 1: the server or a check said no. JSON `{"ok": false, "error": ...}` on stdout. Read `hint` and do that next call.
- Exit 2: bad command, missing `--confirm`, or not logged in. Plain text on stderr.

`truncated: true` means the answer was cut to 8 KB. Narrow the call. Do not retry the same wide call.

## Writes

`--confirm` means the user already approved that write. Do not add it to skip the ask.

`--dry-run` prints method, path, and body and sends nothing. It satisfies the CLI confirm check, except `smaug playbook set` (confirm is the real write) and `notebook source reimport-x` (`--dry-run` forces list-only).

Missing confirm exits 2: `Pass --confirm after the user approved.`

Client plugins that require `--confirm`: `files.write`, `notebook.source.add_url`, `portfolio.manual.draft`, `notify.telegram.ticket`, `notify.telegram.send`.

`plugin run order.submit` never leaves the client. Do not place, withdraw, or transfer. `order.approval.send` reaches the server and is allowed only under that account's autonomy policy. Do not use it to send a ticket the user did not approve. `ibkr-book ticket --confirm` saves a draft and never sends.

## Ids

Notebook and cloud file args take a unique name or a positive int. A digit string is always the int, never a name. If the name matches more than one row, pass the int.

These are ints only: `source_id`, `folder_id`, `rev`, `--parent`, file `--folder`, import `--folder`, `--source-ids`, `--owner-bot`.

`run_id` is an opaque token. It is not a notebook id and not a Smaug `thread_id`.

File CAS field on slim JSON is `document_updated_at`. Pass it back as `--document-updated-at` on `file restore` and on a workflow DAG replace. `file replace` and `file append` will GET a token if you omit it. That is a race. Hold the token from the get you just read.

Do not call `notebook.write`. Body writes are `notebook file create|replace|append` or `notebook local create|replace|append`. `--args` for file and local commands is raw markdown. `--args` for agent, artifact, workflow, and chart is JSON, or `-` for stdin.

Do not overwrite `--out`. An existing path is refused.

## When the server and this CLI disagree

This CLI calls MCP tool names such as `fmp_mcp.quote`. A deployed server can still expose `fmp.quote` in `tool list` and `fmp_mcp.quote` only in `plugin list`. `fmp quote AAPL` then returns `Unknown tool: fmp_mcp.quote`.

Do not retry the same verb. Read the catalog on that server:

```bash
python -m app.cli tool list
python -m app.cli tool describe fmp.quote
python -m app.cli plugin describe fmp_mcp.quote
```

Then call the name that server has:

```bash
python -m app.cli tool call fmp.quote --args '{"ticker":"AAPL"}'
python -m app.cli plugin run fmp_mcp.quote --args '{"endpoint":"quote","symbol":"AAPL"}'
```

`plugin describe` is the schema for `plugin run`. `tool describe` is the schema for `tool call`. Follow `hint`. A rate limit or 503 is the provider saying no, not an empty quote.

## Commands

Run `<command> --help` before you invent a flag. Top-level names:

```
login logout whoami mcp-url account settings connection support trading
company market fmp etf etf-flows sentiment compare memo research decision
portfolio watchlist ibkr ibkr-book revx-book revx-account revx-fill-wake revx-funding-liqs
notebook
smaug chat bots notifications search events
plugin tool
```

Prove a CLI change against the live command help or a test, not a remembered flag. Before a contract change, name the blast radius. [Blast radius](../build/references/blast-radius.md). [Prove it works](../build/references/principle-prove-it-works.md).

Market reads: `company TICKER`, `fmp quote|earnings|earnings-calendar|economics-calendar|economics-high-impact|crypto-quote`. Calendars need `--from` and `--to`. Earnings calendar `time` is `bmo`, `amc`, or `tba`.

Session status: `market status [SYMBOL ...]` returns one row per market with `state`, `session_date`, `next_open_at`, `closes_at`, and a `line` in the user's time zone. No symbols means the US market. Ask it before saying a move happened "today".

Portfolio reads: `portfolio` (slim current state), `portfolio list|holdings|performance|exposure|pnl|activity|dump`. `--source` is `manual`, `ibkr`, or `revx`. Writes `create|rename|archive|restore|delete|import` need `--confirm`.

`watchlist add|remove|delete` need `--confirm`. `watchlist create` and `rename` do not.

`ibkr sync` and `ibkr flex upload` need `--confirm`. `ibkr-book live` is a read. `ibkr-book ticket --confirm` saves a draft and never sends.

`revx-book refresh`, `revx-account sync`, `revx-fill-wake`, and `revx-funding-liqs refresh` need `--confirm`. They write notebook files. They are not broker order entry. Do not pass a revx argv the CLI does not expose.

`compare get` is the saved snapshot. Starting a run is `plugin run compare.run`, not a `compare run` verb. `memo get` reads a saved memo. `memo run` starts one and does not take `--confirm`. Poll with `memo status`.

`research ask --ticker NVDA` reuses the latest completed run or starts one. A follow-up is `--run-id` plus `--question`.

`decision portfolio` needs `--confirm` when it judges. `decision rank|history|verdict|replay` are reads. No orders.

Notebook verbs that need `--confirm`: `delete`, `folder delete`, `file replace`, `file delete`, `file restore`, `source add-url`, `source upload`, `source add-web-search`, `chart render`, `artifact generate|start|steer|cancel`, `workflow create|update|run|archive|delete|schedule`, `decision create|update|run`, `settings set`, `agent steer|cancel`, `setup`.

`notebook source reimport-x` without `--confirm` lists candidates. With `--confirm` it reimports. `--dry-run` forces the list.

`notebook chat` is a notebook turn. `chat send` and `smaug ask` are Smaug. `chat send` works only for bot `smaug`. Other bots return `can_send: false` with a `hint`.

## Reading chats and messages

Read a chat with `chat open BOT [--last N]` (default 20, oldest first). For `smaug` it is the conversation. For other bots it is the inbox. You get the stored raw markdown (`content` or `body`), not the rendered text. A `**Title**` is a heading the user saw in bold.

```bash
python -m app.cli chat open smaug --last 5
python -m app.cli chat open news --last 5
python -m app.cli chat get news --since today --kind earnings --ticker NVDA --limit 10
python -m app.cli smaug threads get smaug-main --since today --last 10 --full
python -m app.cli chat send smaug "What moved?"
```

- Every `ts` is UTC with `+00:00`. `today`, `yesterday`, and `YYYY-MM-DD` are days in the user's Settings time zone. Both `notifications list` and `smaug threads get` echo the resolved `since` and `until` with the offset.
- A thread message has `channel` (`telegram` when it went through the bot, else the app scope) and `telegram_message_id`.
- An inbox row has `channel` (`telegram` or `in_app`), `delivery` (`delivered`, `failed`, `suppressed`, `pending`, `not_sent`) and `delivered_at`. `delivered_at` is null unless Telegram accepted the message. `delivered_at_source: "row"` means there was no ledger row and the time is when the row was written after the send. `--full` adds `body` and `body_format` (`markdown` or `text`).
- `chat send smaug` waits for the turn and prints `thread_id`, `user_message {id, ts}` and `reply {id, ts, channel, content}`. `--dry-run` sends nothing.

`smaug model` is GET labels (`fast`, `medium`, `high`, `tool`). It does not PUT a model. To use one for a turn, pass `--model` on `smaug ask` or `chat send smaug`.

`smaug memory --tune`, `watch add`, `questions answer`, `feedback`, `playbook set`, `automation create`, `bot create`, `bot run`, `agent spawn|cancel`, and `job pause|resume|cancel|run-now` need `--confirm`. `job` has no `list`. List jobs with `smaug automations`.

`events dest set`, `subscribe`, and `unsubscribe` need `--confirm`. Do not pass `--sender-key` on argv.

`notebook local` is a granted Native root, not a second cloud store. `setup` places the public Mac zip. It does not mint Agent keys. Do not bypass a Gatekeeper failure.

Native commands need the Notebook Mac app. `--dry-run` previews and does not open the app or send the request. Writes below need `--confirm` after the user agrees.

- `notebook device` covers `control`, `preferences`, `provider`, `role`, `folder`, `project`, `permissions`, `computer`, `browser`, `diagnostics`, `update`, and `model`. `preferences set`, `folder grant`, and `update install` need `--confirm`.
- `account status` reads the signed-in account. `sign-in` and `sign-out` need `--confirm`. `account agent-key list` reads names and ids. `create` and `revoke` refuse and point at Notebook Settings. Do not print a key.
- `connection list` and `check` are reads. `add`, `connect`, `disconnect`, `remove`, and `tool-policy set` need `--confirm`.
- `support list` and `support get` read this account only. They do not refresh Linear.
- `trading policy get` is a read. `trading policy propose` and `trading stop` need `--confirm`. Neither places an order.

`sentiment today` is the news-bot pack. `book`, `watchlist`, `vs-portfolio`, and `delta` read the stored sector packet. `notifications list` without `--limit` pages every match, so pass `--since` or `--limit`. `notifications wait` polls until a quiet window and exits 0 with the last batch.

`portfolio snapshot` is the latest stored book view. `compare` takes two to eight portfolios. `tree` shows parents. `sets` lists saved compare sets. Those four are reads.

`smaug bot list` is a read. `smaug bot create` and `smaug bot run` need `--confirm`. `smaug bot workflow` is the same binder as `smaug workflow`.

`settings get` never prints secrets. `--full` keeps sent-history and Smaug memory blobs. Do not paste those into chat. `settings models get` and `settings usage` read account data. Do not change the model from the CLI.

## Agent-friendly shape (gaps to close)

Two patterns from OpenAI's `cli-creator` and Anthropic's `mcp-builder` skills (both Apache-2.0, rewritten here). Neither exists yet. Build them in their own PR, not inside an unrelated change.

- **`doctor --json`.** One read-only command that reports base URL, `host_hint`, auth state, and the auth source category (`workspace_env`, `env`, `config`, `missing`), never the key itself, plus one probe call's result. `whoami` covers most of this today. Adding it means a new top-level command, so update the command block above and the audit test in the same PR.
- **MCP tool annotations.** Every tool in `backend/app/agent_tools/defs/` should declare `readOnlyHint`, `destructiveHint`, and `idempotentHint`, emitted on `/api/mcp` `tools/list`. Reads are read-only and idempotent. File replace is destructive and idempotent. Append, proposals, and sends are neither. A client can then auto-approve reads and ask before destructive calls. Run `python -m app.agent_tools.generate` after editing defs.
- **Errors say the next call.** Keep `hint` on every `ok: false` so an agent can recover without reading source.

## MCP apps

Building a ChatGPT Apps SDK app (MCP server plus widget UI) on top of Marauder tools: read [chatgpt-apps](references/chatgpt-apps/index.md). It is docs-first because the Apps SDK bridge changes often. Tools still come from `backend/app/agent_tools/defs/`, never a parallel catalog.

Index: [chatgpt-apps](references/chatgpt-apps/index.md) is the only reference in this skill.

## Audit

Before you change this skill, run from the repo root:

```bash
python3 backend/tests/test_marauder_cli_skill_audit.py
```

The test fails if `.agents/skills/marauder-cli/SKILL.md` and `.cursor/skills/marauder-cli/SKILL.md` differ, or if a top-level command in the live parser is missing from the command block above.
