# Releases

## v0.2.45 — 2026-05-16

_Tag: [`v0.2.45`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.45)_

## v0.2.45 — match wins now count after long pauses + SOCD picker actually applies offline

Two real-world bugs from the forum, both "thing I configured silently does nothing" with a confusingly-cheerful UI:

### 1. Match wins not counting after a long pause (aprl, Spooder, pringle, Patrick, toki)

**Symptom:** a long pause mid-set, then subsequent wins don't update the W/L. aprl reported 12-5 instead of 15-5 after a ft5. Spooder reported "played a looooong set, but not a single match was recorded." pringle suspected the long-pause angle, called it.

**Cause:** when the hub WebSocket dropped (long-pause keepalive timeout is the classic trigger), `PollMatchOutcome` early-returned silently. Every subsequent match outcome was lost for the rest of the set. The local `results.csv` mirror was also gated behind the same early-return, so the per-user record drifted too.

**Fix:** outcomes now fall through on disconnect. Local CSV writes immediately (every match is captured offline-side, no matter what the hub is doing). The hub send is queued onto a per-launcher `pending_match_results` vector; on reconnect the K::Connected handler drains the queue and replays every entry. W/L/D + recent-matches refresh after the catch-up, so the UI snaps to the corrected totals right away.

Launcher restarts will still lose pending entries — but the local CSV already captured them, so the **per-user** record stays accurate even if the hub is permanently down.

### 2. SOCD picker doesn't apply offline (Froglet)

**Symptom:** the SOCD combo in the input binder appeared to save fine but the spawned game always ran at the default (Hitbox SOCD — L+R neutral, U+D = U wins). No matter what you picked, you got the default.

**Cause (two-fold):**
- The launcher wrote `FM2K_SOCD_MODE_P1` / `_P2` via `_putenv_s`, but the hook reads `FM2K_SOCD_MODE` (no suffix), AND `_putenv_s` writes the CRT env table — CreateProcess inherits the **Win32** env table. So even the suffixed vars never reached the child process.
- `StartOfflineSession` never set `FM2K_SOCD_MODE` at all; only the online K::MatchStart handler did. Offline play therefore always ran at the compiled-in default.

**Fix:** `::SetEnvironmentVariableA("FM2K_SOCD_MODE", ...)` from `LoadSocdState` (so the first launch inherits the saved setting) and from the combo change handler (live update). Online K::MatchStart still re-applies the role-resolved per-slot value on top, so host/guest each pick their own.

### Also in this build

- v0.2.44's launcher resilience (Ianthina's bad-UTF-8 manifest crash) is still in.
- Hook now emits valid UTF-8 manifests (game_id + paths via the wide Win32 APIs).
- Per-game input override actually loads offline (v0.2.43).
- Idle CPU/GPU vsync soft-cap (v0.2.42).
- Games-folder window has Browse buttons via SDL3's native folder picker.
- New: per-game input binder offline routing works for all FM2K games (was online-only).

### Known issues still being investigated

- Computed-delay can pick different values on the two peers when the link is jittery (Melancholy, toki). On stable links both peers see the same RTT and agree; on jittery links the means drift. **Workaround:** flip the launcher's Delay combo from "computed" to a fixed value (8–12 is a good starting range) to force symmetric delay until the fix lands.
- Replay desync past ~4000 frames and spectator-mid-battle determinism (para). Still open.
- Audio/music continues into CSS (toki). Acknowledged as a core 2dfm-rollback issue.

**Downloads:**
  - [fm2k_v0.2.45.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.45/fm2k_v0.2.45.zip) (9.2 MB)

---

## v0.2.44 — 2026-05-16

_Tag: [`v0.2.44`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.44)_

## v0.2.44 — hotfix: launcher startup crash from malformed upload manifest

**Reporter:** Ianthina [YURI] (game = Higurashi vs Touhou Universe 2)

**Symptom:** After a desync mid-match, the launcher crashed on every startup. Reinstalling didn't help. Only manually deleting the game's `upload_queue/` folder fixed it. v0.2.40 launchers worked; v0.2.41+ broke. Windows 10.

**Cause:** The hook in v0.2.41 wrote desync manifests using ANSI Win32 APIs (`GetModuleFileNameA` / `GetCurrentDirectoryA`). On Japanese-locale Windows with Japanese-named game folders, those return raw Shift-JIS bytes — not valid UTF-8. The manifest JSON ended up with mojibake in `file_paths` and `game_id`.

When the launcher's `PollUploadQueue` (called every render frame) picked up that manifest and tried to construct a `std::filesystem::path` from the bytes, MinGW's libstdc++ threw on the invalid UTF-8. The exception propagated out of `Render` → out of `SDL_AppIterate` → process abort, before the user could click anything.

**Fix** (launcher-side resilience):
- **UTF-8 validation gate** — before constructing any `fs::path` from a manifest's `file_paths`, the launcher now verifies every entry is well-formed UTF-8. If not, the whole manifest is quarantined and the launcher moves on.
- **try/catch wrapper around the upload-queue processor** — anything that does throw (filesystem error, I/O failure, anything else) is logged and swallowed. The launcher stays alive even if a future bad manifest slips past the validation gate.

Existing valid manifests from JP-locale users still upload as before — the gate only rejects malformed bytes, not Japanese content in well-formed UTF-8.

**For Ianthina + anyone else stuck:** updating to v0.2.44 is enough. The launcher will now scan, find the bad manifest, move it to `upload_queue/quarantine/`, and continue startup normally. No more manual deletion needed.

The hook-side fix (write UTF-8 manifests to begin with) ships in the next bigger release alongside the spec/desync refactor those changes are tangled with.

**Downloads:**
  - [fm2k_v0.2.44.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.44/fm2k_v0.2.44.zip) (9.2 MB)

---

## v0.2.43 — 2026-05-15

_Tag: [`v0.2.43`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.43)_

## v0.2.43 — hotfix: per-game input override now actually applies offline

**Reporter:** Sheriel [GUP].

**Symptom:** "Use override for `<game>`" forked the per-game `.ini` file and the launcher's binder appeared to save changes, but the changes never took effect once the game launched. Tested on URORFG and CC — same broken behavior on both. Editing the game's own in-engine controls menu also didn't help.

**Cause:** `Hook_GetPlayerInput` (the FM2K offline / CSS / battle code path) called the binder's lazy `Init()` but never called `SetGameProfile()` first. With no game stem set, `Load()` always resolved to the default `%APPDATA%\FM2K_Rollback\fm2k_inputs.ini` and the per-game `fm2k_inputs_<exe_stem>.ini` was never read.

Online play (GekkoNet) wasn't affected because it routes through a different function (`Input_CaptureLocal`) that already did the right thing. So this is purely an offline-play bug.

Side-note for the "in-game controls menu didn't help either" part: when the binder is active (which it always is, with our default bindings), our hook returns the binder's sample and the engine's own `get_player_input` never runs. That's by design — the in-engine controls menu is bypassed by the hook regardless of which profile is loaded. To customize, use the launcher's binder UI with the per-game override toggle.

**Fix:** wire `SetGameProfile()` + a periodic `Load()` into `Hook_GetPlayerInput`'s 1-second binder-active probe, mirroring the existing routing in `Input_CaptureLocal`. The exe stem is logged once on first resolution so field reports are easy to verify.

Drop-in safe; no protocol or save-state changes. Spectator / netcode / upload pipeline unchanged from v0.2.42.

**Downloads:**
  - [fm2k_v0.2.43.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.43/fm2k_v0.2.43.zip) (9.2 MB)

---

## v0.2.42 — 2026-05-15

_Tag: [`v0.2.42`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.42)_

## v0.2.42 — hotfix: launcher idle CPU/GPU

**Symptom:** users reported the launcher (no game running, just sitting on the panel) eating ~17–22% CPU **and** ~17–22% GPU on machines like Xeon E3 1230 v3 + GTX 1060 and on a 3060. That's wildly too much for static ImGui panels.

**Cause:** vsync was the only framerate cap. `SDL_SetRenderVSync(renderer, 1)` can silently fall back to a no-op path (RDP / headless / software renderer / driver refused), and presents on a minimized window complete instantly with no real swap. In either case the launcher spun `SDL_AppIterate` as fast as the CPU would allow.

**Fix** (in `SDL_AppIterate`):
- Window minimized/hidden → skip render entirely, `SDL_Delay(100)`. Events still pump via `SDL_AppEvent` so a restore wakes us up.
- Unfocused → soft-cap to ~30 fps.
- Focused but vsync didn't take → software 60 fps cap via `SDL_DelayNS`.
- Focused with working vsync → unchanged.

Also instrumented renderer init: the launcher log now reports the picked renderer name + vsync state, so a future "this still runs hot for me" report tells us which driver path the user landed on.

No spectator / netcode / upload-pipeline changes since v0.2.41 — those are still cooking. Update is drop-in safe.

**Downloads:**
  - [fm2k_v0.2.42.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.42/fm2k_v0.2.42.zip) (9.2 MB)

---

## v0.2.41 — 2026-05-13

_Tag: [`v0.2.41`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.41)_

## v0.2.41 — auto-upload diagnostics default on + cross-peer match grouping

### Auto-upload is now always-on

The dev-section toggle and Settings → Diagnostics tab are gone. Every
v0.2.41 build automatically uploads crash and desync reports to
`hub.2dfm.org/logs` so we can investigate user-side bugs without
having to ask each player for logs. The uploaded data is the same as
in v0.2.40: partially-redacted hook logs (IPs masked, no game footage,
no inputs), tagged with `session_id` / `match_id` / `client_version` /
`game_id` / `hook_dll_sha1`.

### Multi-dir scan

Previously the launcher only checked the *currently-selected* game's
`upload_queue/`. If a user crashed mid-match then opened the launcher
without re-selecting their game, the orphan report sat on disk forever.
The launcher now round-robins through every installed game's queue, one
manifest per tick, so reports always get sent regardless of what panel
the user is looking at.

### Server-side analytics for cross-peer desync analysis

When both peers crash on the same desync, they upload separately with
different `session_id`s but the same `match_id`. The hub now indexes
match_id and exposes:

- `GET /summary` — counts by kind/version/game + a top-10 list of
  matches where both peers uploaded (the high-signal desync diffs)
- `GET /by_match/<match_id>` — pull both peers' bundles for one match
- `GET /by_version/<v>` — every upload from one client build, for
  regression hunting after a release
- `GET /recent` now accepts `kind` / `version` / `game_id` /
  `match_id` / `since` filters

### Dev tool additions

`tools/fm2k_logs.py`:
- `summary` — first thing to run; shows what's worth investigating
- `pull-match <id>` — bulk-fetch both peers for one match
- `show-match <id>` — print both peers' meta + log tails interleaved,
  P1 first then P2, for direct inline LLM analysis
- `recent --version X --game Y --match Z` — server-side filtering

### What's not changed since v0.2.40

Spectator code path still on `FULL_SESSION` default. Next release will
wire `/F` boot-to-battle into the spectator launch path so mid-battle
joiners skip the visible title/CSS replay.

**Downloads:**
  - [fm2k_v0.2.41.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.41/fm2k_v0.2.41.zip) (9.2 MB)

---

## v0.2.40 — 2026-05-13

_Tag: [`v0.2.40`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.40)_

## v0.2.40 — auto-upload diagnostics + boot-to-battle + immediate desync termination

### New features

**Crash & desync auto-upload.** When the game crashes or GekkoNet detects
state divergence, the hook drops a manifest and the launcher posts the
diagnostic bundle (debug log tail, desync diff, RNG trace) to
`hub.2dfm.org/logs`. Off by default — opt in via the new dev-section
checkbox **"Auto-upload crash/desync diagnostics"**. Bundles are tagged
with `session_id` / `match_id` / `client_version` / `game_id` /
`hook_dll_sha1` for filtering. Launcher round-robin-scans every
installed game's `upload_queue/` so orphan reports get picked up even
if you launch the launcher without re-selecting your game.

**Immediate desync termination.** Previously the game played on through
desyncs — accumulating tens of cascading divergences until eventually
crashing in `character_state_machine` ~7000 frames later. Now the hook
calls `TerminateProcess` on the very first divergence, publishes a
distinct `FM2K_MATCH_OUTCOME_DESYNC` outcome, and the launcher shows a
"desync detected — match not recorded" toast instead of looking like
the game crashed for no reason. Set `FM2K_NO_DESYNC_KILL=1` to keep
the old behavior for diagnostic sessions.

**Boot straight to battle (`/F`, dev).** Engine has a built-in
`g_debug_mode=3` path that skips splash/title/CSS and dispatches a
battle-init game object directly. Hook detours
`InitializeGameFromCommandLine` to re-stamp `g_iniFile_nameOverride`
(which `hit_judge_set_function` otherwise clobbers with an empty
default on shipped binaries). New dev checkbox + P1/P2 char/stage/meter
inputs let you pick the exact matchup without touching CSS. ~150 ms
from launch to battle frame.

### Bug fixes

- **KOF retention HP-init jitter.** SafetyHook midhook at `0x411CB1`
  substitutes the winner's snapshotted HP into `eax` before the engine's
  `mov [ebx+0xDF05], eax` writes it, so the engine no longer briefly
  displays `max_hp` between rounds.
- **Damage multiplier doesn't break Vanpri stage scripts.** Previously
  the multiplier scaled all six `health_damage_manager` call sites,
  including the script-side mirror writes that Vanpri's stage triggers
  use for tightly-balanced self/opponent exchanges. Now the hook
  stack-walks the return address and only scales damage from
  `hit_detection_system`'s two actual hit-damage call sites.

### Misc

- Per-game patches v2: `team_css_dupe_lock`, `team_kof_retention`,
  `team_size`, `damage_multiplier_pct`, `gs_pic_fix` all driven by
  per-game INI under `%APPDATA%\FM2K_Rollback\game_patches\<game_id>.ini`,
  edited via the launcher's Patches panel.
- Story-init AI hijack midhook at `0x411C8F` for the 1P→VS-CSS
  sub-modes (VS CPU / Training / CPU vs CPU) so P2 actually gets AI
  fields instead of standing still.
- Stats site restyle + per-match rounds table.

### What's not changed since v0.2.39

Spectator code path is unchanged. FULL_SESSION remains the default;
CURRENT_MATCH (snapshot join) still opt-in via
`--spectate-mode current`. Next iteration will wire `/F` into the
spectator launch path so mid-battle joiners don't visibly walk title +
CSS before the snapshot applies.

### Dev tool

`tools/fm2k_logs.py` for pulling uploaded bundles:
- `recent --limit N [--kind X]` — list uploads
- `pull <session_id>` — rsync a session's bundles locally
- `show <session_id> --tail 200` — pull + print meta + log content
- `tail` — long-poll for new uploads

**Downloads:**
  - [fm2k_v0.2.40.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.40/fm2k_v0.2.40.zip) (9.2 MB)

---

## v0.2.39 — 2026-05-09

_Tag: [`v0.2.39`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.39)_

**v0.2.39 — spec hook now sends STUN probe**

Diagnosed via v0.2.38's `[spec] sent spec_incoming` log — the UDP port hub forwarded to the host (51148) was learned from an EARLIER game's hook STUN, not the spec hook itself. Spec's UDP socket has a different external port; host's punch went nowhere.

Two fixes:
- `Netplay_InitAsSpectator` now calls `SendStunProbe()` (was only in the player-path init).
- `FM2K_HUB_UDP_ADDR` + `FM2K_HUB_USER_ID` set process-wide at hub-connect time (was only in match_start handler — meaning a fresh spec inherited neither, so STUN couldn't fire).

This is the *real* fix for cross-NAT spec on cone NATs. v0.2.35-38 each closed one layer; this one closes the layer that was actually blocking the connect — spec's hook never registering its external UDP mapping at hub.

**Both spec AND host need v0.2.39.** Earlier-version hosts still won't fire the TCP punch on `spectator_incoming`. Spec gets a buddy update too.

**Downloads:**
  - [fm2k_v0.2.39.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.39/fm2k_v0.2.39.zip) (8.6 MB)

---

## v0.2.38 — 2026-05-09

_Tag: [`v0.2.38`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.38)_

**v0.2.38 — client_version sent on hello, logged on hub for diagnosis**

Adds `client_version` field to the hello WS message (sourced from `version_local.h`). Hub stores on `User.client_version` and prints alongside every operator-facing log line:

```
[+] Armonté (154454454001729537) v0.2.38 from ('108.197.57.133', ...)
[match] start abc12345 challenger(v0.2.38) vs target(v0.2.34) (game=pkmncc)
[spec] grant Armonté(v0.2.38) -> Suicidal Muffin(v0.2.34) same_nat=False host=...
```

When a cross-NAT spec attempt fails, the hub log now reveals "spec is on v0.2.38, host is still on v0.2.34" at a glance — that's the most common failure mode (host's hook on a build that doesn't fire the TCP punch). Pre-v0.2.38 clients omit the field; logged as `no_ver` so log scrapers still parse cleanly.

No client behavior change otherwise — purely additive.

**Downloads:**
  - [fm2k_v0.2.38.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.38/fm2k_v0.2.38.zip) (8.6 MB)

---

## v0.2.37 — 2026-05-09

_Tag: [`v0.2.37`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.37)_

**v0.2.37 — TCP-STUN DNS resolve timeout 1s → 5s**

Hotfix for v0.2.36: TCP-STUN was timing out on the resolve step alone.
SDL_net's async DNS resolver has cold-start overhead — first call after
process boot can take most of a second just spinning up the resolver
thread, exceeding the 1000ms cap.

Bumped NET_WaitUntilResolved to 5000ms. DNS itself is fast (UDP-STUN's
raw-getaddrinfo path completes in <100ms on the same host); the
overhead is purely SDL_net's first-call thread setup. TCP-STUN failure
is still non-fatal — we fall back to local listener port for the punch.

**Diagnostic note**: cross-NAT spec needs all three sides on a recent
client (spec, host, hub). If your spec attempt still fails after
updating, the host you're trying to spec is probably on an older
launcher build — they need to restart their launcher to pick up the
auto-update prompt.

**Downloads:**
  - [fm2k_v0.2.37.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.37/fm2k_v0.2.37.zip) (8.6 MB)

---

## v0.2.36 — 2026-05-09

_Tag: [`v0.2.36`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.36)_

**v0.2.36 — TCP-STUN env var fix**

Hotfix for v0.2.35: `FM2K_HUB_TCP_STUN_ADDR` was being set only on match_start, so it was undefined when users clicked Spectate before joining a match themselves. The spec hook logged `FM2K_HUB_TCP_STUN_ADDR unset — skipping` and TCP-STUN never ran.

Now set at hub-connect time alongside `FM2K_HUB_HOST` so every spawned game (player + spectator) inherits a valid value.

**Note**: TCP-STUN absence doesn't break v0.2.35's Tier 1 (TCP simultaneous-open punch) — that uses `local_tcp_port` regardless. This fix only affects users on symmetric / non-port-preserving NATs who were silently falling back to local-port punch (which fails for them). For most users on cone NATs with port preservation, no behavioral change.

**Reminder**: cross-NAT spec needs all three sides on a recent client (spec, host, hub). If your spec attempt still fails after updating, the host you're trying to spec is probably on an older launcher build.

**Downloads:**
  - [fm2k_v0.2.36.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.36/fm2k_v0.2.36.zip) (8.6 MB)

---

## v0.2.35 — 2026-05-09

_Tag: [`v0.2.35`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.35)_

**v0.2.35 — TCP simultaneous-open spec hole-punch + TCP-STUN**

Cross-NAT spectator connections were failing because UDP NAT punch only opens UDP — the actual INPUT_BATCH stream rides TCP, which uses an entirely separate NAT mapping. Spec's TCP SYN was getting dropped at host's NAT every time, leading to the "stuck on Connecting..." users were seeing.

**Tier 1 — TCP simultaneous-open punch**
- Vendored SDL_net gets `NET_CreateClientBound` (one new function): `SO_REUSEADDR + bind(local_port)` between socket() and connect().
- Hub forwards `spec_tcp_port` in `spectator_incoming` events.
- Host hook fires both UDP heartbeat *and* a raw-winsock TCP punch from listener_port → spec_ext:spec_tcp_port — opens host's NAT for the spec's incoming SYN.
- Spec hook sources its outbound TCP connect from its own listener port via `NET_CreateClientBound`.
- Works on full-cone, restricted-cone, and port-restricted-cone NATs (most consumer routers).

**Tier 2 — TCP-STUN against the hub**
For NATs that don't preserve port across translation (some symmetric NATs, certain CGNAT setups), local_tcp_port ≠ external_tcp_port and Tier 1 punches the wrong port.
- New hub TCP-STUN responder on port+2 (7713). Mirrors UDP STUN protocol but over TCP.
- Spec hook does an outbound TCP-STUN at init: connect from listener_port to hub:7713, reads back observed external addr.
- Result reported to hub via new `tcp_addr` WS message; hub prefers external_tcp_port over local_tcp_port in spectator_incoming.
- Coverage now extends to non-port-preserving cone NATs.

**Coverage limit**
True symmetric NATs (per-destination port randomization) still fail — v0.2.36 will add a hub-relayed TCP fallback tier (option 3 from the design discussion) that works on every NAT type at the cost of hub roundtrip latency.

**For comparison: CCCaster** (looked at the source) does direct TCP/UDP → UDP-via-VPS-coordinated-punch → fail. No data-relay tier. Our Tier 1+2 covers more cone-NAT cases than their approach; v0.2.36 will add the relay tier they don't have.

**Other**
- `vendored/SDL_net` converted from submodule to inline-vendored source. Submodule pointer was tracking a local commit that didn't exist on libsdl-org/SDL_net upstream; fresh clones would fail. Inline-vendored is the simpler ship.
- Hub deploy required: new `tcp_addr` message handler, TCP-STUN responder. Without it, clients fall back silently to v0.2.34 behavior.

**Downloads:**
  - [fm2k_v0.2.35.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.35/fm2k_v0.2.35.zip) (8.6 MB)

---

## v0.2.34 — 2026-05-09

_Tag: [`v0.2.34`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.34)_

**v0.2.34 — CSS color sync, rematch deadlock fix, crash recovery**

Bug fixes:
- **CSS palette sync**: replays/spectators now play with the *actual* color picked (was always color 0). Fixed both the MATCH_START header (hardcoded 0 → reads `g_charslot0_color_pick`) and the auto-confirm injected button bit (was always 0x10 → derives from recorded color).
- **BATTLE_ENTERING swap deadlock** (black screen on 2nd+ match): stale packets from prior matches arriving in the ~150ms window between `Netplay_EndBattle` and the new CSS session were poisoning `g_battle_entry_swap_frame`. Both peers then latched into a perpetual ping-pong echo that interfered with the battle GekkoNet sync handshake. Fixed via arm/disarm gates that only accept signaling between "CSS SYNCED" and "battle session started".
- **Stage_id accuracy**: MATCH_START / hub match_result were recording the *post*-random-roll stage (which is for the next match) instead of the actually-playing pre-roll stage. Now snapshots pre-roll.

New: crash mitigation
- **Unhandled-exception dumper** writes faulting address, registers, AV r/w/exec flag, bad address, and a module+RVA backtrace into the per-player log on any crash.
- **Vectored render guard** for the vanilla `sprite_rendering_engine` AV at 0x40CCAE (occasional alt-tab/title-drag during CSS demo) — recovers via early-return, game stays alive.

Quality of life
- **Replay browser** now shows configured games-root paths in the header + logs per-root scan results to help diagnose missing replays.
- **Offline replay** auto-terminates when the file finishes (was freezing on the last frame).

**Downloads:**
  - [fm2k_v0.2.34.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.34/fm2k_v0.2.34.zip) (8.6 MB)

---

## v0.2.33 — 2026-05-09

_Tag: [`v0.2.33`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.33)_

Released v0.2.33

**Downloads:**
  - [fm2k_v0.2.33.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.33/fm2k_v0.2.33.zip) (8.6 MB)

---

## v0.2.32 — 2026-05-09

_Tag: [`v0.2.32`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.32)_

v0.2.32 — NAT punch for spectators, quill async logs, JP support polish

Spectator
- **NAT punch coordination** — fixes "stuck on Connecting" for spectators
  outside the host's NAT. Hub now forwards a spectator_incoming event to
  the host carrying the spectator's external UDP addr; host fires a
  burst-punch toward it to open the NAT mapping; spectator's existing
  reconnect loop's next attempt traverses cleanly. Same-NAT loopback
  unaffected.
- Snapshot-join pre-WinMain crash fix (deferred apply via mode≥3000 gate)
- Stale-subscriber-slot reset on re-JOIN_REQ — fixes silent reconnect loop
- StashSnapshot guarded against rolling-back state

Performance / dev quality-of-life
- **Quill async logging** — hook DLL routes SDL_Log through quill's
  backend writer thread. Synchronous fprintf+fflush per line is gone.
- **Diagnostic backtrace ring** — [SPEC-FP], [HOST-FP], [SPEC-TRACE],
  [HOST-TRACE], pb_queue=N captured in a 4096-line in-memory ring;
  auto-flushed to disk only when LOG_ERROR fires. Great context on
  bug reports without continuous IO. FM2K_SPECTATOR_DEBUG=1 routes to
  disk for live diagnostic.
- **Console window hidden** for release builds (FM2K_DEV_MODE=1 keeps
  it visible)

User-facing
- **JP-renamed game support** — .kgt fallback in CreateFileA scans the
  game dir for any *.kgt when the hardcoded ASCII filename misses.
- **Manual delay can be 0** — for sub-1ms LAN / loopback / hotseat
- **Replay browser** Refresh button ID conflict fixed
- **Hooks queue+apply at boot** — single thread-freeze for the whole
  install sequence (faster boot)

Hub schema 2 (rolled in earlier this dev cycle)
- session_id + match_index_in_session + per-round mini-records on
  match_result. /sessions stats view aggregates by session.

**Downloads:**
  - [fm2k_v0.2.32.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.32/fm2k_v0.2.32.zip) (9.3 MB)

---

## v0.2.31 — 2026-05-08

_Tag: [`v0.2.31`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.31)_

v0.2.31 — replay browser, hub spectate, JP support, console hide, lots of fixes

User-facing
- **Replay browser** (Settings → Replays…) — Session → Match tree, click Watch to play back any .fm2krep / .fm2kset on disk
- **Hub Spectate button re-enabled** in the active-pairs lobby table — defaults to FULL_SESSION (replay-from-start) which is the proven-working path; CURRENT_MATCH (snapshot join) is opt-in via env var while it bakes
- **JP-renamed game support** — games whose .kgt was renamed to a Japanese basename now load via a directory-scan fallback in CreateFileA
- **Manual delay can be 0** — for sub-1ms LAN / loopback / hotseat play
- **Console window hidden** for release; FM2K_DEV_MODE=1 keeps it visible
- **Stats site /sessions** view groups matches by play-session with per-match round breakdown

Infra / fixes
- Offline replay playback works end-to-end (CSS auto-lock + INPUT pop gate)
- C10 schema 2 — match results carry session_id + match_index + per-round results
- Boot perf — batch MH_QueueEnableHook with single MH_ApplyQueued instead of per-hook freeze/thaw cycles
- Spec-debug log spam silenced by default (gated on FM2K_SPECTATOR_DEBUG=1)
- Snapshot-join crash fixes (deferred apply, stale-subscriber-slot reset, rolling-back gate) — kept in tree for the eventual unblock

Spectator testing TLDR
- Click Spectate on any active match in the lobby → walks through title + CSS naturally → enters battle. Works for typical-length matches; bf~8000 sim drift may show on very long sessions (fix queued).

**Downloads:**
  - [fm2k_v0.2.31.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.31/fm2k_v0.2.31.zip) (8.5 MB)

---

## v0.2.18 — 2026-05-08

_Tag: [`v0.2.18`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.18)_

v0.2.18 hotfix: one device per player; device picker swaps bindings (no more dual-device defaults)

**Downloads:**
  - [fm2k_v0.2.18.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.18/fm2k_v0.2.18.zip) (8.0 MB)

---

## v0.2.17 — 2026-05-08

_Tag: [`v0.2.17`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.17)_

v0.2.17 hotfix: don't auto-fill alt-slot xinput defaults over existing pad bindings

**Downloads:**
  - [fm2k_v0.2.17.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.17/fm2k_v0.2.17.zip) (8.0 MB)

---

## v0.2.16 — 2026-05-08

_Tag: [`v0.2.16`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.16)_

v0.2.16: dual input bindings — bind stick AND dpad to the same direction (or any kb+gamepad combo)

**Downloads:**
  - [fm2k_v0.2.16.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.16/fm2k_v0.2.16.zip) (8.0 MB)

---

## v0.2.15 — 2026-05-08

_Tag: [`v0.2.15`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.15)_

v0.2.15: pre-resolve hub IP in launcher so the in-game hook's DllMain never blocks on DNS

**Downloads:**
  - [fm2k_v0.2.15.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.15/fm2k_v0.2.15.zip) (8.0 MB)

---

## v0.2.14 — 2026-05-08

_Tag: [`v0.2.14`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.14)_

v0.2.14: inject timeout 5s -> 30s so Defender first-load DLL scan doesn't kill legit launches

**Downloads:**
  - [fm2k_v0.2.14.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.14/fm2k_v0.2.14.zip) (8.0 MB)

---

## v0.2.13 — 2026-05-07

_Tag: [`v0.2.13`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.13)_

v0.2.13: launcher pre-match STUN (hub finally has correct external ports before match_start) + hard-fail on handshake timeout (no more bleed-through to phantom CSS)

**Downloads:**
  - [fm2k_v0.2.13.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.13/fm2k_v0.2.13.zip) (8.0 MB)

---

## v0.2.12 — 2026-05-07

_Tag: [`v0.2.12`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.12)_

v0.2.12 hotfix: STUN id fix - hub now learns peer external port so hole-punch can land on real NAT mapping

**Downloads:**
  - [fm2k_v0.2.12.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.12/fm2k_v0.2.12.zip) (8.0 MB)

---

## v0.2.11 — 2026-05-07

_Tag: [`v0.2.11`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.11)_

v0.2.11 hotfix: build script kills running launcher before deploy so v0.2.10 fix actually lands in memory on next run

**Downloads:**
  - [fm2k_v0.2.11.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.11/fm2k_v0.2.11.zip) (8.0 MB)

---

## v0.2.10 — 2026-05-07

_Tag: [`v0.2.10`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.10)_

v0.2.10 — fix cross-NAT match handshake (launcher was setting peer_ip=127.0.0.1 + hook short-circuited relay)

**Downloads:**
  - [fm2k_v0.2.10.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.10/fm2k_v0.2.10.zip) (8.0 MB)

---

## v0.2.9 — 2026-05-07

_Tag: [`v0.2.9`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.9)_

v0.2.9 — FM2K_DEV_USER_SUFFIX env (allowlisted dc_ids only), self-matches refused at commit, stats site shows nick aliases per user.

**Downloads:**
  - [fm2k_v0.2.9.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.9/fm2k_v0.2.9.zip) (8.0 MB)

---

## v0.2.8 — 2026-05-07

_Tag: [`v0.2.8`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.8)_

v0.2.8 — hub migrated to hub.2dfm.org

**Downloads:**
  - [fm2k_v0.2.8.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.8/fm2k_v0.2.8.zip) (8.0 MB)

---

## v0.2.7 — 2026-05-07

_Tag: [`v0.2.7`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.7)_

v0.2.7 — auth diagnostics: real WinHttp error code in /pair/begin failures + auto NO_PROXY retry. Replaces the useless 'status=0' message.

**Downloads:**
  - [fm2k_v0.2.7.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.7/fm2k_v0.2.7.zip) (8.0 MB)

---

## v0.2.6 — 2026-05-07

_Tag: [`v0.2.6`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.6)_

v0.2.6 — auth UX fix.

When ShellExecute("open") silently fails (admin process / AV blocking
/ no http handler), the Discord auth modal now auto-copies the URL to
the clipboard on first detection and flips the status text to an
explicit yellow warning telling the user to paste with Ctrl+V. The
Copy + Reopen buttons added in v0.2.5 are still there as manual
controls — the auto-copy makes the fallback impossible to miss.

User-reported case: launcher could reach /pair/begin and got a real
authorize_url, but ShellExecute returned without launching the
browser. v0.2.5 surfaced the URL but the user didn't notice the
small Copy button. v0.2.6 puts the URL on the clipboard automatically
so Ctrl+V works immediately.

**Downloads:**
  - [fm2k_v0.2.6.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.6/fm2k_v0.2.6.zip) (8.0 MB)

---

## v0.2.5 — 2026-05-06

_Tag: [`v0.2.5`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.5)_

v0.2.5 — small QoL release on top of v0.2.4.

Auth:
- Discord auth modal now surfaces the URL with Copy/Reopen buttons
  when the OS browser auto-launch fails (admin process / no http
  handler / AV-blocked spawn). Hub also tries `cmd /c start` as a
  fallback before giving up.

FM95:
- match outcome now reports correctly (round-win-counter comparison
  mirroring obj_match_result_state, replacing the FM2K HP path that
  doesn't apply on FM95).
- vs/story-mode random-stage works via a LoadStageFile_alt trampoline
  hook that rewrites the function's first arg from the rolled stage_id.
- IDA-verified address map for the per-player main-object-ptr array,
  win counters, damage taken, and selected-stage globals.

HUD:
- in-game HUD scaffolding continues but ships HIDDEN by default. F9
  reveals both the dev overlay and the in-game HUD; the default-off
  state ensures no half-finished UI is exposed to users until the
  chat layer + delay/score wiring are ready.

Misc:
- wndproc_subclass install log now includes the captured prev_wndproc
  pointer so dual-client local testing can confirm both launchers
  wired their own subclass chain (not crossed-process).

**Downloads:**
  - [fm2k_v0.2.5.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.5/fm2k_v0.2.5.zip) (8.0 MB)

---

## v0.2.4 — 2026-05-06

_Tag: [`v0.2.4`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.4)_

v0.2.4 — locales/ + FM95Hook.dll now bundled in release zip; AUMID + dedupe peer-disconnect toast

**Downloads:**
  - [fm2k_v0.2.4.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.4/fm2k_v0.2.4.zip) (7.9 MB)

---

## v0.2.3 — 2026-05-05

_Tag: [`v0.2.3`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.3)_

v0.2.3 — temporarily disable game-data hash check (was rejecting peers with byte-identical files due to a canon-build corruption we haven't pinned down yet). Set FM2K_HASH_CHECK=1 env var to re-enable locally for testing.

**Downloads:**
  - [fm2k_v0.2.3.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.3/fm2k_v0.2.3.zip) (5.4 MB)

---

## v0.2.2 — 2026-05-05

_Tag: [`v0.2.2`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.2)_

v0.2.2 — relay engages only when direct UDP demonstrably fails (handshake-success gate); hash-mismatch popup now finds the manifest even when the game's hook log lives in Windows VirtualStore; on-mismatch manifest dump bypasses the cached-string parser to avoid an entry-render corruption observed in v0.2.1.

**Downloads:**
  - [fm2k_v0.2.2.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.2/fm2k_v0.2.2.zip) (5.4 MB)

---

## v0.2.1 — 2026-05-05

_Tag: [`v0.2.1`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.1)_

v0.2.1 — hash-mismatch UX: loosened .exe filter so installer leftovers / launchers / antimicrox no longer trip cross-install mismatches; launcher now opens a popup showing the local manifest (filename | size | content_hash) so users can diff against their peer's.

**Downloads:**
  - [fm2k_v0.2.1.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.1/fm2k_v0.2.1.zip) (5.3 MB)

---

## v0.2.0 — 2026-05-05

_Tag: [`v0.2.0`](https://github.com/Armonte/fm2ktest/releases/tag/v0.2.0)_

v0.2.0 — hub matchmaking + in-place rematches + stats site + FM95 + cnc-ddraw integration. See https://github.com/Armonte/wanwan/releases/tag/v0.2.0 for the full notes.

**Downloads:**
  - [fm2k_v0.2.0.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.2.0/fm2k_v0.2.0.zip) (5.3 MB)

---

## v0.1.9 — netplay parity for [EB] palette/shake + spectator TCP — 2026-05-05

_Tag: [`v0.1.9`](https://github.com/Armonte/fm2ktest/releases/tag/v0.1.9)_

## What's new

**[EB] palette flash and screen shake now look identical between offline and online.** Pre-0.1.9 the renderer's per-frame mutations (palette interpolation, shake offsets, render-frame counter parity) were getting clobbered by the rollback save/load path, so attacks like Bewear 214B, Tyrogue's fade-to-black, slither wing 6A landing, URORFG Loader's screen shake on every hit, and Breloom's super-fade looked subtly wrong online. Six independent state-leak fixes across the GekkoNet save/load + render-side state-protection paths now make the visual state evolve like vanilla 2DFM.

**Spectator stream moved to TCP.** The host→spectator INPUT_BATCH stream now rides a dedicated TCP channel instead of hand-rolled UDP redundancy + INPUT_REQUEST recovery. Packet-loss-induced spectator desyncs that occasionally surfaced are gone — TCP delivers each frame in order exactly once.

**Lobby gets tier-aware name colors.** Tester ($5+) names render in blue, Special Thanks ($10) in gold, and the maintainer in red. Hub side resolves your tier from your Discord roles at OAuth time.

**Logs moved to a tidy folder.** All hook diagnostics (debug logs, [EB] diag, RNG trace, parity recorder, replay-diff log) now write to `<game_dir>/logs/` instead of cluttering the game folder root.

**Rollback determinism preserved.** The rng-handling refactor includes a post-render back-patch into the saved slot so forward sim and rollback replay see identical rng state. Verified across multi-frame netplay rollbacks — no rng-region forward/replay diff in desync logs.

## Upgrade notes

The auto-updater will fetch this build the next time you launch. If you previously had the EB diag toggle on under Developer Mode, your toggle state persists across the update via `%APPDATA%\FM2K_Rollback\dev_flags.ini`.

**Downloads:**
  - [fm2k_v0.1.9.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.1.9/fm2k_v0.1.9.zip) (5.1 MB)

---

## v0.1.8 — 2026-05-04

_Tag: [`v0.1.8`](https://github.com/Armonte/fm2ktest/releases/tag/v0.1.8)_

Granular diagnostics for the most common bug reports.

**Injection failures.** The vague "DLL injection timeout or failed: 258" line now turns into one of three actionable messages depending on what actually went wrong:

  - **Game crashed mid-injection** — usually SmartScreen/AV killed it. Suggests unblocking + adding an exclusion.
  - **LoadLibrary hung past 5s** — DLL is blocked (Mark of the Web), antivirus scanning, missing runtime, or non-ASCII path. Suggests right-click → Properties → Unblock.
  - **LoadLibrary returned NULL** — DLL was rejected outright. Same set of suggestions, all spelled out in-message.

The launcher's console now also emits `Inject: …` lines as the injection progresses so we can see how far it got before failing. Polled in 250 ms increments instead of one big 5s wait, so a process exit during injection is caught immediately.

**WM_QUIT diagnostics.** The trampoline's `WM_QUIT — exiting` log now includes wParam + hwnd. wParam=0 is normal close; anything else means the game's own startup code asked to quit (DirectDraw/DirectInput init failure on that user's GPU/AV environment is the typical cause). Helps users with a 4ms-after-launch crash get a useful bug-report line.

This release is **diagnostics only** — no behavior changes. If you're already on v0.1.7 and things work, you don't need to update; this just makes failure modes legible for users hitting them.

**Downloads:**
  - [fm2k_v0.1.8.zip](https://github.com/Armonte/fm2ktest/releases/download/v0.1.8/fm2k_v0.1.8.zip) (5.0 MB)

---

