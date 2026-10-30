# OmniTrunker interface, round 3: a Site column, the `activity` topic, and several calls at once with panning

Repo: `D:\trunk-interface-omnitrunker` (git, `main`, clean; the interface is `index.html`, plain HTML/JS, no build step). The first brief, `docs/brief.md`, and the port report, `docs/port-report.md`, say how the page is built and what it reads. Read `docs/port-report.md` first, then the parts of `index.html` named below. The original page it was ported from is `D:\reporter\web\omnitrunker.html` with `D:\reporter\web\audio-engine.js` beside it: the L/R column and auto-pan come back from there.

The recorder is running at http://127.0.0.1:8080 (recording the `marcs` system, three sites, busy) and serves this folder as the interface at **http://127.0.0.1:8083/** (its config names the folder; a reload shows every edit). Its API guide: `D:\trunk-recorder-pro-104\docs\api\README.md`; the client library it serves at `/api/client.js`: `D:\trunk-recorder-pro-104\docs\api\client.js` (this copy has the new `LivePlayer` options; the served one gains `context` once the recorder is restarted, see C). The recorder will be **restarted once by the controller while you work**, to pick up the new client.js; the page reconnects by itself. Never start a second recorder.

Three changes, each its own commit (or a few), in this order.

## A. A Site column in Active Voice Channels

The top table has no site column (the original had none either), so with one system nothing tells the three sites apart. The feed's rows already have one.

- Add `<th>Site</th>` right after `System` in `#channels-table` (line ~802) and the matching cell in `buildChannelCells` (line ~1332, after the System cell) and in `renderChannelRow`'s in-place update (line ~1559: every `ch[n]` index after System shifts by one; the `ch.length < 10` guard becomes 11). The frequency view builds rows with its own functions (`addActiveChannelFreqMode` and whatever renders its rows: find them, they share the column layout): give them the cell too.
- The cell shows `data.site` (already in the row data from `callToData`, line ~1900, from `CallView.site`). A call the recorder also hears on other sites comes with `CallView.alsoOn` (an array of site names; one copy is saved): add `also_on: c.alsoOn || []` to `callToData`, and show it in the cell as a muted suffix, e.g. the site name then ` +2` with the full list as the cell's `title` ("also on 001 Butler County Simulcast, 013 Warren Co Simulcast"). Style the suffix with `color: var(--text-muted)`.
- A column toggle like the others: the settings drawer's column list (line ~886, `id="col-channel"` …) gets `col-site`; the `.hide-col-*` rules (line ~624) use `nth-child` indexes: the new column is child 3, so every later index shifts by one; the loader at line ~2203 and the handler at ~2706 need nothing if they are generic (check).
- Check live: the cell shows `004 Hamilton County Simulcast`, `013 Warren Co Simulcast`, `001 Butler County Simulcast` as calls come (all three are on the air within a minute).

## B. The recorder's `activity` topic

The live recorder sends the frequent events to a connection that subscribes to the topic `activity`: `monitorEvent` messages of kinds `unitActive` (`system`, `unit`, `talkgroup`), `unitAffiliated` (`system`, `unit`, `talkgroup`), `tgActive` (`system`, `talkgroup`, `units[]`) and `callVolume` (`system`, `perMin`). Subscribe with `rec.subscribe(['activity'])` once after `rec` is made (client.js sends it again on every reconnect; don't wait for hello). Verify it with the `subscribed` message or by seeing the rows.

What the feed does with them, decided:

- `unitAffiliated` → a feed row of type `unitAffiliated` (getFilterKey already maps it to `affiliation`; make sure the type filter dropdown lists it when it first appears, like the other kinds).
- `unitActive` and `tgActive` are **not** feed rows: they say the same as the grant rows the feed already makes from each call's `sources[]` in `syncCalls` (the recorder emits `unitActive` per new talker on a call and `tgActive` at a call's start with its units), and those rows carry the frequency, alpha tag and site. Take both out of `FEED_KINDS` (line ~2000) so nothing doubles.
- `callVolume` is a per-minute figure for a system, not a radio event: no row.
- Update the comment above `FEED_KINDS` (it says the recorder does not send these; now it does on request) and `docs/port-report.md`'s feed section accordingly.
- Check live: affiliation rows appear (about 7 a minute on this system) and grant rows are not doubled (count grant rows in 60 s against calls' new talkers, or just watch for identical consecutive rows).

## C. Several calls at once, each in its own place

Today one `LivePlayer` plays one call at a time (`followCall` true, its default): with ten calls on the air the listener hears one. The original mixed every unmuted talkgroup, each panned to its own spot (auto-pan), with an L/R column showing where. Bring that back on top of the recorder's live audio.

How `LivePlayer` now works (D:\trunk-recorder-pro-104\docs\api\client.js, class `LivePlayer`, and the README's "new LivePlayer(opts?)" table): options `followCall`, `quietS`, `gain`, `pan` (-1…1), `context` (an `AudioContext` to share), `onNowPlaying`; methods `resume()`, `play(chunk)`, `setGain(v)`, `setPan(v)`, `stop()` (with a shared `context`, `stop()` releases only that player's nodes and leaves the context open); `nowPlaying` is `{ callId, system, talkgroup }` or null. The served client.js gains `context` when the controller restarts the recorder during this round; before that, a player given `context` ignores it and makes its own `AudioContext` on `resume()`. Code for both: pass `context`, and it simply works either way (after the page's LISTEN click, the browser lets later contexts start too). Don't copy client.js into the folder.

The design:

- A shared `AudioContext` made on the LISTEN click (a user gesture), kept while listening, closed on LISTEN off.
- One player per call on the air: `players: Map<callId, LivePlayer>`. On each `audio` chunk, after the existing system-filter and mute checks: get or make the call's player with `{ followCall: false, gain: currentGain, pan: panFor(chunk), context: sharedCtx }`, call `player.resume()` the first time (the shared context is already running, so no gesture is needed), then `player.play(chunk)`. Record the talkgroup in `audibleTGs` as today (any player).
- A player goes when its call does: in `syncCalls`, where an ended call leaves `liveCalls`, `stop()` and drop its player; also drop a player whose `nowPlaying` has been null for a while (quietS passed) as a safety net, so the map never grows without bound.
- Master volume (0–100 → gain, the mapping that exists at line ~2345) → `setGain(g)` on every player, and the gain for new ones. Delete the private-field poke (`player._gain`).
- Pan: auto-pan on by default. `panFor(chunk)` with auto-pan on is the original's deterministic spot from the talkgroup: `h = ((tgid * 2654435761) >>> 0) % 10000; pan = -0.8 + (h / 10000) * 1.6`; off: 0. Toggling auto-pan re-pans every live player (`setPan`). Two controls as in the original: the `Pan` checkbox beside the volume slider in `#audio-controls` (original line ~812, `id="auto-pan-toggle"`) and the **Auto-pan** row in the settings drawer (original lines ~908–918, `id="set-auto-pan"`), kept in step, saved as `autoPan` in `omni-settings` (false is the only value that needs saving; default on).
- The L/R column: `<th title="Pan">L/R</th>` after the speaker column (original line ~830), the `.pan-indicator` CSS (original lines ~389–415: a 33×6 px bar with a centre tick and a 4 px dot), a cell per row holding the indicator; the dot's `left` is `((pan + 1) / 2) * 29` px. The row's pan is `panFor` of its talkgroup (the same function, so the dot is right whether or not the call is playing yet); update the dots when auto-pan toggles. This column is child 2, so the Site column of part A becomes child 4 and the `nth-child` rules shift again: do A and C's column work with that in mind, and keep the column toggles right. Give L/R a toggle too (`col-pan`).
- The audio status text (`#audio-status`): "N playing" with N the players whose `nowPlaying` is set (0 → "idle"), refreshed in the existing poll loop.
- `window._omniAudio` keeps its interface (`isMuted`, `toggleMute`, `onSystemFilterChange`); muting a talkgroup also stops and drops any player on a call of that talkgroup, so it falls silent at once.

Check live, in the browser (the claude-in-chrome tools; load them in ONE ToolSearch call: `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__read_console_messages,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp`; call `tabs_context_mcp` first; open http://127.0.0.1:8083/ in a new tab; close your tabs when done): click LISTEN, and within a minute confirm with `javascript_tool` (the page's variables are inside closures, so expose what you need on `window._omniAudio` for the check, e.g. a `debug()` returning `{ players: [...{callId, talkgroup, pan, nowPlaying}], contextState }`) that two or more players exist at once with different pans, that the shared context is running (or, before the restart, that each has its own), that players go when calls end, that the L/R dots sit where the pans say, that muting a talkgroup drops its player, and that the volume slider changes every player's gain. No console errors; no request to any host but the page's own.

## Done means

- Everything above works against the live recorder; `interface.json` version `0.2.0`; README mentions the Site and L/R columns, auto-pan and the `activity` topic in a line each; `docs/port-report.md` has a short "Round 3" section: what changed, what the feed shows now, how it was tested.
- Commits in small steps on `main` with exactly this trailer after a blank line (`git commit -F -` with a heredoc):

```
Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01JXUSP15Bo9JgvCdvA3xtZW
```

- A report at `C:\Users\drewm\AppData\Local\Temp\claude\D--trunk-recorder-pro\21e10349-b74d-4d44-aa29-08c26e715dcb\scratchpad\omni-round3-report.md`: commits, what was checked and what was seen (numbers), anything left out and why. Return only: status (DONE / DONE_WITH_CONCERNS / BLOCKED), the commit list, one line of test summary, concerns.

Don't dispatch subagents. Don't touch the recorder, its config, or anything outside this folder.
