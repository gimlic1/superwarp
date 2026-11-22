# FFXI Addon — superwarp

> [!NOTE]
> **Quick Reference**
> - **Path:** `C:\Windower\addons\superwarp\`
> - **Author / Version:** Akaden / `0.97.8`
> - **Commands:** `//sw`, `//superwarp`, plus direct system commands `//hp`, `//wp`, `//pwp`, `//sg`, `//ew`, `//un`, `//ab`, `//po`, `//vw`, `//spd`
> - **Load Status:** `scripts/init.txt`
> - **Origin:** Third-Party (`https://github.com/AkadenTK/superwarp.git`)

Multi-system teleportation addon that automates menu-driven FFXI warps (Home Points, Waypoints, Survival Guides, Escha/Reisenjima, Unity, Abyssea, Runic Portals, Voidwatch, Proto-Waypoints, and Spatial Displacements) with fuzzy zone matching and Windower IPC multi-box coordination.

## What the Code Does
- **`superwarp.lua` & `map/*.lua`:** Registers 10 warp backends from `map/maps.lua`, each defining `short_name`, `npc_names`, `warpdata`, optional `sub_commands`, `missing` unlock-bit scanners, `validate`, and `build_warp_packets`.
- **Fuzzy Resolution (`fuzzyfind.lua`):** `resolve_warp` checks `settings.shortcuts` first, then calls `fmatch()`—combining exact substring match (`#needle + 1`), Longest Common Subsequence (`LCS`), and Levenshtein distance (`LEV`)—to resolve partial zone and sub-zone names (`e` → `Entrance`, `ah` → `Auction House`, `mh` → `Mog House`, `fs` → `Frontier Station`, `ed` → `Enigmatic Device`) or `settings.favorites`.
- **Action Queue Engine:** `poke_npc` injects an outgoing `0x01A` poke on the nearest matching NPC within `6` yalms (`dist <= 6^2`), captures the incoming menu packet (`0x034` / `0x032`), validates unlock/currency/zone state, builds a sequential `action_queue` of outgoing packets (`0x05B`, `0x05C`, `0x016`, `0x114`) paired with expected `wait_packet` replies (`0x05C`, `0x052`) and optional simulated human delays (`wiggle_value`), and drives them via `perform_next_action()`.
- **Safety & Reset:** Cancels any active warp if the user manually pokes an NPC (`0x01A` non-injected), retries on event-skip (`0x052` message type `2`) or timeout up to `max_retries`, optionally locks/unlocks client state (`0x037` `_flags3` bit `0x02` and `0x058` target injection), and releases stuck menus (`reset()`) via incoming `0x052` injections and outgoing `0x05B` exit (`_unknown1 = 16384`).
- **Multi-Box IPC (`sendall.lua`):** Discovers active local Windower instances via `marco`/`polo` IPC messages, orders participants (`melast`, `mefirst`, or `alphabetical`), and dispatches staggered `execute <player> <cmd>` messages spaced by `send_all_delay`.

## Commands & Usage
- `//[sw] <system> [warp|w] [all|party] <zone> [sub_zone]` — Warp via `hp`, `wp`, `pwp`, `sg`, `ew`, `un`, `ab`, `po`, `vw`, or `spd` (also intercepted directly via `unhandled command` without `//sw`).
- `//[sw] <system> [all|party] missing [max]` — Inspect menu unlock bitmasks and print locked destinations.
- System sub-commands: `hp set`; `ew enter` / `ew exit` / `ew domain` / `ew domain return`; `ab enter` / `ab exit`; `po assault` / `po return`.
- `//sw cancel [all]` / `//sw reset [all]` — Abort active warp and inject client menu-release packets (locally or across all IPC instances).
- `//sw debug` — Toggle debug logging and dump the buffered `debug_stack` from the last attempt.

## Events, Packets & Technical Hooks
- **Windower Events:** `addon command`, `unhandled command`, `ipc message` (in `sendall.lua`), `incoming chunk`, `outgoing chunk`, `zone change`, `unload`.
- **Packets:**
  - Outgoing: `0x01A` (NPC interact), `0x05B` (menu selection/exit), `0x05C` (same-zone warp coordinates), `0x016` (entity update/zone prep), `0x114` (map/menu request).
  - Incoming: `0x034` / `0x032` (menu open & unlock parameters), `0x052` (event release/skip), `0x05C` (menu sub-state), `0x037` (player flags for `client_lock`), `0x058` (local target set).
  - Injected: Outgoing `0x01A`/`0x05B`/`0x05C`/`0x016`/`0x114`; incoming `0x052`, `0x058`, `0x037`.
- **Dependencies / Libraries:** `tables`, `logger`, `functions`, `packets`, `coroutine`, `config`, `resources`, `sendall`, `fuzzyfind`, `map/maps` (and 10 `map/*.lua` modules).

## Local Settings & Customizations
From `data/settings.xml`:
- `max_retries = 10` (bumped from code default `6`), `retry_delay = 2`, `default_packet_wait_timeout = 5`.
- `send_all_delay = 0.4`, `send_all_order_mode = melast`.
- `enable_same_zone_teleport = true`, `stop_autorun_before_warp = true`, `target_npc = true`, `simulate_client_lock = false`, `use_tabs_at_survival_guides = false`, `chat_log_use = log`.
- No custom `<favorites>` or `<shortcuts>` blocks configured in `data/settings.xml`.

## Related Notes
- `FFXI Windower Overview`
- `FFXI Windower Addons Catalog`
