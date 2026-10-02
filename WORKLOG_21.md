# NOBODY — Worklog

| Phase | Title | Status | Version | Date |
|---|---|---|---|---|
| 0 | Full source audit (engine, classic, faces, desktop/installer) | ✅ Done | — | 2026-09-30 |
| 1 | Critical bug fixes + security hardening | ✅ Done (needs runtime QA on Windows) | 1.5.3 | 2026-09-30 |
| 2 | Performance (virtualized lists, render storms, caches, bulk IDB, frame loop, resume IDB) | ✅ Done (needs runtime QA on Windows) | 1.5.4 → 1.5.5 | 2026-09-30 |
| 3 | Core player features (gapless, crossfade, ReplayGain, queue drag, sort, repeat modes in classic) | ✅ Done (needs runtime QA on Windows) | 1.6.0 → 1.6.5 | 2026-09-30 |
| 4 | Library power features (tag editor, smart playlists, genres, stats, M3U, scrobbling, duplicate finder) | ✅ Done (needs runtime QA on Windows + real Last.fm/ListenBrainz) | 1.7.0 | 2026-10-01 |
| 5 | Desktop integration | 🔧 Code done except Discord RPC + auto-update/signing (parked by owner). Installer hardening in 1.8.0-dev.4. Needs Windows QA | 1.8.0 | |
| 5.5 | Design: 3 new faces (CYBERCORE · MAXIMAL · BOHEME), see DESIGN_PHASE.md | ⏳ next | 1.9.0 | |
| 6 | Architecture: merge classic onto the shared engine, tests, CI, signing | ⏳ | 2.0.0 | |

---

## Phase 0 — Audit (done)
Four parallel static audits of ~60k lines. Full reports: `docs/audit/*.md`.
Note: some auditor claims were verified FALSE and not "fixed": MP4 `data` type offset (original code correct), engine `pushState` on every tick (it isn't), Ctrl+K inside inputs (intended).

## Phase 1 — Critical fixes (done, V1.5.3)
### Engine (Cinema / EELA / ALOK)
- engine.ts: load-generation token → rapid Next/jump can no longer play a stale track or revoke the live object URL.
- engine.ts: missing-file skip bounded to one lap of the queue (was an infinite loop on an all-missing queue) + one toast.
- engine.ts: play() errors other than AbortError are logged instead of silently swallowed.
- smartFetch.ts: per-run id → cancelled workers can't corrupt a new run; O(1) item status updates.
- mediaSession: now initialised in EELA and ALOK too (was Cinema-only → no SMTC metadata when booting into them).
- App handoff (3 faces): one-shot, track-bound loadedmetadata listener (old one could seek the NEXT song).
- EELA PlayerBar: dock clock is ref-counted/cancellable, idle when hidden, writes DOM only when a second flips.
### Metadata / lyrics
- audioParser.ts: MP4 v1 `mvhd` 64-bit duration words were swapped.
- audioParser.ts + metadata.ts: ID3v2.3 unsynchronisation + extended header (v2.3 & v2.4) handled.
- metadata.ts: Cinema importer now uses the broad parser for FLAC/Ogg/Opus/MP4/WAV/AIFF (tags + cover were lost).
- lyrics.ts: `[mm:ss]` LRC without fraction is recognised as synced.
### Data layer
- covers.ts: object-URL leak on corrupt covers fixed; concurrent coverUrl() de-duplicated; derived caches bounded; revokeAllCovers clears all.
- db.ts: failed IDB open no longer poisons the session; versionchange handled.
- library.ts: recents no longer grow duplicate IDB rows forever.
### Classic
- Repeat toggle now works on real files (onEnded ignored it).
- Rapid Next no longer pauses the new track (AbortError from the old play()).
- Removing/rescanning a folder keeps the SAME song selected and prunes Ur Queue/Recents ghosts.
- Beat visualizer idles while classic is hidden behind another face.
- Clipboard copy only shows "copied" on success; close veil can't freeze the web preview.
### Electron security
- fs IPC: reads restricted to media/sidecar extensions, writes to .lrc/.txt/.jpg/.png/.webp, 32 MB write cap.
- app:// protocol serves media/sidecar types only (403 otherwise).
- http proxy: redirects followed manually, every hop re-checked against the allow-list (SSRF fix).
- openExternal / window.open: https + host allow-list; will-navigate blocked.

### Phase 1 QA checklist (run on Windows before release)
- [ ] `bun install && bun run build && bun run electron:start` (no TS errors)
- [ ] Rapid Next ×20 in each face → correct track, no silence
- [ ] Repeat on in Classic → loops
- [ ] Import FLAC + M4A in Cinema → tags + cover
- [ ] Lyrics/cover Smart Fetch: start, cancel, restart
- [ ] Remove a folder while playing a track from another folder
- [ ] Support links open in browser; sidecar .lrc/.jpg save still works

## Phase 2 — Performance (done, V1.5.4 + V1.5.5)
### Done
- `src/shared/VirtualList.tsx`: face-neutral windowing (@tanstack/react-virtual), auto-detects the nearest scroll box, flow layout with spacers (row CSS untouched), plain render for lists <= 60.
  Wired into: EELA library cards, ALOK Library/Liked/Recents/Playlist panels, Classic library table + queue panel, Download Center, EELA/ALOK Downloads, Cinema Smart Fetch list.
- ALOK TrackRow: memo + boolean selectors; playlist popover split out (a track change no longer re-renders every row).
- Classic library/queue: memoised Set-based filtering (was includes() per row, O(n·m)).
- `removeTracks(ids)`: one store update + ONE IDB transaction (`db.deleteTracksBulk`); only changed playlists are re-saved. folderManager.removeFolder uses it.
- `subfoldersUnder()`: single-pass O(n) bucketing.
- `loadDesktopFolders()`: bounded pool of 6 (`src/shared/pool.ts`); one bad file no longer drops the whole folder.
- `src/perfMode.ts` + `src/perf.css`: all CSS animation pauses while the window is hidden; Cinema aurora loses permanent will-change and only drifts while playing; opt-in Performance mode (`localStorage nobody-perf-mode = "lite"`) drops backdrop blur + decorative infinite animations.
- `src/shared/frameLoop.ts`: shared visibility-aware rAF scheduler (hidden-window stop, element visibility via IntersectionObserver, park/wake, 30 fps decorative cap in Performance mode).
- Dropped stale `package-lock.json` (bun.lock is the lockfile); no `dist/` in the source zip.
### Done in V1.5.5 (Phase 2 part 2)
- rAF loops migrated onto `frameLoop.subscribeFrame`:
  - EELA: `EelaVisualizer` (canvas-bound, decorative, idle wave capped at 30 fps), `EelaSlider` (element-bound, style written only on change; latest `getFrac` via ref), `NpClock` + dock clock (10 Hz, text change-gated), NP `LyricsSheet` (25 Hz, not subscribed at all unless it shows the current track with synced lines).
  - ALOK: `Core` ring clock (ring-bound, elapsed label now change-gated too), bass glow (decorative, parks via `return false`, woken by engine state / UI switch), mini-lyric whisper (25 Hz), `RadialViz` (canvas-bound, decorative, parks when paused + decayed), `LyricsSheet` (25 Hz).
  - Left alone on purpose: custom-cursor loops (pointer-driven) and one-shot rAFs (layout measure / scroll centering).
- Resume state (`engine.ts`):
  - `saveResume()` debounced 600 ms; `flushResume()` on `visibilitychange→hidden`, `beforeunload` and the 4 s playback tick.
  - localStorage keeps a v2 head `{v, trackId, position, repeat, shuffle, volume, queueLen, queue? | queueRev}`; queues ≤ 2,000 ids stay inline.
  - Bigger queues go to IDB `session` key `resume-queue` = `{rev, ids}` (DB_VER 3, new `session` store; `clearAll` clears it). Written only when `queueRev` changed (setQueue / add / remove / clear / restore), never for position updates. A shrunk queue deletes the stale IDB copy.
  - Restore reads v1 blobs (inline queue) and v2 heads. On a rev mismatch (unload before the async IDB write landed) the older IDB queue is used if it contains the saved track; otherwise it falls back to just that track.
  - `addToQueue` / `removeFromQueue` now persist too (they didn't before).
- Performance mode toggle: Cinema (Experience section), EELA (row above lyrics size), ALOK (settings panel, under accent), Classic (Appearance group). i18n: `stPerfLite` / `stPerfLiteSub` (cinema dict) and `settings.perfLite` / `perfLiteDesc` (classic dict), EN/FA/TR/RU.
- Version: `package.json` + `src/version.ts` → 1.5.5 (version.ts had been left at 1.5.3).
- Type-check: no new errors vs. the V1.5.4 baseline (sandbox tsc 5.6 against stubbed deps; not built, not run).

### Phase 2 QA checklist
- [ ] `bun install && bun run build` (tsc clean in sandbox against stubbed deps)
- [ ] 5k+ track library: scroll EELA / ALOK / Classic / Download lists, check no gaps/jumps, clicks hit the right row
- [ ] Remove a big folder → one quick update, playlists/recents pruned
- [ ] Desktop folder import of a large folder: faster, correct order
- [ ] Minimize while playing → GPU/CPU near idle; restore → animations resume
- [ ] EELA + ALOK: visualizers, ring/progress, clocks and lyrics still move smoothly; switch faces back and forth → they resume (ALOK glow + RadialViz wake after a pause/play)
- [ ] Performance mode switch in each of the 4 Settings screens → blur/decorative motion off instantly, survives restart, flips in all faces
- [ ] Resume: queue of 5k+ tracks → close app mid-song → reopen: same track, same position, whole queue back (check DevTools › IndexedDB › nobody-db › session)
- [ ] Resume: small queue (< 2k) still restores; upgrading from a 1.5.4 profile keeps the last session
- [ ] Settings › Clear storage → no session restored after reload

## Phase 3 — Core playback (part 1, V1.6.0)
### Done: gapless + crossfade (engine faces: Cinema / EELA / ALOK)
- `engine.ts` now owns TWO `<audio>` elements. `engine.audio` is always the current one and is swapped at a transition, so every face (they read `engine.audio` live) follows. All element listeners are gated to the current element.
- WebAudio graph: each element → own fade `GainNode` → preamp → EQ → analyser → destination.
- `planTransition()` runs on timeupdate / play / seeked / ratechange. It pre-buffers the next item on the standby element `PRELOAD_AHEAD_S` (12 s) before the transition point, then arms a precise `setTimeout`, re-armed every timeupdate.
  - Gapless: start `GAPLESS_LEAD_S` (45 ms) before the end, 30 ms equal-power overlap.
  - Crossfade: `setValueCurveAtTime` equal-power in/out over `min(crossfade, dur/2)` of both songs.
- Fallback: if the timer loses the race (throttled background, tiny tail), `onEnded` starts the pre-buffered track at once; if nothing valid is prepped, the old `next(true)` runs.
- Automatic advance only (`autoNextPos`): no transition for repeat-one, sleep "end of track", end of queue, or repeat-all + shuffle wrap (that reshuffles). The prepped track is re-validated against the live queue at transition time, so queue edits and shuffle toggles are safe.
- Hard-cut (`cancelTransition` + `dropPrep`): `loadCurrent` (next/prev/jump/setQueue), `pause`, `seek`, `stopAndClear`, classic handoff (`engine.silence()`).
- Object URLs tracked per element (`urls` map). Volume and rate are applied to both elements; volume, mute and rate are copied to the incoming element at the swap.
- Settings: `gapless` (default true), `crossfade` 0–12 s (default 0; wins over gapless). UI: Cinema Settings → Playback section; EELA Settings rows; ALOK settings panel. i18n keys `stPlayback*`, `stGapless*`, `stCrossfade*`, `stCrossfadeOff` in EN/FA/TR/RU.
- Verified in a headless simulation of the engine (fake audio elements / AudioContext): gapless chain a→b→c, 1 s crossfade chain, manual Next mid-fade (outgoing cut), pause mid-fade, removing the prepped next track, repeat-one. tsc: no new errors. Not built, not run on Windows.
### Known limits
- Classic face is not covered (own `<audio>`; arrives with Phase 6 engine merge).
- ALOK's mute button writes `engine.audio.muted` directly, so a fading-out tail isn't muted mid-fade (at most 12 s).
- With the WebAudio graph unavailable, crossfade degrades to the gapless cut.
### Phase 3 QA checklist
- [ ] Live album / DJ mix, gapless on: no audible gap or click between tracks (FLAC/WAV cleanest; MP3 may show a few ms)
- [ ] Crossfade 6 s: smooth overlap, constant loudness, track title/cover/lyrics switch at fade start
- [ ] Press Next / seek / pause mid-crossfade → instant cut, no leftover audio
- [ ] Repeat one, sleep "end of track", last song with repeat off → no crossfade, same behavior as before
- [ ] Switch to Classic mid-crossfade → engine fully silent
- [ ] Minimized window: transitions still happen (fallback on ended)
- [ ] Volume, EQ and speed stay correct across a transition; media keys/SMTC show the new song

## Phase 3 — part 2a (V1.6.1): sound backend (engine + parsers), UI NOT wired yet
### Done
- `src/shared/replayGain.ts`: ReplayGain/R128 tag parsing, gain factor (track/album mode, preamp, fallback for untagged, clip guard by peak, clamp −24..+12 dB), ID3v1 genre table, `normalizeGenre` / `splitGenres`.
- `src/shared/audioFx.ts`: 6-band (old freqs) + 10-band layouts, 13 presets with explicit 6/10 curves (old 6-band values unchanged), `fitGains` resampling, preamp, `FxChain` (preamp → peaking filters, rebuilt on band-count change). Meant for engine AND Classic.
- `audioParser.ts`: genre (ID3 TCON incl. "(17)" refs + v2.4 multi-value, Vorbis GENRE, MP4 ©gen/gnre, WAV IGNR) and ReplayGain (ID3 TXXX, Vorbis REPLAYGAIN_*/R128_*, MP4 ----:com.apple.iTunes freeform).
- `cinema/lib/metadata.ts` (ID3 walker: TCON/TXXX, ID3v1 genre byte) → `importer.ts` stores `Track.genre` / `Track.rg`.
- Classic import fills `metadata.genre` / `metadata.replayGain`; bridge `listTracks` passes `genre` / `rg`; `bridgeTrackToTrack` maps them.
- `engine.ts`: graph is now element → ReplayGain gain → fade gain → FxChain → analyser. Per-element RG (both songs of a crossfade are levelled), set before the new source sounds; 30 ms ramp on live changes. New API: `eqFreqs`, `eqBands`, `eqPreamp`, `setEqBands(6|10)`, `setEqPreamp(db)`, `applyEqPreset(id)`, `setReplayGain({mode,preamp,fallback,preventClip})`, `replayGainPrefs`, `replayGainFactorNow`, `consumeSleepAtTrackEnd()` (for Classic). Engine subscribes to the settings store (EQ/RG keys) and library (rescan brings new tags).
- Sleep timer (timed) now also pauses Classic.
- Settings: `eqBands`, `eqPreamp`, `replayGain`, `rgPreamp`, `rgFallback`, `rgPreventClip` (defaults: 6 bands, 0 dB, RG off).
- i18n (cinema core dict, EN/FA/TR/RU): new presets, bands, preamp, ReplayGain, Sound/Equalizer, queue save/drag, sortAlbum/sortPlays/sortReverse, lbGenres/lbUnknownGenre.
- tsc: no new errors vs baseline (stubbed deps). Not built, not run.
### Remaining in Phase 3 (next round, in this order)
1. FxSheet UI (Cinema + ALOK): 6/10 toggle, dynamic `engine.eqFreqs`, shared presets, preamp slider, ReplayGain section. EELA: Sound card in Settings. (Today `EQ_FREQS` = 6 bands, so the old sheets still work but can't reach 10 bands / preamp / RG.)
2. Classic: repeat off/all/one, FxChain + RG on its `<audio>` (crossOrigin for app://), speed via `settings.speed`, sleep via `engine.startSleepTimer` + `consumeSleepAtTrackEnd()` in onEnded, Settings UI + classic i18n.
3. Queue drag-reorder (Cinema/EELA grip), `engine.moveQueueItem(from,to)`, play next in row menus, save queue as playlist.
4. Sort keys album/plays + reverse in Cinema/EELA/ALOK, `genres` library category (`splitGenres`).
### Part 2a QA
- [ ] Build clean; playback in Cinema/EELA/ALOK unchanged with EQ off / RG off (default)
- [ ] EQ presets in the existing FxSheets still sound the same as 1.6.0
- [ ] Crossfade still works (graph gained one gain node per element)
- [ ] Rescan a folder with ReplayGain-tagged FLAC/MP3; in DevTools `engine.setReplayGain({mode:"track"})` → loud tracks drop in level

## Phase 3 — part 2b (V1.6.2): Sound UI (Cinema / ALOK / EELA)
### Done
- `src/cinema/lib/useSoundFx.ts`: headless hook (narrow settings selectors + now-playing track `rg`) → `eqOn/bands/freqs/gains/preamp/activePreset/presets/rg/rgNow` + actions (`toggleEq`, `setGain`, `setBands`, `setPreamp`, `applyPreset`, `resetEq`, `setRg`). `fmtDb()` uses a real minus sign. All writes go through the engine API.
- `setBands` keeps a named preset: if the curve equals preset X, switching layout loads X's other-layout curve (exact round trip) instead of the resampled one; custom curves are resampled by the engine as before.
- Cinema `FxSheet`: local 6-band `PRESETS` removed → shared `EQ_PRESETS` (13). 6/10 toggle in the EQ header, preamp column (hairline-separated) left of the bands, dynamic `fx.freqs`, "Custom" badge, Reset (flat + preamp 0), double-click resets a slider. New "Volume leveling" section: Off/Track/Album, leveling boost (−12..+12), untagged level (−12..0), prevent clipping switch, now-playing dB readout (+ "no loudness tag"), rescan hint.
- ALOK `FxSheet`: same features in ALOK styling (`na-seg`, `na-chip`, `na-range`, mono labels, PRE column). ALOK previously showed only 4 presets; now all 13.
- EELA `SettingsView`: new "Sound" card under the main card (Equalizer + 6/10, band sliders, Preset chips, Preamp, Volume leveling mode, boost, untagged level, prevent clipping + now-playing readout in its subtitle, hint). CSS: `.ne-vert` vertical slider, `.ne-chip:disabled`.
- No new i18n keys needed (all added in 1.6.1). Version → 1.6.2.
- Verified: tsc 5.6 against the real react / zustand 5 / framer-motion / lucide typings (+ stubs for node & react-virtual): 26 errors before and after, zero new. SSR smoke render of all three UIs with a mock engine: 6 → 10 bands renders 6 → 10 band sliders + preamp, 13 presets, "Custom" after a resample, now-playing readout −6.5 dB for a −6.5 dB tagged track (peak 0.98), preset round trip jazz 6→10→6 exact, Reset → flat/0 dB. Not built, not run on Windows.
### Part 2b QA
- [ ] Cinema + ALOK FX sheet: 6/10 switch, all bands move and are audible, preamp works, preset chip highlights, Custom after a manual tweak
- [ ] 10 bands at narrow window width (ALOK sheet is 100vw on small screens) → no overflow
- [ ] EELA Settings › Sound mirrors changes made in Cinema/ALOK (and vice versa) after a face switch
- [ ] ReplayGain Per track on a tagged library → loud songs drop; readout matches; untagged slider affects untagged songs only
- [ ] RTL (FA): layouts mirror, dB values readable
## Phase 3 — part 2c (V1.6.3): Classic repeat / EQ / ReplayGain / speed / sleep
### Done
- Repeat: `RepeatMode = "off" | "all" | "one"` (was a boolean that only meant "one"). Default "all" = the old non-repeat behaviour (auto-advance wrapped). Persisted as `PersistedState.repeatMode`. Button cycles all → one → off (Repeat2 / Repeat1 icon, localized title), `R` key cycles.
- `handleTrackEnd()` (onEnded + the no-source fallback ticker): 1) `engine.consumeSleepAtTrackEnd()` → pause; 2) repeat one → loop; 3) repeat off, no shuffle, last song of Ur Queue (if playing from it) or of the library → pause; 4) otherwise `goNext()`. Manual Next still always wraps.
- `src/shared/classicSound.ts`: Classic's `<audio>` → ReplayGain GainNode → shared `FxChain` → destination, reading the SAME settings store as the engine (EQ block, RG prefs). Built lazily: only when EQ is on or RG mode ≠ off, at play time or the moment a feature is switched on while playing. Only for graph-safe sources: blob:, app:// with `crossOrigin="anonymous"`, same-origin http(s). Other schemes (legacy Tauri asset://) never build it (no EQ there, playback untouched). One MediaElementSource per element, ever.
- `<audio crossOrigin>` = "anonymous" for app:// (set before `src`), same as the engine's `setCrossOriginFor`; the app:// handler already answers CORS + Range.
- Speed: `settings.speed` → `playbackRate` + `defaultPlaybackRate` (survives src changes, re-applied on loadedmetadata), `preservesPitch`. Speed is now shared across faces.
- RG per track: `classicSound.setTrack(track.metadata.replayGain)` on every track change (jump, no ramp); pref changes ramp 30 ms.
- Sleep: timed already paused Classic (1.6.1, via `__NOBODY_BRIDGE__.pause`); "end of track" now works in Classic too.
- Classic Settings: new full-width "Sound" group (`SoundGroup`, uses `useSoundFx` + `engine.setRate` / `startSleepTimer` / `cancelSleepTimer`). Now-playing RG readout uses Classic's own track tags (the engine mirror's `currentId` isn't Classic's). Classic i18n: new `sound` section (31 keys + 13 preset names) in EN/FA/TR/RU. CSS at the end of `src/index.css` (`.sound-group`, `.classic-eq*`, `.sound-presets`, light-mode + ≤900 px single column).
- Verified: tsc no new errors (26 → 26, stub-related). Headless test of classicSound with a fake AudioContext: defaults build NO graph; speed 1.5 applied with pitch kept; EQ switched on mid-play builds the graph exactly once; RG −6 dB → 0.501; untagged fallback −3 dB → 0.708; 10-band curve reaches Classic's filters; app:// without CORS / asset:// → no graph, app:// with CORS → graph. Not built, not run on Windows.
### Part 2c QA
- [ ] Classic, EQ + RG OFF: playback identical to 1.6.2 (no WebAudio involved)
- [ ] Classic desktop folder (app://): songs still play and seek after the crossOrigin change
- [ ] Turn EQ on while a Classic song plays → sound changes, never goes silent; 10 bands + preamp audible
- [ ] ReplayGain Per track on tagged files in Classic → loud songs drop
- [ ] Repeat: all wraps at the end, one loops, off stops after the last song; setting survives a restart
- [ ] Speed 1.5× in Classic keeps pitch and survives Next; sleep "end of track" pauses after the song; 15 min timer pauses Classic
- [ ] Switch Classic ⇄ Cinema: both follow the same EQ / RG / speed settings
## Phase 3 — part 3 (V1.6.4): queue drag-reorder, play next, save queue
### Done
- `engine.moveQueueItem(from, to)` (LISTEN positions, `to` = final index). Shuffle off: the move is written into `queue` (order stays identity, `queueRev++`) so it survives resume and a shuffle on/off round trip; a legacy non-identity order (pre-1.6.4 swaps) is normalised into `queue` first. Shuffle on: only `order` changes. `pos` follows the playing song; playback is never interrupted. A pre-buffered next track is kept (its `prep.pos` re-pointed) if it's still next, else dropped; `planTransition()` re-runs while playing. `moveInQueue(pos, ±1)` (ALOK chevrons) now delegates to it — it used to swap `order` only, which resume and `setShuffle(false)` threw away.
- `engine.playNext(id)`: empty queue → play it; current → no-op; already queued AHEAD → moved to `pos+1` (no duplicate); otherwise inserted via `addToQueue(id, true)`. Used by row menus and the Cinema command palette.
- `useLibrary.createPlaylistFrom(name, ids)`: one store update + one IDB write, deduped, unknown ids dropped.
- `src/cinema/lib/queueActions.ts`: `saveQueueAsPlaylist(t)` (listen order, name "Queue · <Intl date/time in UI lang>", made unique, success toast `queueSavedAs`) and `playNextWithToast()`.
- `src/shared/dragReorder.ts`: `useDragReorder({count, onMove, scrollRef})` → `gripProps(i, label)`, `rowProps(i)` (`data-qi`, `data-drag-src`, `data-drop=before|after`), `rowKeyDown`. Pointer events + pointer capture on the grip (mouse/pen/touch, `touch-action:none`), edge auto-scroll (40 px zone, ≤16 px/frame), Esc / window blur / pointercancel / unmount cancel. Drop slot = pure `computeDropIndex()` over the MOUNTED rows by index, so it's virtualisation-safe. Keyboard: ↑/↓ on a focused grip, Alt+↑/↓ on a row; focus follows the moved grip. `src/shared/dragReorder.css` (imported in `main.tsx`): grab cursors, 2 px accent drop line, dimmed source row.
- Cinema `QueueSheet`: grip is live, Enter on a row jumps, new Save-as-playlist button (BookmarkPlus) in the header.
- EELA `QueueSheet`: rows are `div role=button` now (a grip can't live inside a `<button>`), grip + keyboard, Enter/Space jump (Space no longer reaches the global play/pause), Save button next to Clear.
- ALOK `QueueSheet`: Save button (chevrons kept; they now persist).
- Play next (ListStart icon, `playNext` key): Cinema `TrackList` ⋯ menu (top item), EELA library row (between like and add-to-queue; `.ne-track-row` acts column 84 → 104 px, 68 → 102 px ≤ 720 px), ALOK `TrackRow`.
- i18n: no new keys (`playNext`, `queueSave`, `queueSavedAs`, `queueDefaultName`, `queueDragHint` already existed in EN/FA/TR/RU since 1.6.1).
- Verified: tsc 24 → 24 errors (all stub/env related, zero new). Headless engine test (fake `<audio>`, real stores) 29/29: moves before/after/of the current song, shuffle-off writes `queue` + bumps rev, shuffle-on changes only `order`, legacy order normalised, move survives shuffle on→off, prep kept / re-pointed / dropped, all `playNext` cases (ahead → moved, new → inserted, next/current → no-op, behind → re-added, under shuffle), resume head carries the new order, `createPlaylistFrom` dedupe, 7 drop-index cases incl. a virtualised window. SSR smoke render: Cinema + EELA sheets render 4 rows / 4 grips / Save for a 4-song queue, ALOK renders Save. Not built, not run on Windows.
### Part 3 QA
- [ ] Cinema + EELA queue: drag a song by the grip up and down, incl. the playing one → drop line shows, order changes, music doesn't stop or skip
- [ ] Drag in a long queue → sheet auto-scrolls at top/bottom; Esc mid-drag cancels (and doesn't close the sheet)
- [ ] Touch screen / pen: grip drag works, the list doesn't scroll instead
- [ ] Keyboard: Tab to a grip, ↑/↓ moves it and focus follows; Alt+↑/↓ on a row
- [ ] Gapless / crossfade on: move a different song right after the current one near the end of a track → that song is the one that plays next, seamlessly
- [ ] Reorder, close app, reopen → same order; shuffle on then off → order kept
- [ ] Play next from Cinema ⋯ menu / EELA row / ALOK row / command palette → plays right after the current song; for a song already queued later it's moved, not duplicated
- [ ] Save queue as playlist in all three faces → new playlist in Playlists with the queue order; saving twice gives "(2)"
- [ ] EELA library table at narrow width: 3 action buttons fit, no overlap; RTL (FA) looks right
## Phase 3 — part 4 (V1.6.5): sort keys + reverse, Genres — Phase 3 complete
### Done
- `src/cinema/lib/trackSort.ts`: one comparator set for all faces. `TrackSortKey = recent | imported | title | artist | album | duration | plays`. `Intl.Collator(numeric, sensitivity:"base")`. Natural directions: recent → most recent first, imported → newest first, text keys A→Z, duration → longest, plays → most. Tie-breaks: title → artist → file; artist → album → file; album → artist → file (file path/name = track order proxy, there is no track-number field). `sortTracksBy(list, key, reverse, {playCounts, recentRank})` is stable, never mutates, and keeps "no value" items (never played under recent, empty artist/album) at the END in both directions, sorted by title. `recentRankOf(recents)`, `loadSortPref/saveSortPref` (validated against the face's allowed keys, corrupt JSON → default), `SORT_LABEL_KEY` (existing i18n keys).
- `libraryIndex.ts`: `genreGroups(tracks, unknownLabel)` via `splitGenres`: a multi-genre track lands in each genre, case-variants merge (first spelling wins), one genre counted once per track, untagged → `g:` group last. `"genres"` added to `LibraryCategoryId` / `LIBRARY_CATEGORIES` (after years → 12 categories).
- Cinema `LibraryView`: local `SORT_CMP` removed → `sortTracksBy` (memoised with `playCounts`); options imported/title/artist/album/duration/plays; Reverse button (ArrowDownUp, `aria-pressed`) beside the sort menu; persisted `nobody-cinema-sort`. "Recently added" is newest-first now (was `a.importedAt - b.importedAt`, oldest-first). Genres tile (Tags icon, `.nc-cat[data-cat="genres"]` recipe in cinema/index.css), group category + drill-down.
- EELA `LibraryView`: same shared sort with all 7 keys, Reverse button (shown where sort applies), persisted `nobody-eela-sort`; Genres chapter (ROMAN extended to XIV); plays column shown while sorted by Most played.
- ALOK `LibraryPanel`: 3-button segment → shared `SelectMenu` (7 keys) + Reverse + genre filter dropdown ("Genres · N" = all, then each genre with its count; hidden when nothing is tagged; a vanished remembered genre falls back to all). Persisted `nobody-alok-sort` / `nobody-alok-genre`. `.na-select` in alok/index.css maps the dropdown's generic tokens onto `--na-*`.
- i18n: no new keys (`sortAlbum`, `sortPlays`, `sortReverse`, `lbGenres`, `lbUnknownGenre` were added in 1.6.1; `sortBy` / `recentlyPlayed` existed). Version → 1.6.5.
- Verified: tsc 24 → 24 (stub-related only). Headless test 23/23: numeric + accent-insensitive title order, reverse, empty artist/album and never-played stay last in both directions, album/artist/file tie-breaks, imported newest-first, duration/plays, input not mutated, pref round trip + bogus/corrupt fallback, genre merge/multi-genre/untagged/unique keys/in-track dedupe, category position. 1.6.4 queue test still 29/29. SSR smoke (mock stores, stubbed virtualizer/animations): Cinema + EELA Genres views list the genres, genre drill-downs contain exactly the tagged songs, EELA plays column + pressed Reverse from a persisted pref, ALOK renders sort + Reverse + Genres filter. Not built, not run on Windows.
### Part 4 QA
- [ ] Each face: every sort option orders as labelled; Reverse flips it; choice survives a restart
- [ ] Cinema "Recently added" → newest import on top
- [ ] Songs with no album/artist stay at the bottom with Album/Artist sort, both directions
- [ ] Genres (Cinema tile, EELA chapter XII… check numbering, ALOK filter) on a tagged library; rescan older folders first so genres are read
- [ ] Multi-genre file ("Rock; Pop") appears in both; untagged under "Unknown genre"
- [ ] ALOK dropdowns: readable in void + graphite themes, open over the list (not clipped), RTL (FA) mirrors
- [ ] 5k+ library: sorting/switching genre stays instant
## Phase 4 — part 1a (1.7.0-dev): tag write-back core (main process) — UI NOT wired yet
### Done
- `electron/tags/` (plain Node, no deps): `common` (sources, patch validation, JPEG/PNG sniff, verify), `id3` (read v2.2/2.3/2.4 + v1, write v2.3 or keep v2.4, unknown frames kept, "discard on alter" honoured, v1 updated), `vorbis`, `flac` (blocks rebuilt, in place via padding, ID3 prefix kept), `ogg` (Vorbis/Opus, re-pagination, seq renumber + CRC on stream copy), `mp4` (ilst rebuilt, `free` absorbs size change, else moov rewrite + stco/co64 shift; fragmented refused), `index` (detect by bytes, per-file lock, plan → verify virtual result → in place or temp + fsync + re-verify + atomic rename with Windows retry).
- Fields: title, artist, album, albumArtist, year, genre (multi), track, disc, composer, comment; cover set (JPEG/PNG ≤12 MB) / remove / keep.
- IPC `tags:read` / `tags:write` (resolve `{ok, code}`: E_INVALID, E_UNSUPPORTED, E_READONLY, E_BUSY, E_VERIFY, E_NOT_FOUND, E_NO_SPACE, E_CORRUPT, E_TOO_LARGE); preload `readTags` / `writeTags`.
- Verified: `docs/tests/tags-run.cjs` against ffmpeg-made MP3 (v2.3+cover+v1, v2.4, untagged), FLAC (± cover), Ogg Vorbis, Opus, AAC M4A (moov end + faststart), ALAC: 252/252 — decoded-audio MD5 identical after every write, ffprobe sees the new tags/cover, Ogg page CRC/sequence valid, second edit in place, no temp leftovers, 5 parallel writes serialized, bad input rejected.
### Found (fix next)
- The app's own `audioParser.ts` reads NO tags from ALAC/M4A files that have a cover or a `free` atom inside `meta` (ffmpeg's ALAC original fails too) → must fix before the editor ships, or M4A edits look lost after a rescan.
- MP3 without Xing: parser's CBR duration counts the ID3 padding (6 s → 11.7 s after a big cover).
### Next (in order)
1. Fix the two parser issues above.
2. Shared `TagEditor` dialog (Cinema ⋯ menu, EELA row, ALOK row), i18n EN/FA/TR/RU, bridge `patchTags` in App.tsx, `engine` reload of the playing track after a rewrite, library-only mode for web/blob tracks.
### QA (Windows)
- [ ] Edit a song that is playing → keeps playing; file busy in another app → clear error, original intact

## Phase 4 — part 1b (1.7.0-dev): parser fixes (both issues from 1a)
### Done
- `audioParser.ts` MP4: moov is located by walking the exact top-level box chain (head → tail → random access) instead of "moov in 3 MB head, else grep the 1 MB tail". Root cause: when moov STARTED inside the head but its udta/ilst ran past 3 MB, mvhd was found, the tail retry never ran and tags were lost; a half-read cover was stored as a broken image. Items that get cut off are now skipped, never stored.
- Tail fallback scan no longer stops at the first false "moov" hit. QuickTime-style `meta` (no version/flags) is read.
- MP4 codec: stsd format was read at +8 (the entry size) and showed as "\0\0\0Z". It is now read at +12, so AAC/ALAC show correctly.
- New `ParseAudioOptions.readRange` + `desktopRangeReader(path)` (Electron, 8 MB chunks over fs:read-slice), passed from App.tsx for partial desktop records. A complete browser File gets random access automatically, so the Cinema importer now reads moov-at-EOF M4A too.
- ID3 tag bigger than the 3 MB head (e.g. a 5 MB cover from the editor) is fetched whole, so frames after APIC survive and the MPEG frame after the tag is still found.
- MP3 CBR duration now counts only first frame → audio end, minus ID3v1/APEv2. VBRI header supported. Also fixed: Layer II MPEG-2 samples/frame (1152).
- Verified (`docs/tests/parser-run.mjs`, Node 22 strip-types, ffmpeg fixtures; three modes: full File / desktop head+tail / head+tail+readRange): no-Xing MP3 with a 2.2 MB cover went from 143.7 s to 6.034 s (ffprobe 6.034256); 40 s MP3 matches ffprobe. ALAC with moov straddling 3 MB: tags + full cover. 2.2 MB cover at EOF: tags always, cover with readRange. Files written by electron/tags read back correctly. FLAC/Ogg/Opus/MP3 Xing/MP2/WAV output is identical to 1.7.0-dev.
### Next
1. TagEditor dialog (shared `src/shared/tagEditor/`, mounted inside each face root: CSS vars are scoped there, so no body portal). Entry points: Cinema TrackList ⋯ menu, EELA `renderRow` acts, ALOK TrackRow acts. i18n in the cinema dict (EN/FA/TR/RU).
2. Bridge `readTags` / `patchTags` in App.tsx: writeTags → re-parse the file (loadDesktopFiles + readRange) → patch the classic track in place (same id). Library-only mode for web/blob/WAV tracks.
3. Playback: for mode `rewrite`, set sourceUrl to `app://…?r=<n>` (the app:// handler strips queries). Engine `hotReload(id)`: preload standby element, seek ahead, 30 ms equal-power swap. Classic: one-shot loadedmetadata restores the position. No reload is needed for `inplace`.
4. CoverArt effect deps only include hasCover/isPlaceholder, so a replaced cover doesn't refresh. Add `coverKey` (from the bridge coverUrl) to Track + deps.

## Phase 4 — part 2 (1.7.0-dev.3): TagEditor dialog (Cinema / EELA / ALOK)
### Done
- `src/shared/tagEditor/tagService.ts`: `editability(track)` (desktop + absolute path + taggable ext → `file`; else `none: noFile | format`), `readTrackTags` / `writeTrackTags` over `electronAPI.readTags/writeTags`, renderer-side per-track in-flight guard, validation mirrored from `normalizePatch` (year, track/disc n or n/total ≤ 65535, 1000/8000 length caps), `diffFields` with normalised compare ("3 / 5" = "3/5", "Rock;Pop" = "Rock; Pop"), cover file sniff (JPEG/PNG magic bytes, ≤ 12 MB), E_* → i18n key map. After a successful write it calls the optional `bridge.patchTags(id, {path, mode, fields, hasCover, coverChanged})` (part 3).
- `src/shared/tagEditor/store.ts`: zustand `{trackId, face}`; `openTagEditor(id, face)`. Each face mounts its own `<TagEditorDialog face>` inside its root; only the pinned face renders (all panes stay mounted).
- `TagEditorDialog.tsx` + `tagEditor.css`: file is the source of truth (fields from tags:read, only changed fields on the wire). 10 fields (title, artist, album, album artist, year, genre, track, disc, composer, comment), inline hints/errors, changed-field marker, cover preview / choose / drop / remove / undo, file name + format + tag type chips. Read-only mode with a reason for web/blob tracks, WAV/AIFF/…, and read failures. Esc (asks when dirty), Ctrl/⌘+S or Ctrl/⌘+Enter saves, Tab trapped, focus returns to the opener, keys and drags never reach the face's window listeners (no transport shortcuts, no "drop to import" overlay). Local `--te-*` tokens mapped from each face palette; logical properties (RTL mirrors); single column ≤ 640 px; reduced-motion respected. ALOK wheel-volume ignores the dialog.
- Entry points: Cinema TrackList ⋯ menu "Edit tags…", EELA library row acts (pen icon), ALOK TrackRow acts (pen icon).
- i18n: 52 `te*` keys in the cinema dict, EN/FA/TR/RU. `desktopWindow.ts` types `readTags?` / `writeTags?`; `desktopBridge.ts` adds `TagWriteInfo` + optional `patchTags`.
- Version: `package.json` + `src/version.ts` → 1.7.0-dev.3.
- Verified: tsc 5.6 with real react / zustand 5 / framer-motion / lucide typings (stubs for node, react-virtual): 22 errors before and after, zero new (all stub/Tauri-module related). Headless Chromium run of the real dialog (esbuild bundle, mocked electronAPI) in all 3 faces: fields read from file, Save disabled when clean/invalid, year/number errors, no-op normalisation, Esc-when-dirty confirm, fake PNG rejected, E_BUSY message keeps the dialog open, success sends ONLY the changed field + cover bytes and toasts, zero keydowns reach window, Tab stays inside, WAV / no-path / E_NOT_FOUND read-only notes, no overflow at 1100 px / 520 px, RTL mirrored (cover on the right). Not built, not run on Windows.
### Known gap until part 3
- After Save the FILE is written but the library row, cover and playback don't refresh yet (`bridge.patchTags` isn't implemented in App.tsx). A rescan does not refresh existing rows either (it only imports new paths).
### Next (part 3, in order)
1. App.tsx `patchTags(id, info)`: loadDesktopFiles([path]) + parseAudioFileMetadata(readRange) → patch the classic track in place (same id, keep lyrics/accent/likes/queue), new cover blob URL (revoke old), notify mirrors. Library-only mode for web/blob/WAV tracks (title/artist/album/year/genre into the library only).
2. Playback: for `rewrite`, `sourceUrl` → `app://…?r=<n>`; engine `hotReload(id)` (standby element, seek, 30 ms equal-power swap); Classic one-shot loadedmetadata restores position. Nothing for `inplace`.
3. CoverArt: add `coverKey` to Track + effect deps so a replaced cover refreshes (today only hasCover/isPlaceholder are deps).
### Part 2 QA (Windows)
- [ ] ⋯ → Edit tags in Cinema, pen icon in EELA and ALOK → dialog opens in that face's style, FA mirrors
- [ ] MP3 / FLAC / M4A / Opus: change title + genre + cover → Save → re-open → new values read back from the file
- [ ] File open in another app / read-only file → clear message, dialog stays open, original intact
- [ ] WAV or a web-imported track → read-only note, no Save
- [ ] While the dialog is open: Space / N / P / arrows don't touch playback; dropping an image on the cover doesn't trigger import

## Open items carried to Phase 2+
See ROADMAP.md.

## Phase 4 — part 3 (1.7.0-dev.4): library + playback refresh after a tag save
### Done
- App.tsx `patchTagsFromFile` → bridge `patchTags(id, info)`: re-reads the file (loadDesktopFiles + parser + readRange) and patches the classic row IN PLACE (same id: likes, playlists, Ur Queue, recents, lyrics, engine queue stay). Text fields from the writer's verified read-back, genre/ReplayGain/size from the parser; merged onto the freshest row. Cover only touched when the editor changed it (new blob URL, old one revoked, palette/accent recomputed); a generated placeholder cover / placeholder lyric line follows a new title/artist.
- `rewrite` → `sourceUrl = app://…?r=<n>`. Classic: one-shot loadedmetadata restores the position. Engine `hotReload(id, url)`: the standby element opens the new URL, seeks ahead, waits for the live element to catch up, 30 ms equal-power swap; same queue position / played counter; next/seek/pause/load during it aborts; a failed open retries once, then leaves the live element playing. A pre-buffered next track pointing at the edited file is dropped and re-resolved. `inplace` → no reload.
- `sourcePathFromSourceUrl` strips `?r=`/`#` (Smart Fetch / Downloads sidecar paths).
- Cover bug: `Track.coverKey` (from the bridge cover URL); `rehydrateTracks` invalidates cached cover URL/backdrop/palette/accent once when it changes; added to deps in CoverArt, ALOK Aura, Cinema LyricsView palette, EELA NowPlaying/AmbientFullscreen backdrop, EELA Downloads thumbs. Media Session (SMTC) refreshes when the playing row's title/artist/album/cover changes.
- Verified: tsc 5.6 no new errors (27 → 27, all missing @types/react-dom / Tauri / vite stubs). Headless Chromium, real `<audio>` + WebAudio: hot reload while playing (max step 13 ms at 10 ms sampling, swap offset ±2 ms), paused swap, pause/next mid-reload abort, non-current id, 404 retry keeps playing, gapless after reload, and a REAL electron/tags rewrite (FLAC + 1400px cover) of the playing file. Full app bundle with mocked electronAPI: Classic and Cinema edits of the playing song → same id, new title/genre/cover, `?r=1`, keeps playing from the same spot. Not built, not run on Windows.
### Still open
- Library-only editing for web/blob/WAV tracks (dialog stays read-only for them).
- A folder rescan still doesn't refresh rows changed outside the app.
### Part 3 QA (Windows)
- [ ] Edit title + cover of the PLAYING song in each face → row, cover, now-playing, SMTC update; no gap, no restart
- [ ] Same for a big cover on MP3 (forces rewrite) and FLAC with padding (in place)
- [ ] Edit the NEXT song with gapless/crossfade on → transition still clean
- [ ] Liked / playlist / queue membership survives the edit

## Phase 4 — part 4 (1.7.0-dev.5): library-only edits + rescan refresh
### Done
- `src/shared/tagEditor/libraryOverrides.ts`: per stable track id, text fields in localStorage `nobody-tag-overrides-v1` (only changed fields, cap 5000, oldest dropped), cover bytes in its own IDB `nobody-overrides/covers`. Laid over the parsed tags at boot import and rescan.
- Editor LIBRARY mode for WAV/AIFF/…, web/blob imports and E_UNSUPPORTED/E_CORRUPT reads: title/artist/album/year/genre + cover, "Save to library", file never touched. Desktop rows persist; web rows are session-only (note says so). "Reset to file tags" when an override exists. Standalone web build writes its IDB library directly.
- Bridge `patchLibraryTags` / `resetLibraryTags`; App.tsx refactor: shared `readTrackFile` / `commitTrackPatch` / `reopenPlayback`.
- Rescan: `scanDesktopFolder` = walk + stat only. New files imported, deleted rows dropped, CHANGED files (size:mtime `sourceStamp`) re-read and patched in place (same id); unchanged files are not read at all. Unreachable folder keeps its rows (used to drop them all). Writer's size/mtime update the stamp, so our own saves are not re-read.
- Cover origin tracking (`coverOrigin` embedded/sidecar/fetched/user/fallback + `coverSig`): an external change only replaces embedded/placeholder covers, never Smart Fetch or user covers. Embedded lyrics follow the file; .lrc / fetched lyrics stay.
- i18n: 11 `teLib*` keys EN/FA/TR/RU. Version → 1.7.0-dev.5.
- Verified: tsc no new errors (27 → 27). Headless Chromium, full app bundle + mocked electronAPI over real files and the real electron/tags writer (`docs/tests/library-refresh-run.mjs`): 38/38 — WAV library edit + restart + reset, external FLAC rewrite while the engine plays (same id, new tags/cover, ?r=, no position jump), only changed/new files read, fetched cover kept, delete, unreachable folder, in-app write skipped on rescan, web track session-only.
### Not verified
- Classic face reload after an EXTERNAL change: the test harness started Classic's <audio> outside React, so it ended paused; the code path is the same one part 3 verified, but check it on Windows.
### QA (Windows)
- [ ] WAV: edit title + cover → restart → still there → Reset to file tags
- [ ] Edit a playing song in Mp3tag, rescan its folder → row updates, no gap (Cinema AND Classic)
- [ ] Unplug a USB music drive, rescan → rows stay
- [ ] Big library rescan is noticeably faster (nothing re-read when unchanged)
### Next
1. Smart playlists (rule engine) + ratings.
2. M3U/M3U8 import/export.

## Phase 4 — part 5a (1.7.0-dev.6): ratings + smart playlists — core + Cinema (EELA/ALOK views NOT wired yet)
### Done
- `cinema/lib/smartPlaylist.ts`: rule engine. 14 fields (title, artist, album, genre, year, rating, plays, liked, last played, date added, length, format, bitrate, file path), text/number/date/bool operators, match all/any, sort (all library keys + rating + seeded random), limit by songs/minutes/hours. Text compare is case/accent-insensitive and folds Arabic ي/ك → Persian ی/ک. Half-typed rules are ignored. `normalizeSmartDef` sanitises stored data. 6 templates (top rated, recently added, heavy rotation, never played, forgotten favourites, 50 random). Per-playlist memo cache.
- Library store: `ratings` (1–5, localStorage `nobody-ratings-v1`), `lastPlayed` (`nobody-last-played-v1`, written by `recordPlay`), `setRating`, `createSmartPlaylist`, `updateSmartPlaylist`; `Playlist.smart`; add/remove/move are no-ops on smart lists; `isManualPlaylist`; playlists sanitised at hydrate.
- `cinema/lib/usePlaylistTracks.ts`: `usePlaylistResolver`, `usePlaylistTrackIds`, `playlistTrackIds`, `useRating`.
- `trackSort.ts`: new `rating` key (highest first, unrated last both directions); added to Cinema / EELA / ALOK sort menus.
- `shared/rating/StarRating`: slider semantics, hover preview, click current star = clear, ←/→ (RTL-aware), Home/End, 0–5; never leaks clicks/keys to rows or face shortcuts. "quiet" mode for dense rows.
- `shared/smartPlaylist/SmartPlaylistDialog` (+ store, css on the tag-editor shell): name, templates, rules, sort/reverse/reshuffle, limit, live preview from the real engine, Esc/Ctrl+S/Tab trap, discard prompt. Mounted in all three faces.
- Cinema: Playlists view (New smart playlist, Smart badge, Edit rules, read-only rows, resolved counts/Play/Shuffle, stars on rows), TrackList stars + only manual playlists in "Add to playlist", Now Playing stars, command palette plays smart lists.
- i18n: 90 keys (`rt*`, `sp*`, `sortRating`) EN/FA/TR/RU. Version → 1.7.0-dev.6.
- tsc: no new errors vs baseline.
### NOT done yet (next session, in order)
1. EELA PlaylistsView + ALOK PlaylistsPanel/PlaylistDetail: resolver counts, New smart / Edit rules buttons, hide remove/add-current for smart lists; ALOK PlaylistPop → `isManualPlaylist`.
2. Stars in EELA NowPlaying + library row (album cell), ALOK InfoCard.
3. Cinema playlist rows: Enter key to play.
4. Tests: node test for the rule engine; Chromium run of the dialog in all faces. Nothing has been run yet.
5. Then M3U/M3U8 import/export.

## Phase 4 — part 5b (1.7.0-dev.7): smart playlists in EELA / ALOK, stars everywhere, rule-engine tests
### Done
- EELA `PlaylistsView`: resolved counts (smart + manual via `usePlaylistResolver`), Sparkles icon for smart lists, "New smart playlist" chip under the composer, Smart badge + songs · total time in the sheet header, "Edit rules", Shuffle, read-only note, no remove (×) on smart rows, stars (quiet) on rows, Liked shows the localized name. Enter only plays when the row itself has focus.
- ALOK `PlaylistsPanel` / `PlaylistDetail`: resolved counts + Smart label on cards, "New smart playlist" button, Smart chip + "Edit rules" in the detail header, "Add current" hidden for smart lists, no remove on smart rows, read-only note / "No songs match yet". `PlaylistPop` → `isManualPlaylist` (smart lists no longer offered).
- Stars: EELA NowPlaying (under the meta line), EELA library row (album cell = text + quiet stars; `.ne-track-album-txt`), ALOK InfoCard (new Rating row; CSS keeps the mono label style off the star spans; Esc still closes).
- Cinema playlist rows: Enter plays (row focus only).
- BUG (found by the new tests): desktop/bridged rows had `importedAt = 1e12 + order` (Sept 2001), so "Date added in the last N" and the Recently added template matched NOTHING on desktop. New `cinema/lib/addedAt.ts`: first-seen registry per stable id (localStorage `nobody-added-at-v1`, cap 60k); on the upgrade session (no registry yet) the file mtime from `sourceStamp` seeds it, afterwards new ids get "now". `Track.addedAt`; bridge passes `mtimeMs`; smart "added" uses `addedAt || importedAt`; ALOK InfoCard "Imported" shows it. `importedAt` (sort key) unchanged → library sort order unchanged.
- BUG: classic sends bitrate as "320 kbps" strings → bitrate rules never matched on desktop. `kbpsOf()` in `bridgeTrackToTrack` + string-safe `bitrateOf` in the engine.
- Tests: `docs/tests/smart-run.mjs` + `smart.spec.ts` (esbuild bundle, plain node): 110/110 — folding (accents, ي/ك, harakat, ZWNJ, Cyrillic), every text/number/date/bool op, all/any, half-typed rules ignored, ruleError, sort (rating unrated-last both ways, plays, recent, imported, seeded random stable/spread), limits songs/min/h, 6 templates, normalizeSmartDef junk, cache invalidation (ratings, likes, time), 20k tracks in ~50 ms, bridged bitrate/addedAt.
- tsc: 26 → 26, no new errors. Version → 1.7.0-dev.7.
### NOT run yet
- Chromium run of the dialog / views in the 3 faces (harness needs a @tanstack/react-virtual shim + Tailwind v4 CSS; not built this session). Nothing run on Windows.
### Next (in order)
1. Chromium UI run (all 3 faces, RTL): New smart → Create selects it, Edit rules, counts update on rating change, no × on smart rows, ALOK popover hides smart lists, stars keys don't hit transport shortcuts.
2. M3U/M3U8 import/export.
### QA (Windows)
- [ ] Upgrade from dev.6: "Recently added (30 days)" shows files by their date, not empty, not the whole library
- [ ] Bitrate ≥ 256 rule on a desktop library returns the MP3 320 / FLAC rows
- [ ] EELA + ALOK: create / edit / delete a smart list; rate in NowPlaying / InfoCard → list updates live

## Phase 4 — part 5c (1.7.0-dev.8): Chromium UI run of smart playlists + bridge-mirror fix
### Done
- BUG (found by the UI run): only Cinema subscribed to classic-library changes (AppSwitch mounts only the visible face). Booting straight into EELA or ALOK on desktop hydrated against classic's still-empty library (its folder import is async) → EMPTY library, and rescans / imports / tag edits never reached those faces. New `cinema/lib/bridgeMirror.ts` (`useBridgeMirror`, ref-counted, one catch-up pass on acquire) used by Cinema, EELA and ALOK; Cinema's inline copy removed.
- Cinema smart badge got `data-smart-badge` (EELA/ALOK already had it).
- UI harness `docs/tests/ui-harness/` (esbuild bundle of the real app, React 18 / Tailwind v3 approximation, react-virtual shim = render all rows, mocked electronAPI over `server.cjs`). `smart-ui-run.mjs`: 3 faces × EN/FA = **227/227**: dialog mounted in the face root, direction ltr/rtl, focus in, 6 templates, typing/Ctrl+S never reach window shortcuts, Tab/Shift+Tab trapped, no overflow at 1280/520 px, RTL label/close placement, Create selects the new list, rating → live count + order, no remove on smart rows, star keys RTL-aware and never hit transport/rows, Edit rules live preview, Esc dirty → discard prompt → keep editing, Save re-resolves, clean Esc closes + focus returns, Cinema/EELA Enter on a row plays, ALOK popover lists manual lists only, ALOK InfoCard stars (Home/End, Esc still closes), smart detail hides "Add current", no page errors.
- Version → 1.7.0-dev.8.
### Not run
- Windows. Classic face is not covered by the harness.
### Next: M3U/M3U8 (design decided, not coded)
- Main IPC: `playlist:pick-import` (open dialog .m3u/.m3u8, multi, return {path, bytes} ≤ 8 MB), `playlist:pick-export(name)` (save dialog, remembers the chosen path), `playlist:write(path, text)` (only that remembered path, .m3u8 UTF-8, .m3u UTF-8+BOM). Do NOT widen WRITABLE_EXTS.
- `src/shared/m3u/m3u.ts` (pure): parse (BOM, CRLF, #EXTINF attrs + "Artist - Title", file:// decode, http skipped, relative vs m3u dir, `\` root-relative), decode (UTF-8 fatal → windows-1256/1252/1251/1254 by lang, pick the one matching most library paths), match (normalized lowercase path → unique basename → folded artist+title ± 3 s), build (relative when same drive, CRLF).
- Missing-but-existing files: bridge `importPaths` treats paths as FOLDERS (importFolderPaths) → needs a file-level `importFiles(paths)` on the bridge first.
- UI: Import next to "New smart playlist", Export in each playlist header (smart/Liked export resolved songs), web build via `<input type=file>` + blob download; i18n EN/FA/TR/RU.
### QA (Windows)
- [ ] Boot straight into EELA and ALOK with a desktop library → songs appear (no Classic switch needed); rescan in EELA updates rows

## Phase 4 — part 6 (1.7.0-dev.9): M3U / M3U8 import + export
### Done
- `src/shared/m3u/m3u.ts` (pure): parse (BOM, CRLF/CR/LF, #EXTM3U, #PLAYLIST, #EXTINF attrs + "Artist - Title", #EXTART, file:// incl. UNC/localhost, http skipped, relative vs playlist dir, `\x` root-relative on Windows, drive-relative rejected), decode (UTF-8/UTF-16 BOM → fatal UTF-8 → windows-1256/1251/1254/1252 ordered by UI lang, library-match score picks), match (normalised path → unique file name → folded artist+title ±3 s), build (relative when same drive/share/root, CRLF, `#` guard), `.m3u` = UTF-8 + BOM.
- Main IPC `playlist:pick-import` (bytes ≤ 8 MB, ≤ 50 files), `playlist:pick-export` (save dialog, grants that path for 10 min), `playlist:write` (granted path only, once, .m3u/.m3u8, temp + rename). WRITABLE_EXTS / READABLE_EXTS untouched. Preload + `desktopWindow.ts` types.
- Bridge `importFilePaths(paths, {remember})` → `{added, ids}` (file-level, waits until listTracks sees the rows). New `PersistedState.looseFiles` (cap 20k): M3U-added files outside managed folders come back at boot.
- `src/shared/m3u/playlistIO.ts`: import (match → stat + import on-disk files → rehydrate → `createPlaylistFrom`, unique name "X (2)", no empty playlist), summary toast (added / not found / streams / duplicates), export of resolved songs (smart + Liked too), web build via `<input>` + blob download, shared busy flag.
- UI: Import next to "New smart playlist" (Cinema, EELA, ALOK list), Export in every playlist header. i18n 15 `m3u*` keys EN/FA/TR/RU. Version → 1.7.0-dev.9.
- Verified: tsc 27 → 27 (no new errors). `docs/tests/m3u-run.mjs` 55/55 (paths, file URLs, parse, 1256/1251 decode by lang and by library score, matching incl. ي/ك + ZWNJ folding, build, round trip, 20k in ~0.3 s). Chromium `ui-harness/m3u-ui-run.mjs` 3 faces × EN/FA = **201/201** (import order, 2 outside files imported, toast, export relative/CRLF/EXTINF, round trip, 1256 .m3u, BOM, too large / cancel / nothing found, 1280/520 px, RTL, restart keeps outside files + playlist).
### Not verified
- Windows (real dialogs, `C:\` paths on disk, rename over an existing file). Classic face has no M3U UI (its playlists are separate).
- smart-ui-run regression needs its original fixtures (Kooch/Glass with genre tags); with rebuilt fixtures the failures were all genre/fixture checks, not reproduced cleanly.
### Open question
- Classic "Add music files" still doesn't survive a restart (only M3U imports use looseFiles). Probably should.
### Next
1. Listening stats. 2. Last.fm / ListenBrainz scrobbling. 3. Duplicate finder. Then Phase 4 closes (1.7.0).
### QA (Windows)
- [ ] Import an old Winamp/foobar .m3u (ANSI Persian) and a VLC .m3u8 → right songs, right order
- [ ] Playlist pointing at songs outside the library → added, still there after restart
- [ ] Export → open in VLC / foobar / Windows Media Player

## Phase 4 — part 7a (1.7.0-dev.10): smart-UI regression re-run, Classic loose files, listening-history core (UI NOT wired yet)
### Done
- smart-ui-run re-run with rebuilt fixtures (`docs/tests/ui-harness/fixtures.sh`, 6 FLAC with genres): first 221/227. Two real causes:
  - Cinema "badge visible" ×2: TEST timing only (AnimatePresence mode=wait cross-fade still showing Liked after 300 ms; the list WAS selected). Test now waits for the badge.
  - EELA star keys ×2: REAL BUG. Playlist rows keyed `${id}-${i}` → rating a song in a smart list re-sorted it, remounted every row and dropped focus to <body>; the next Enter/Space hit the global play/pause. Keys are now id + occurrence. → **227/227**.
- Classic "Add music files" now remembers the picked paths in `looseFiles` (same list M3U uses) → they survive a restart. Shared `rememberLooseFiles()`.
- `src/shared/stats/listenLog.ts`: own IDB `nobody-stats/plays` (main DB version untouched), cap 200k rows, memory fallback, cached read, change emitter.
- `src/shared/stats/listenTracker.ts`: engine + Classic feed timeupdate samples; counts media time actually heard (steps ≤ 3 s; seeks/pauses excluded), closes on song change / repeat-one loop / ended / unload / close veil. `full` = > 30 s song heard ≥ 50 % or ≥ 4 min (Last.fm rule, reused by scrobbling), `skip`, < 5 s not stored. Open listen mirrored in localStorage `nobody-listen-open-v1` and recovered at boot (crash / unload beating the IDB write). Setting `statsEnabled` (default on).
- Classic plays now also count in Most played / last played / smart "plays" rules (5 s mark → `recordPlay`); before only engine plays counted.
- `src/shared/stats/stats.ts` (pure): ranges 7d/30d/90d/365d/all, total time, plays, skip rate, unique songs/artists, discoveries, top songs/artists/albums/genres (folded), per-day series, hour/weekday histograms, streak current/best, peak day, CSV export with formula-injection guard.
- Tests: `docs/tests/stats-run.ts` (esbuild → node) 9/9. Version → 1.7.0-dev.10.
### NOT done / not verified
- tsc not run this session (the ../stubs dir from earlier sessions is not in the zip). Next session: recreate stubs, check for no new errors.
- No stats UI yet. m3u-ui-run not re-run (its fixtures are not in the zip either).
### Next (in order)
1. Stats sheet (shared dialog on the nte shell, Cinema/EELA/ALOK): range picker, totals, day chart, top lists (click = play), hour/weekday, streak, "keep history" toggle, Clear, CSV export; i18n EN/FA/TR/RU; Chromium run.
2. Last.fm / ListenBrainz scrobbling on top of the `full` listens (offline queue).
3. Duplicate finder. Then 1.7.0.
### QA (Windows)
- [ ] Classic: Add music files → restart → songs still there
- [ ] EELA smart list: rate a song with the keyboard → focus stays on the stars, Space doesn't toggle playback
- [ ] Play songs in Classic and Cinema, kill the app mid-song → DevTools › IndexedDB › nobody-stats has the rows after restart

## Phase 4 — part 7b (1.7.0-dev.11): listening stats sheet (Cinema / EELA / ALOK)
### Done
- Type-check stubs recreated and now kept in the zip (`docs/tests/stubs/`, copy to `../stubs`). Baseline dev.10 = 23 errors, all env/stub (tauri, vite, react-dom types, Uint8Array generic). After this round: 23 → 23, zero new.
- `src/shared/stats/StatsSheet.tsx` + `statsSheet.css` + `sheetStore.ts`: one sheet on the .nte shell, mounted in each face root. Period 7d/30d/90d/12m/all (remembered, `nobody-stats-range`), cards (time, plays, songs, artists, new finds, skip rate, streak current/best), per-day bars (grid, gap shrinks for 120+ days), top songs/artists/albums/genres (click = play: songs → top list queue, groups → their library songs; songs gone from the library are listed but disabled), hour + weekday charts (week starts Sat in FA, Mon in TR/RU, Sun in EN), keep-history toggle, CSV export (UTF-8 BOM, blob download, Electron shows its save dialog), Clear with a confirm. Live updates while open. Esc / Tab trap / RTL-aware arrows on period + tabs, keys never reach the face.
- Entry points: "Listening stats" in the playlists rail of Cinema, EELA and ALOK + Cinema command palette.
- stats.ts: library placeholders ("Unknown Artist/Album") group with unknown and fall back to the library tag. tracker `discard()` (Clear history mid-song: no stale listen, Most played not double-counted).
- BUG fixed: Cinema Settings › Clear storage did not clear listening history (comment claimed it did) → now clears `nobody-stats` + the open-listen mirror.
- i18n: 47 `ls*` keys EN/FA/TR/RU (FA uses Persian digits + Persian calendar dates).
- Tests: stats-run 12/12 (3 new). `ui-harness/stats-ui-run.mjs` 3 faces × EN/FA = **273/273**. smart-ui-run regression **227/227**. Version → 1.7.0-dev.11.
### Not verified
- Windows. m3u-ui-run not re-run (its fixtures are still not in the zip). Classic face has no stats sheet (its plays ARE counted).
### Next
1. Last.fm / ListenBrainz scrobbling on the `full` listens (offline queue). 2. Duplicate finder. Then 1.7.0.
### QA (Windows)
- [ ] Open Listening stats in each face, switch periods, click a top song/artist → plays
- [ ] Export CSV → opens right in Excel (Persian titles readable)
- [ ] Clear history while a song plays → empty; Settings › Clear storage → history gone too

## Phase 4 — part 8a (1.7.0-dev.12): scrobbling core + duplicate finder core (UI NOT wired yet)
### Done
- `src/shared/scrobble/`: `md5.ts` (RFC 1321, verified vs node crypto), `services.ts` (Last.fm 2.0 signed client: getToken / browser auth / getSession / updateNowPlaying / scrobble ×50 with per-item ignored; ListenBrainz: validate-token / playing_now / single / import ×100, X-RateLimit-Reset-In), `scrobbler.ts` (offline-first per-service queue in localStorage cap 10k, `id@start` dedupe ring, backoff 30 s → 30 min, `online` retry, auth → keep queue + reconnect, 400 batch → one by one, Last.fm > 14 days dropped locally), `index.ts` (desktopFetch transport, secrets via safeStorage IPC, build-time `VITE_LASTFM_API_KEY/SECRET` or the user's own API account).
- listenTracker: `onClosed(rec)` hook (fires with history OFF too) + meta on `onGenuine`; scrobbling installed before `recover()` so a crash-recovered listen is sent.
- electron: allow-list + https-only for ws.audioscrobbler.com / api.listenbrainz.org; openExternal last.fm / listenbrainz.org; `secret:get/set/delete` (safeStorage, `scrobble.*` names only, userData/secrets.json 0600); `fs:trash` (audio files only, shell.trashItem = Recycle Bin, never a hard delete). Preload + desktopWindow types.
- `src/shared/dupes/`: `dupes.ts` (modes song / tags / file, title + artist folding incl. feat./remaster/track-number stripping, length clustering, quality-ranked keeper, `planMerge`, "not duplicates" memory), `excludedPaths.ts` (removed copies left on disk are skipped by loadDesktopFolders / scanDesktopFolder; un-excluded on Add music files / M3U / importPaths), `actions.ts` (`resolveDuplicates`: merge playlists/Liked/rating/plays/history into the keeper, playing song never removed).
- Library store `mergeTrackData(map)`; listenLog `remapPlayIds(map)`; bridge `removeTracks(ids, {trash, mergeInto})` in App.tsx (Recycle Bin first, failed trash keeps the row, Classic playlists remapped, looseFiles pruned).
- Tests: `docs/tests/scrobble-run.ts` (run via `node docs/tests/run-ts.mjs …`) **55/55**. tsc: see below.
### NOT done (next session, in order)
1. ScrobbleSheet (.nte shell; Last.fm connect / own API key fields, ListenBrainz token, pending count, Send now, now-playing switch) + entry points (Settings in 3 faces, stats sheet footer, command palette) + i18n `sc*` EN/FA/TR/RU.
2. DuplicatesSheet (mode seg, groups, keep radio, "not duplicates", Resolve all, Recycle Bin checkbox + confirm) + entry next to Listening stats + palette + i18n `dp*`; node test for dupes.ts; Chromium runs of both sheets.
3. M3U fixtures script (lib of 7 mp3, road.m3u8, windows-1256 fa.m3u, big.m3u8) in its own dir → re-run m3u-ui-run. Then 1.7.0.

## Phase 4 — part 8b/9b (1.7.0-dev.13): Scrobbling + Duplicates sheets wired (UI runs NOT done yet)
### Done
- `shared/scrobble/ScrobbleSheet.tsx` + css + `sheetStore.ts`: Last.fm (browser auth with wait/open again/copy link/cancel, own API key + secret form when the build has no key, remove own key), ListenBrainz (token + link to settings), per service: status pill, scrobble on/off, now-playing on/off, pending / sent / not accepted / last sent, offline note with next retry time, auth-lost note + Reconnect (queue kept), Send now, Disconnect (confirm, warns about unsent plays). No-tags counter, Last.fm rule in the footer. Esc / Tab trap / keys never reach the face.
- `shared/dupes/DuplicatesSheet.tsx` + css + `sheetStore.ts`: mode switch song/tags/file (remembered, RTL arrows), summary (groups · extra copies · space), per group keep radio (suggested = best quality), format/kbps/length/size/cover/lyrics/rating/plays chips, play-this-copy, Not duplicates (memory) + Show hidden, Remove others per group, Remove all extra copies, Recycle Bin checkbox (desktop only, remembered) + confirm, result toast. Playing song is flagged and kept. 60 groups per page.
- `dupes.ts`: `dupeTracksFrom()` (library rows → finder input), `fmtBytes()`; title key now folds ZWNJ/apostrophes and strips "07 " from file-name fallbacks.
- Entry points: Scrobbling in Settings of Cinema / EELA / ALOK (with live status), stats sheet footer, command palette. Find duplicates next to Listening stats in all 3 faces + command palette. `shared/sheetKeys.ts` (Tab trap, roving arrows, openExternalUrl).
- i18n: 101 keys (`sc*`, `dp*`) EN/FA/TR/RU.
- Tests: `docs/tests/dupes-run.ts` 58/58 (it found 3 real bugs: ZWNJ, "Don't" vs "Dont", numbered file names). scrobble 55/55, smart 110/110, m3u 55/55. tsc 23 → 23.
### NOT done (next session)
1. Chromium UI runs of both sheets (scrobble-ui-run with a fake Last.fm/LB httpFetch, dupes-ui-run with duplicate FLAC fixtures + trashFiles mock), 3 faces × EN/FA.
2. M3U fixtures script (t-m3u: 7 mp3 in lib/Album incl. "07 سلام.mp3", outside/Outside A+B, road.m3u8, windows-1256 fa.m3u, big.m3u8 > 8 MB) → re-run m3u-ui-run.
3. Then version 1.7.0.

## Phase 4 — part 10 (1.7.0-dev.14): M3U re-run, Scrobbling UI run, 3 real bugs fixed
### Done
- `docs/tests/ui-harness/fixtures-m3u.sh` (own dir `t-m3u`: 7 mp3 incl. "07 سلام.mp3", outside A+B, road.m3u8, windows-1256 fa.m3u, big.m3u8 > 8 MB). m3u-ui-run (T env, default t-m3u) 3 faces × EN/FA = **201/201**.
- Harness: fake Last.fm + ListenBrainz behind the http:fetch contract (mock.ts `fakeNet`, `__net` offline / approve / revoke), safeStorage stand-in (server `secret*`), `trashFiles` mock (moves to trash-bin outside the library, `__trashFail` → E_BUSY), `openExternal` log.
- `scrobble-ui-run.mjs` (77 checks per face/lang): own API key, browser auth (open again / copy / cancel / poll / approve), LB token, scrobble to both, offline + next retry + Send now, switches, now playing, revoked → Reconnect with queue kept, disconnect confirm with pending warning, secrets never in localStorage, layout 1280/520, Tab trap, Esc, live Settings summary, restart, stats footer + palette entries. Cinema EN **77/77**.
- BUG: Last.fm Connect stayed disabled forever after Cancel / any finished run in dev (StrictMode cleanup left `alive=false`). alive is set again on mount; Cancel frees the button at once; a superseded run can't touch the newer one.
- BUG: inline confirms (Disconnect, Remove duplicates, Clear history) did not get focus (`autoFocus` lost when the confirm replaced the focused button) → Esc/Tab started from <body>. New `focusOnMount` callback ref in sheetKeys.ts, used in Scrobble / Duplicates / Stats sheets.
- GAP: ALOK has its own command palette without the 1.7.0 sheets → added Listening stats / Scrobbling / Find duplicates. (EELA has no palette.)
### NOT done (next session)
1. scrobble-ui-run on EELA + ALOK × EN/FA and Cinema FA (one run showed a flaky focus check, re-run passed; re-check).
2. dupes-ui-run (duplicate FLAC fixtures + trashFiles mock already in the harness).
3. tsc 23 → 23 check after this round. Then 1.7.0.

## Phase 4 — part 11 (1.7.0): UI runs finished, Phase 4 closed
### Done
- scrobble-ui-run: Cinema / EELA / ALOK × EN / FA = **460/460** (EELA has no palette → 76 checks).
- NEW `ui-harness/dupes-ui-run.mjs` + `fixtures-dupes.sh` (own dir `t-dupes`, rebuilt per face/lang): 3 faces × EN/FA = **376/376**. Song/tags/file modes, RTL arrows + Home, keeper pick, Not duplicates + Show hidden (survives reopen), preview queue, Recycle Bin + confirm + Esc, inherited rating/plays/Liked/playlist, E_BUSY stays, playing copy kept, library-only removal not re-imported after restart, mode remembered, Tab trap, 1280/520, palette.
- BUG (flaky focus check, real): `focusOnMount` waited for a rAF (seen 120+ ms on a busy frame) → an early Esc/Tab went to <body>. Now focuses synchronously in the commit, rAF only as fallback.
- BUG: after "Not duplicates" / "Show hidden" the focused button vanished → focus on <body>, Esc/Tab dead. New `useFocusRescue(dialogRef)` in sheetKeys.ts (Scrobble, Duplicates, Stats sheets).
- BUG: desktop bridge sent `fileSize` as the "0.1 MB" label → duplicate finder "Same file" found nothing, "space to free" was 0, ALOK info card showed "—". Classic metadata now stores `bytes`; bridge sends exact bytes (fallback: sourceStamp); desktopBridge accepts numbers only.
- Duplicate chips show MP3 instead of the container name MPEG.
- tsc **23 → 23** (all env/stub). Unit: dupes 58/58, scrobble 55/55.
### Not checked
- stats-ui-run / m3u-ui-run not re-run after useFocusRescue + bridge bytes change (small, shared code).
- Windows build, real Last.fm / ListenBrainz, real Recycle Bin.

## Phase 5 — part 1 (1.8.0-dev.1): window state, Open with, tray, taskbar
### Done
- `electron/desktop.cjs` (new): argv parsing (audio + .m3u/.m3u8, relative → cwd, file:// URLs, `--nobody-cmd=play-pause|next|prev`, ≤ 500), window state read/write/fit to current monitors, per-user "Open with" reg.exe commands (HKCU only, OpenWithProgids REG_NONE, never UserChoice), tray (now playing, Play/Pause, Next, Previous, Show/Hide, Quit), thumbnail toolbar ⏮ ⏯ ⏭ (icons in electron/assets, @2x), jump list tasks.
- `main.cjs`: second-instance forwards files (focus) and jump-list commands (no focus); macOS open-file; files queue until `desk:ready`; `desk:read-playlist` only for playlists opened with NOBODY; close-to-tray (✕, Alt+F4, taskbar) with a one-time balloon, tray Quit plays the veil; window size/position/maximized remembered (never the mini-player or fullscreen size), restored only onto existing monitors; window icon resolved from dist/ (public/ is not packaged); taskbar title = song.
- `uninstall.cjs`: removes the "Open with" entries.
- Renderer: `shared/desktop/` (api, prefs store, integration, useDesktopRows). Labels in EN/FA/TR/RU, now playing from Classic (bridge effect) or the engine mirror, opened files imported by path (remembered) and played (Classic: `bridge.playIds`, rest → Ur Queue), playlists imported + played. Settings rows "Keep playing in the tray" + "Show NOBODY in Open with" in Classic / Cinema / EELA / ALOK.
- Tests: `docs/tests/desktop-run.cjs` **90/90** (part B boots the real main.cjs against a fake Electron). `ui-harness/desk-ui-run.mjs`: Cinema EN 27/28 (see below). tsc 23 → 23.
### Not done / open
1. desk-ui-run: the last check ("Open with" on after a failed write) fails: the 2nd deskSetPref is still busy at check time. Probably the harness mock (sessionStorage state), not yet confirmed. Run all 4 faces × EN/FA.
2. Windows: tray, thumbar, jump list, reg.exe registration never ran for real.
3. Next parts: configurable global hotkeys, Discord RPC, auto-update (electron-updater) + signing, installer hardening.

## Phase 5 — part 2a (1.8.0-dev.2): UI-run fix, Classic open-with stall, global hotkeys backend
### Done
- desk-ui-run last check: harness race, NOT an app bug. The switch flips optimistically; the check read the mock state before the 120 ms mock write finished. Now waits for `busy === null`. Settings rows waited on a fixed 500 ms (flaky) → waitForSelector.
- Real bug (Classic): nothing hydrated the library mirror in Classic, so `libraryReady()` waited the full 20 s before an "Open with" song played. `integration.ts` now hydrates it itself once the bridge is up.
- Harness: Classic boots/counts on the bridge (`listTracks`), disabled EELA switch clicked through the DOM.
- Result: desk-ui-run **222/222** (Classic / Cinema / EELA / ALOK × EN / FA) BEFORE the hotkey code; after it, Classic + Cinema EN re-run.
- Global hotkeys backend: `desktop.cjs` normalizeAccelerator / cleanHotkeys / createHotkeys (canonical "Ctrl+Alt+Shift+Super+Key", Ctrl/Alt/Win required except F13–F24, OS-reserved combos refused, duplicates refused). Off by default; defaults Ctrl+Alt+Space / ←/→ / ↑/↓ / N. Stored in desktop.json `hotkeys`.
- main.cjs: `desk:get-hotkeys`, `desk:set-hotkeys` (E_INVALID / E_DUPLICATE), `desk:hotkeys-suspend` (while recording; 20 s cap, reset on reload). show-hide → toggleMainWindow, rest → media:key. New media actions vol-up / vol-down (engine + Classic, 5 % steps).
- Renderer: `shared/desktop/hotkeys.ts` (store, recorder on KeyboardEvent.code = layout-independent, swap when a combo is reused), `HotkeyList.tsx` + `hotkeys.css`, "hotkeys" row in useDesktopRows, i18n hk* EN/FA/TR/RU. Version 1.8.0-dev.2.
### Not done / open
1. `<HotkeyList face=…/>` is NOT mounted yet in the 4 Settings screens, and their row icon/label ternaries still assume tray|openwith (the hotkeys row will show with the Open-with icon/label in the faces that hard-code it; Classic maps by id). Next: mount + fix mappings.
2. Mock in ui-harness/mock.ts lacks deskGetHotkeys/SetHotkeys/Suspend; desktop-run.cjs has no hotkey tests yet (fake globalShortcut needs register/unregister). Then full matrix again + tsc.
3. Windows: still never run for real (tray, thumbar, jump list, reg.exe, globalShortcut).
4. Remaining P5: Discord RPC (needs a Discord application client ID), auto-update (electron-updater not in bun.lock, needs internet) + code signing (needs a cert), installer hardening.

## Phase 5 — part 2b (1.8.0-dev.3): global hotkeys UI wired + tested
### Done
- `<HotkeyList/>` mounted under the "Global shortcuts" row in all 4 Settings screens (Cinema/EELA: own row + keyboard icon; ALOK: after the row; Classic: `lang` passed).
- Correction to part 2a note 1: it was CLASSIC that hard-coded tray|openwith (title + description) and showed "Open with" for the hotkeys row; Cinema/EELA only had the wrong icon (labels came from rowKeys). All fixed.
- Real bug (a11y): after saving a recorded combo, focus was lost (buttons are disabled while main saves, refocus ran too early). Now refocuses after the save settles.
- ALOK: monospace only on the key caps (mono spread Persian/Russian labels apart).
- Harness: mock.ts implements desk:get-hotkeys / set-hotkeys / hotkeys-suspend (E_INVALID/E_DUPLICATE/E_UNSUPPORTED, taken combos, save failure); entry exposes useHotkeys.
- Tests: `ui-harness/hotkeys-ui-run.mjs` **284/284** (4 faces × EN/FA: row title/icon, on/off, 7 defaults, recording + suspend, face shortcuts never fire while recording, plain Space / reserved / lone modifier, Persian layout "ح" → Ctrl+Alt+P, Esc, move-on-reuse, clear, taken, save fail, reset, persistence, unsupported main). `desk-ui-run` **222/222** again after the wiring. `desktop-run.cjs` **90 → 128** (A6–A8 accelerator rules / config / registration on a fake globalShortcut incl. media keys untouched; B6 main IPC end to end, reload mid-recording, desktop.json round trip + hand-edited file). tsc **23 → 23**.
### Phase 5 remaining
1. Discord RPC: needs a Discord application client ID from the owner.
2. Auto-update (electron-updater, not in bun.lock → install with internet) + code signing (needs a certificate). Plan together.
3. Installer hardening: per-user without UAC, reject junctions, async copy in a worker, kill only own PID.
4. Windows QA of everything in P5 (tray, thumbar, jump list, reg.exe, globalShortcut incl. AltGr layouts where Ctrl+Alt+letter may clash).

## Phase 5 — part 3 (1.8.0-dev.4): installer hardening (audit #8 #9 #10 #11 #12)
### Done
- No UAC for "Just me": setup is `asInvoker`, default scope = Just me (%LOCALAPPDATA%\Programs, HKCU). "All users" → note under the scope switch → Install relaunches the setup elevated (PowerShell Start-Process -Verb RunAs via -EncodedCommand) with the choices in one `--nobody-resume=<base64url>` arg (resume.ts, re-validated); the elevated copy goes straight to installing. UAC refused → stays on Destination, says nothing was installed (EN/FA/TR/RU). main refuses scope "all" unelevated (E_NEEDS_ELEVATION). Launch-after-finish from an elevated setup goes through explorer.exe (player never runs elevated). All-users shortcuts → ProgramData Start Menu + Public Desktop.
- Junctions/links: every folder + file checked with lstat right before it is written (safeMkdirUnder / safePlaceFile); dest itself never a link; elevated: no linked ancestors, never into a foreign non-empty folder (E-INPLACE-ELEVATED). Existing files are unlinked first + COPYFILE_EXCL (hardlink trap). Rollback/uninstall never delete through a link.
- Async copy: copy, hardlink, verify (streamed sha256) and payload staging are async (libuv thread pool). Measured: longest main-thread stall 6 ms vs 413 ms (whole install) before, 96 MB payload.
- Kill only our own install: process list with full image paths (CIM), only PIDs inside the target folder, never ourselves; graceful `NOBODY.exe --nobody-quit` first (player 1.8.0-dev.4+ honours it only from its own install dir; with nothing running it exits without a window), bounded force-kill fallback. Same in the player uninstaller (winproc.cjs; winproc.ts twin in setup).
- Player uninstaller: all-users install → relaunches itself elevated; in-place delete lines skip anything behind a link.
- Setup: sandbox:true + every IPC must come from the wizard window; the uninstall marker's stored installDir is no longer trusted (could redirect an elevated delete).
- Real bug: Setup's own uninstall mode ignored in-place ownership (removed the whole foreign folder). Fixed.
- Tests: docs/tests/installer-run.cjs **58/58** (new), desktop-run **132/132** (B7 --nobody-quit), setup-ui-harness/uac-ui-run **18/18** (real wizard UI, EN+FA). Setup tsc: no new errors (stub types). Player tsc 23 → 23.
### Open
1. Installing page shows NO message when an install fails (error only in console; chip falls back to "PREVIEW"). Needs an error card + "back to Destination". Next.
2. Runtime files are copied from the portable %TEMP% extraction and only size-checked; for an elevated install they should be hash-checked too.
3. Windows QA: UAC flow, junction refusal, --nobody-quit upgrade, all-users uninstall.
4. Parked by owner: Discord RPC, auto-update + signing.

