# MacBook run (November 22, 2026)

Both walkthroughs used this MacBook (macOS 27.2.1, Apple silicon), Node.js 26.13.0, npm 11.20.0, TrueForge 0.185.1, and `openai/gpt-5-4-mini`. The owner connected OpenAI through the TrueForge UI.

## Browser quickstart

A fresh install in a temporary project opened directly to **Settings → Models** on port 18794. For the live model calls, the test used the owner's previously configured demo instance on port 8790. From **New Chat**, the tester selected **gpt-5-4-mini** and sent the article's two CSV prompts. The replies were `PENCIL-4` and `PENCIL-9,5`.

The demo server received SIGINT and came back on port 8790 with its original SQLite path. The process listening on the port changed from PID 24429 to PID 63725 and the API docs returned HTTP 200. From the home page, selecting the CSV entry under **Chat History** reopened the same session and both replies. **Sessions** also listed the exchange with two turns. The [screenshot](new-chat-after-restart.png) was captured after reopening the chat. The [turn records](new-chat-turns.json) were read back from the restarted server. The second record's `previous_turn_id` matches the first record's `id`.

The install started at 21:38:27 Pacific. The reopened conversation was verified at 21:41:24, **2 minutes 57 seconds** later. That span includes the clean install, browser checks, both model calls, and the restart, with an already connected provider for the model calls. The article gives ten minutes as a planning estimate with Node and a key ready. In the earlier saved-agent run below, the elapsed time from creation of the project directory (19:50:06) to a completed reply after restart (19:54:00) was about four minutes, including provider setup by the owner.

## API walkthrough and initial provider setup

| Step | Result | File |
| --- | --- | --- |
| `npm install --save-exact @truefoundry/trueforge@0.185.1` | exit 0 in 8 s; npm 11.20 skipped the `protobufjs` postinstall script (no effect seen) | `install.log` |
| Start server (`HOST=127.0.0.1 SQLITE_PATH=… npx --no-install trueforge`) | listening on 127.0.0.1:8790, 3 controller loops | `server.log` |
| `GET /api/v1/docs` | 200 in ~4 ms | — |
| Local sandbox fallback | unavailable: `/usr/bin/python3` is the Xcode shim; dyld load of `Python3` blocked by the sandbox | `server.log` |
| `POST /api/v1/agents` before a provider exists | 422 `Unknown model "openai/gpt-5-4-mini" — provider not configured` | — |
| Owner adds OpenAI key in Settings → Models | provider `openai` connected, 16 models | — |
| `POST /api/v1/agents` | success, body wrapped in `data` | `create-agent-response.json` |
| `GET /api/v1/agents/{id}` | server added `context_management.compaction`, `large_tool_response`, `sandbox.file_downloads` | `saved-agent.json` |
| `POST /api/v1/agents` again | 409 `Agent name already exists` | — |
| `POST /api/v1/sessions` | session created | `create-session.json` |
| Turn 1 (startup log question) | done, 3.19 s server-side, 289 in / 386 out | `turn1.json` |
| Turn 2 (Redis follow-up) | done, 0.93 s, 691 in (messages 423 = 21 + 386 + 16) | `turn2.json` |
| Stop server, start again | back in ~3 s; agent, provider, both turns still there | `server-restart.log` |
| Turn 3 ("what did I ask first?") | done; model answered with the agent instructions, not the first user message | `turn3.json` |
| Turn 4 (login follow-up) | done, 1.0 s, correct line quoted, 799 in | `turn4.json` |
| `GET /api/v1/sessions/{id}/turns` | 4 turns, unbroken `previous_turn_id` chain, `cache_read_tokens` 0 on all | `turns-list-final.json` |

Screenshots `01`–`05` were taken from the running UI on this Mac. `openapi.json` is the server's own spec (`GET /api/v1/openapi.json`).

## DeepWiki connector (November 23, 2026)

On the same 8790 server, **Settings → Connectors → deepwiki → Connect** added the connector with no auth. `GET /api/v1/settings/mcp-servers` then listed `deepwiki`, and `GET /api/v1/mcp-servers/deepwiki/tools` returned `ask_wiki_question`, `read_wiki_contents`, and `read_wiki_structure`. In a new chat with **Tools → Connectors → deepwiki** ticked and `gpt-5-4-mini` selected, the article's prompt was sent at 13:33:45Z. The turn finished at 13:34:06Z (about 21 s) with 3 tool calls: `list_tools`, `get_tool_info` for `ask_wiki_question`, and `call_tool` → `ask_wiki_question` with `repoName: expressjs/express`. The reply was `Express supports Node.js **18** minimum, as stated in package.json (engine >= 18).` `npm view express engines` printed `{ node: '>= 18' }` (express 5.3.0). Turn metrics: 7,042 input tokens, 260 output tokens.

Files: `07-deepwiki-connected.png`, `06-deepwiki-tools-menu.png`, `08-deepwiki-chat.png`, `deepwiki-turn-events.json` (tool calls and responses, trimmed to 400 characters each).

## Numbers used in the rewritten article (read back November 23–24, 2026)

- CSV chat turns (`GET /api/v1/sessions/01m83ep6qv61557d8vx35ey09v/turns`): turn 1 took 2.4 s with 1,516 input / 19 output tokens; turn 2 took 2.3 s with 1,545 / 57. One model call each.
- DeepWiki turn: 4 `model.message` events (3 tool calls plus the final answer), 7,042 input tokens.
- Third turn in the CSV chat, sent by the owner in the UI: "where is the csv stored ?" Reply began "It isn't stored anywhere persistent". A read-only `grep -a -c "PENCIL-9,3"` found the text in `~/trueforge-starter/data/db.sqlite` (2) and `db.sqlite-wal` (9).
- Connector catalog (`GET /api/v1/catalogs/mcp-servers`): 18 entries, 5 with no auth (deepwiki, exa, parallel-web, you, keenable), 13 needing OAuth or an API key.
- The `EADDRINUSE` screenshot (`09-eaddrinuse-mac.png`) is the owner's terminal, cropped to the window. The old server (PID 63725) was still holding port 8790.

## One-command start (November 23, 2026, 23:26 local)

From an empty scratch folder, `PORT=8791 npx --yes @truefoundry/trueforge@0.185.1` was listening in 9 s on `http://localhost:8791` (lsof: `[::1]:8791`), with `Standalone mode: sqlite at /Users/rikin/Library/Application Support/trueforge/db/db.sqlite`. PORT was only changed because the old 8790 server was still running. After SIGINT and the same command again, it printed the same sqlite path and `GET /api/v1/sessions` returned the 25 sessions already in that database. The test server was then stopped. The source confirms the defaults: `DEFAULT_HOST = "localhost"`, `DEFAULT_PORT = 8790`, and the database goes under `envPaths("trueforge").data` unless `SQLITE_PATH` is set. The article's chats and screenshots were made on the 8790 server, which used an explicit `SQLITE_PATH`. It's the same version and code path.

## Cost figures

Prices come from OpenAI's gpt-5.4-mini model page, read November 23, 2026: $0.75 per 1M input tokens, $0.075 per 1M cached input, $4.50 per 1M output (and $2.50 per 1M input for GPT-5.4). Cost = (input − cached) × 0.75/1M + cached × 0.075/1M + output × 4.50/1M, using each turn's `state.metrics`:

| Turn | Input | Cached | Output | Cost |
| --- | --- | --- | --- | --- |
| CSV question | 1,516 | 0 | 19 | $0.00122 |
| CSV follow-up | 1,545 | 1,408 | 57 | $0.00046 |
| DeepWiki question | 7,042 | 4,608 | 260 | $0.00334 |

The harness comparison table in the article (TrueForge, Claude Managed Agents, deepagents) is quoted from TrueFoundry's "Best Open Source Agent Harness" post, not measured here.

## Not done

Homebrew Python was not installed, so whether it fixes the sandbox fallback is untested. No sandbox providers were used. The server was **not** restarted after DeepWiki was added, so whether the connector and the DeepWiki chat survive a restart is untested here (the article only claims restart persistence for the CSV chat).
