# OmniTrunker as a Trunk Recorder Pro interface — the port brief

## What it is

`D:\reporter\web\omnitrunker.html` (2,706 lines, plus `theme-config.js`, `theme-engine.js`, `audio-engine.js`, `auth.js` beside it) is tr-engine's "OmniTrunker" page: a dispatch-wall style view with two panes. Top: **Active Voice Channels**, a table of the calls on the air right now (system, talkgroup, alpha tag, frequency, elapsed time, source unit, mode, a speaker icon per row to listen), with a view toggle (by talkgroup vs by frequency), a system filter, a LISTEN toggle with master volume and per-talkgroup mute list in a settings drawer (talkgroup search, unmute/mute all, auto-pan, column choices), and an event counter. Bottom: **Unit Activity**, a scrolling feed of radio events (buffer size, type filter, auto-scroll, clear). It is fed today by tr-engine's REST (`/api/v1/systems`, `/api/v1/talkgroups`) and SSE stream (`/api/v1/events/stream`: `call_start`, `call_end`, `unit_event`), and plays audio through tr-engine's WebSocket PCM stream via `audio-engine.js`.

Port it to a **Trunk Recorder Pro interface**: a folder served by the recorder on a port of its own, talking to the recorder's own API. Keep the look, the layout, the behaviour and the themes. Change only what feeds it.

## The target

Repo folder: `D:\trunk-interface-omnitrunker` (git, branch `main`, empty). Layout per TRP's docs:

```
index.html          the page (the port of omnitrunker.html)
interface.json      { "id": "omnitrunker", "name": "OmniTrunker", "version": "0.1.0", "description": "Live voice channels and unit activity, dispatch-wall style, with per-talkgroup listening.", "api": 1, "access": "listen", "license": "MIT", "repository": "https://github.com/trunk-reporter/trunk-interface-omnitrunker" }
theme-config.js     copied as is
theme-engine.js     copied as is (strip any tr-engine-only hooks it has; see below)
README.md           short: what it is, how to serve it (Interfaces → Add a folder…, or trunk-pro --ui), what it shows
LICENSE             MIT
docs/brief.md       this file
```

Rules for interfaces (from `D:\trunk-recorder-pro-104\docs\interfaces\publishing.md`): relative paths for its own files, absolute for the recorder's (`/api/client.js`, `/api/ws`, `/calls/…`); nothing under `api/` or `calls/`; **load nothing from a CDN or the internet** (a recorder may be offline): the Google Fonts `<link>` goes; the themes' font stacks keep their names and fall back to local/system fonts. No `auth.js`: the recorder's own port carries the access. Listen access is enough: the page changes nothing.

## The recorder's API

Read these first, in this order:

1. `D:\trunk-recorder-pro-104\docs\api\README.md` — the guide: Quick start, Serving your interface, Access, `client.js` (`TrunkClient`, `LivePlayer`), Concepts, every message.
2. `D:\trunk-recorder-pro-104\docs\api\client.js` — the client library the recorder serves at `/api/client.js`. Use it (`import { TrunkClient, LivePlayer } from "/api/client.js"`), do not copy it into the folder.
3. `D:\trunk-recorder-pro-104\docs\api\examples\wall.html` and `scanner.html` — small working interfaces; the shape to follow for connecting, listening and playing.
4. `D:\trunk-recorder-pro-104\web\src\protocol.ts` — every message as types with comments: `CallView` (a call on the air: id, system, systemName, site, talkgroup, alphaTag, freqHz, slot, analog, nxdn, encrypted, decrypted, emergency, startS, sources[], patched, state, reason), `EngineStatus` (`status.systems[]`, each with its `calls: CallView[]`), `CallEntry` / `CallRecord` (a recorded call), `MonitorEventBody` (kinds `tgFirstSeen`, `unitFirstSeen`, `tgActive`, `unitActive`, `unitAffiliated`, `callVolume`, `tonePattern`, `controlLost`, `controlRegained`, …, with `system`, `talkgroup`, `alphaTag`, `unit`, `alias`, `units`, `text`, `freqHz`), `unitAlias`, `talkgroups`.

The mapping, decided:

| OmniTrunker today (tr-engine) | In the port (TRP) |
|---|---|
| `GET /api/v1/systems` | `rec.config.systems` (short names, names) and `rec.status.systemNames` |
| `GET /api/v1/talkgroups?limit=5000` | `rec.talkgroups(shortName)` per system (`talkgroupRows` + edits): talkgroup → `{ talkgroup, alphaTag, description, tag, category }` |
| SSE `call_start` / `call_end` | the `calls` array on `rec.status` after each `change`: a call that appears is a start, one that disappears is an end; `CallView.id` keys the row; elapsed from `startS` against `rec.status.nowS` |
| SSE `unit_event` (tr-engine's `event_type`: on / off / ackresp / join / data / ans_req / location / call) | `monitorEvent` messages (`rec.on("message", m => m.type === "monitorEvent")`) of kinds `unitFirstSeen`, `unitActive`, `unitAffiliated`, `tgFirstSeen`, `tgActive`, `tonePattern`, plus each call's `sources[]` as it appears (a "call" row per new talker). The recorder's interface API does not carry raw registration/affiliation/ack events, so the feed is the recorder's notable events; label them with their kind. Keep the type filter, built from the kinds actually seen. |
| `audio-engine.js` + tr-engine's PCM WebSocket | `LivePlayer` from `/api/client.js`: `rec.listen()` / `rec.listen({ system, talkgroup })` / `rec.listen(false)`, `rec.on("audio", chunk => player.play(chunk))`, `player.resume()` after the click; master volume → `player.gain`; per-talkgroup mute → drop the chunk when its `talkgroup` is muted (the chunk carries `system`, `callId`, `talkgroup`); auto-pan if the player offers a pan, else leave the checkbox out. |
| `auth.js` | removed |
| the `card-*` `<meta>` tags, the "— tr-engine" title suffix | removed; title "OmniTrunker" |

Keep: the two panes, the divider drag, the by-talkgroup / by-frequency view toggle, the system filter, the settings drawer (talkgroup search, mute/unmute all, column choices, volume), the event counter, the themes (`theme-config.js` + `theme-engine.js` as they are, minus anything that reaches tr-engine: grep them for `fetch`, `/api/`, `trAuth`, `localStorage` keys are fine to keep), settings persistence in `localStorage` (the port runs on its own origin, so keys need no prefix change).

## How to work

- Start by reading the page end to end once (its CSS, HTML and the big script), then the API guide and `client.js`, then `wall.html`.
- Build `index.html` as a port, not a rewrite: keep the page's own functions and names where the data still fits (`addActiveChannel`, `removeActiveChannel`, `buildActivityRow`, `renderTGList`, …); replace `connect()`, the fetches, the SSE handlers and the audio section. Where a field no longer exists (tr-engine's `call_id` format, `audio_type` strings, `unit_event` fields), adapt the small helper that read it (`mapAudioType`, `getSystemName`, `parseStartTimeMs`, …) rather than the renderers.
- No build step, no bundler, no framework: plain HTML/CSS/JS as the original. ES modules are fine (`<script type="module">`), which `client.js` needs.
- Test it against the live recorder: it runs at `http://127.0.0.1:8080` (recording now). Serve the folder as an interface for the test without touching the recorder's saved config: `cd D:\trunk-recorder-pro-104 && .\target\release\trunk-pro.exe` must NOT be started a second time (the recorder is running; a second instance fights for the radios). Instead use the Interfaces page's "Add a folder…" over the API: a `setConfig` that appends `{ "folder": "D:/trunk-interface-omnitrunker", "port": 8083, "access": "listen" }` to `server.interfaces` (see `docs/configuration/server-and-log.md` § interfaces for the exact shape; look at how the UI does it in `web/src/Interfaces.tsx` if unsure). A script that already does a `setConfig` round trip over `ws://127.0.0.1:8080/api/ws` is at `C:\Users\drewm\AppData\Local\Temp\claude\D--trunk-recorder-pro\21e10349-b74d-4d44-aa29-08c26e715dcb\scratchpad\trp-add-dvcf.mjs`: copy its pattern. Then open `http://127.0.0.1:8083/` in the browser tools (the claude-in-chrome tools) and check: it connects, the top pane fills with the calls on the air within a minute (the `marcs` system is busy), rows leave when calls end, the feed shows events, LISTEN plays audio after a click, the mute list works, the system filter works, the view toggle works, the themes switch, no console errors, no request leaves to any host but the page's own.
- Browser automation is available (the `mcp__claude-in-chrome__*` tools: load them with ToolSearch `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_console_messages,mcp__claude-in-chrome__read_network_requests,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp` in ONE call; call `tabs_context_mcp` first and create a new tab). Use it for the checks above and take a screenshot of the working page into `screenshots/wall.png` (about 1280 × 800) for the README and `interface.json` (`"screenshots": [{ "path": "screenshots/wall.png", "caption": "Calls on the air and the activity feed" }]`).
- Commit in small steps on `main` with the trailer convention below; the last commit leaves the folder complete.

## Done means

- `index.html` works against the running recorder as above, with no CDN/internet request and no `auth.js`.
- `interface.json` validates against the table in `publishing.md`; the folder has only its own files (no `api/`, no `calls/`).
- README and screenshot in place.
- A report at `D:\trunk-interface-omnitrunker\docs\port-report.md`: what was mapped how, what the feed shows now versus before, what could not be carried over and why, how it was tested (which checks, what was seen), the commits.

## Commit convention

Subject in plain English, blank line, then exactly:

```
Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01JXUSP15Bo9JgvCdvA3xtZW
```

(`git commit -F -` with a heredoc.)
