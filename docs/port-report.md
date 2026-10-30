# OmniTrunker port: report

tr-engine's `web/omnitrunker.html` ported to a Trunk Recorder Pro interface (recorder 0.2.11,
interface API 1). Same two panes, divider, view toggle, filters, settings drawer, event counter and
themes; the feed, the audio and the system/talkgroup lookups now come from the recorder's
connection through `/api/client.js`.

## How each piece was mapped

| tr-engine | Port |
|---|---|
| `GET /api/v1/systems` | `rec.config.systems` + `rec.config.conventional` on each `config` message (`loadSystems`): short name is the system's id everywhere in the page (filter values, mute keys, `data.system_id`); `name` (else the short name) is what is shown. Calls and audio chunks name their system by number; `CallView.systemName` / `systemName(rec.status, n)` turn it into the short name. |
| `GET /api/v1/talkgroups?limit=5000` | `rec.talkgroups(shortName)` for every system, rebuilt on `config`, `talkgroups` and `talkgroupEdited` (`loadTalkgroups`) into the same `tgCache` (`"shortName:tg"` → `{ alpha_tag, description, group, tag }`). Note: client.js returns the category as `group`, not `category`. |
| SSE `call_start` / `call_end` | `syncCalls()` on every `status` (and `state`): the calls in `rec.calls` are diffed by `CallView.id` against the last set. New → `addActiveChannel` + a feed row; gone → `removeActiveChannel` + a CALL END row; changed (talker, alias, alpha tag, encrypted, emergency, frequency, mode) → the row is re-rendered. `callToData()` turns a `CallView` into the field names the renderers already read, so `buildChannelCells`, `renderChannelRow`, `buildActivityRow`, `addActiveChannel(FreqMode)`, `removeActiveChannel(FreqMode)`, `switchViewMode` are unchanged apart from the pan column. Elapsed: wall-clock start = now − (`status.nowS` − `startS`), fixed when the call is first seen; `parseStartTimeMs` takes that number. |
| SSE `unit_event` | `monitorEvent` messages of kinds `unitFirstSeen`, `unitActive`, `unitAffiliated`, `tgFirstSeen`, `tgActive`, `tonePattern` (`onMonitorEvent`), plus a GRANT row for each radio in a call's `sources[]` as it first appears. Badges: GRANT (talker / `tgActive`), UNIT ACTIVE, AFFILIATION, NEW UNIT, NEW TG, TONES (the pattern's `text` in the TG Alpha column), CALL END. The type filter starts empty and gains a type the first time a row of it arrives; a saved filter is applied as types appear (`ensureTypeItem`, `_pendingTypeFilter`; "all" is saved as `null`). |
| `audio-engine.js` + PCM WebSocket | `LivePlayer` + `rec.listen()`. LISTEN click → `player.resume()` then `rec.listen(filter)`: `{ system }` when exactly one system is selected, else everything. Each chunk is dropped when its system is filtered out or its `"shortName:tg"` is muted, else `player.play(chunk)`. Stop → `rec.listen(false)` + `player.stop()`. Row glow / "N active" come from chunks of the call the player is actually playing. The system filter now re-sends `listen` (in tr-engine that call sat behind `typeof engine`, which was never visible from that script, so it never ran). |
| master volume | 0–100 → gain `2.5 × v / 100` (80, the default, is the player's own 2). See Concerns: LivePlayer has no `gain` property. |
| per-talkgroup mute | Unchanged model (`mutedTGs`, composite keys, the drawer list, Unmute all / Mute all, speaker icons). A talkgroup not in the talkgroup file can now be muted from its row (its own key); before, the speaker click did nothing for those. |
| auto-pan | LivePlayer has no pan: the Pan checkbox, the settings toggle and the L/R column are gone (the brief: leave the checkbox out). |
| `auth.js`, `card-*` meta, "— tr-engine" title, Google Fonts | Removed. Title "OmniTrunker"; the theme header's title/subtitle come from `THEME_ENGINE_OPTIONS`. |
| `theme-config.js` | Copied as is (no fetches, no URLs). |
| `theme-engine.js` | Copied, minus its two tr-engine fetches: the nav page list (`/api/v1/pages`; now "no other pages") and the update badge (`/api/v1/health`); default subtitle "Trunk Recorder Pro". |
| connection status | `rec.connected`, `rec.phase`, `rec.exited` (`updateConnectionStatus`): connected / connected, recorder idle / reconnecting / recorder stopped. |
| stale-row sweep (5 min without audio) | A row is stale when no call on its talkgroup is on the air (the recorder lists them all), e.g. after a dropped connection. The 5-minute rule would have faded long calls. |

## The feed now versus before

Before: every control-channel unit event tr-engine decoded (registration, deregistration,
affiliation, acknowledge, data grant, location, call alert) plus a GRANT per call start and a
CALL END per call end.

Now: a GRANT per radio that talks on a call (from `sources[]`), a CALL END per call end, NEW UNIT
and NEW TG as the recorder first hears them (plus the recent ones it replays on connecting), and
TONES. On the live `marcs` system the feed was mostly GRANT and CALL END, with NEW UNIT / NEW TG
interleaved.

## What could not be carried over, and why

- **Raw unit events.** The interface API does not carry registrations, affiliations, acks, data
  grants, locations or call alerts. Those badge types are gone from the filter (their mappings
  stay in `getBadgeInfo`, unused).
- **`unitActive`, `unitAffiliated`, `tgActive`, `callVolume` never arrive.** The brief maps them,
  and the page handles them, but the recorder builds these "frequent" kinds only for its internal
  watchers and does not send them as `monitorEvent` (`crates/trunk-app/src/stats/events.rs`:
  `MonitorEvent::frequent()`; `Monitor::emit` returns no message for them). `tonePattern` comes
  only for a system's own signals. So in practice the recorder's events in the feed are NEW UNIT,
  NEW TG and TONES; per-talker GRANT rows stand in for unit activity.
- **Mixing and stereo pan.** tr-engine's engine mixed several talkgroups at once and panned them.
  LivePlayer plays one call at a time (the first heard until quiet for 1.5 s; overlapping calls are
  skipped) and is mono. Pan is gone; overlapping calls are not heard together.
- **`addAudioChannel`** (rows created from audio frames) is dropped: its only caller was already
  disabled in tr-engine.
- **Site in monitor events.** Only control-channel events carry a site, so NEW UNIT / NEW TG rows
  have no site.

## Fixes made in passing

- `renderChannelRow`'s in-place update read `encrypted`, which was not defined in that function
  (a ReferenceError on every re-render in tr-engine). Defined, and the Encryption cell now
  updates when a call turns out encrypted.
- Encrypted calls the recorder is decrypting (`decrypted`) are not marked encrypted / no audio.
- The phone-width rule hid Speaker, Elapsed and Mode while its comment says Speaker, Mode and
  Encryption; with the pan column gone it now hides what the comment says.
- Feed de-duplication: talker rows de-duplicate per (unit, type, talkgroup), so several radios
  on one talkgroup within 10 s each get a row; call-level rows still de-duplicate per talkgroup.

## How it was tested

Against the running recorder (0.2.11, system `marcs`, recording). No second recorder was started.
One config change: `server.interfaces.omnitrunker = { port: 8083, enabled: true, access: "listen",
path: "D:/trunk-interface-omnitrunker" }`, sent as a `setConfig` over `ws://127.0.0.1:8080/api/ws`
(config version 44 → 45), left in place. The brief described this as appending
`{ folder, port, access }` to a list; the config's `server.interfaces` is a map by id with `path`,
so it was added in that shape (`docs/configuration/server-and-log.md`).

In the browser (Claude in Chrome), at `http://127.0.0.1:8083/`:

- **Connects:** status "connected"; system filter lists `oh-marcs` (config name).
- **Calls fill the top pane:** 5–8 rows within seconds, with frequency, talkgroup, alpha tag,
  radio, unit alias, elapsed and FDMA.
- **Rows leave when calls end:** two snapshots 20 s apart showed calls replaced, ended ones with
  `fading` and gone afterwards.
- **Feed:** 60+ events in the first minute: GRANT, CALL END, NEW UNIT, NEW TG; the type filter
  listed exactly those four; unticking Grant hid the GRANT rows.
- **LISTEN:** after a real click: "LISTENING", status alternating "1 active" / "idle", the playing
  call's row glowing (51509, 55649, 55651 in turn). A second connection confirmed audio frames
  flowing (2,043 in 10 s across 10 talkgroups). The sound itself was not heard by a person.
- **Mute list:** muting every row's talkgroup from its speaker icon turned the icons to muted,
  saved `marcs:<tg>` keys, the drawer showed "6 muted"; afterwards only other talkgroups glowed
  (one carry-over tick of under 0.5 s right after muting). The drawer listed 6,650 talkgroups
  under "oh-marcs (6650)"; search "FD01" found FD01DISP…; Unmute all → "all playing".
- **System filter:** None hid every channel row and feed row; All showed them again.
- **View toggle:** FREQ → "Frequency Monitor", one row per frequency, an ended one dimmed (idle);
  TG → "Active Voice Channels" with its rows back.
- **Themes:** 11 themes; the picker and `ThemeEngine.apply` changed the colours and font stacks
  (obsidian shown in the screenshot).
- **Console:** no errors from the page (only the browser extension's own log lines).
- **Requests:** `/`, `theme-config.js`, `theme-engine.js`, `/api/client.js` and the WebSocket,
  all on `127.0.0.1:8083`; nothing to any other host.

Not tested: a reconnect after the recorder restarts (it was not restarted), the resume hint after
a reload with listening saved, phone width, and a conventional system (none configured).

The screenshot (`screenshots/wall.png`, 1279 × 799) was taken of the page in a 1280 × 800 frame,
as the browser window could not be resized below its maximised 4K size.

## Commits

- 543d853 Start the OmniTrunker interface: manifest, license, the brief, and the themes as they are
- 8f215d2 Themes: drop the engine's tr-engine fetches (page list, update badge)
- 41141ac Bring in tr-engine's omnitrunker.html as index.html, unchanged, so the port reads as a diff
- b2c5d87 Port the page to the recorder's API: calls on the air, monitor events, LivePlayer
- 71927c2 README and the screenshot, named in interface.json
- (this report)

## Concerns

- **Volume reaches into LivePlayer.** The brief says `player.gain`; LivePlayer has `gainValue`
  (read only when its output is made) and a private `_gain` node. `setVolume` sets both. A
  `player.setGain()` in client.js would make this clean.
- **The screenshot shows real data** from the live system: talkgroup alpha tags and radio aliases.
  The store review asks for nothing private in screenshots; check it, or retake on a capture,
  before publishing.
- The interface's `repository` (github.com/trunk-reporter/trunk-interface-omnitrunker) does not
  exist yet; nothing was pushed.

## Fixes after review

Five fixes from the review, all in `index.html`:

1. **The TG view went stale after time in FREQ (Important).** While FREQ showed, `activeChannels`
   was never updated, so switching back showed the calls from before the switch, and a session
   that started in FREQ had an empty TG view. `switchViewMode` now rebuilds the view it switches
   to from `liveCalls` (the calls on the air): TG clears `activeChannels` and adds each live call;
   FREQ seeds `freqChannels` from every live call, so a second call on one talkgroup is not lost.
2. **A saved type filter hid types that arrived later (Important).** It was saved as the ticked
   types (a whitelist), so any type first seen later was unticked. It is now saved as the
   unticked types (`typeHidden` in `omni-settings`, `omni-type-hidden`), and a type not in that
   list is shown. Unticked types that have not appeared yet in a session stay remembered.
3. **The recent events replayed on every reconnect.** The greeting resends the recent
   `monitorEvent`s. The page keeps the newest event time it has taken, and drops greeting events
   (before `hello`, when `rec.connected` is false) at or before it. Live events are not cut,
   because several can share one time (several radios first heard on one call).
4. **A row could keep an ended call's data.** When a call ended while another call on the air
   held its row (same talkgroup, or same frequency in FREQ), the row still showed the ended call.
   It is now redrawn with the call that holds it.
5. **No systems chosen still pulled all audio.** With the system filter at None, LISTEN now sends
   `rec.listen(false)` instead of listening to everything and dropping every chunk.

Also removed `docs/review-package.diff` (the reviewer's input; it was never committed).

Re-tested in the browser at `http://127.0.0.1:8083/` against the live recorder:

- **1:** TG showed 5 calls; switched to FREQ (5 rows), waited 25 s (25 frequency rows, ended
  ones dimmed), switched back to TG. The TG view held 9 rows, none of the 5 from before the
  switch, matching the talkgroups the recorder listed on the air at that moment (checked on a
  separate connection; two talkgroups had two calls each, one row each). After the next `status`
  every row's call id matched a call on the air.
- **2:** Cleared storage, loaded the page, and unticked NEW UNIT as soon as the first types
  appeared. Only NEW UNIT and NEW TG had been seen at that point, and the saved value was
  `["new-unit"]`. After a reload, GRANT and CALL END (never seen before the untick) arrived ticked
  and their rows showed. NEW UNIT stayed unticked and hidden, and the button read "3 types".
- No console errors. Fixes 3 to 5 were checked by reading the code, not in the browser (a
  reconnect needs the recorder restarted).

### Round 2: the replay cut-off

The first version of fix 3 compared greeting events against `lastEventT`, which the replay
itself moved forward. So a replayed event sharing a time with the one before it was dropped,
on first load too. Now the cut-off (`replayCut`) is taken once, from `lastEventT`, when the
connection drops (`rec.on('connection', false)`), and greeting events are compared with that.
`lastEventT` keeps advancing separately for the next drop. On the first connection the cut-off is
0, so nothing is cut.

Re-tested in the browser on port 8083 with cleared storage. I loaded the page into a same-origin
frame with its WebSocket wrapped, to count the greeting's messages before `hello`, because `rec`
is module-scoped and not reachable from the console. The greeting carried 200 `monitorEvent`s.
66 of them were the kinds the feed shows (59 `unitFirstSeen`, 7 `tgFirstSeen`), and 5 of those
shared a time with another. The feed had 66 rows of those kinds, so they matched. Then I closed
the socket. The page reconnected (a second `hello`), and the feed still had the same number of
those rows (67 before, 67 after), so nothing was replayed twice.
