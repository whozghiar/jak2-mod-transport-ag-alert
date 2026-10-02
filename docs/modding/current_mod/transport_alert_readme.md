# Crimson Guard Alert Drop-Ship — Mod Readme

> - **Repository:** [`whozghiar/jak2-mod-transport-ag-alert`](https://github.com/whozghiar/jak2-mod-transport-ag-alert)
> - **Game:** Jak II (OpenGOAL)
> - **Status:** Working — scripted troop transport tied to the city alert level, ~1 per minute

---

## 1. Overview & Objective

While Haven City is on **alert (level ≥ 1)**, a **Crimson Guard Troop Transport** (`transport-ag`, the retail drop-ship) descends near the player roughly **once per minute**, deploys a squad of Crimson Guards, and leaves. It is a *scripted* actor — you cannot board it, and it is not part of the ambient traffic pool.

This is the sibling of the [`whozghiar/jak2-mod-transport-ag-traffic`](https://github.com/whozghiar/jak2-mod-transport-ag-traffic) mod (the drivable, traffic-integrated gunship). They share only the `.fr3` merc-geometry injection that makes the transport hull renderable in free-roam.

## 2. How it works

| Piece | File | What it does |
|---|---|---|
| `update-alert-transport` | `traffic-manager.gc` | New `traffic-manager` method, called every frame from `active:post`. Polls the alert level; when it's ≥ 1 and the 1-per-minute cooldown has elapsed and no transport is currently alive, it `process-spawn`s a `transport` at a random point 10–18 m from the player. |
| `alert-transport` / `alert-transport-next-check` | `traffic-manager.gc` | New fields: a handle to the live drop-ship and the earliest `current-time` the next one may spawn. Initialised in `reset-and-init` and `traffic-manager-init-by-other`. |
| scripted `transport` | `transport.gc` (retail) | Unmodified retail drop-ship logic — `come-down` → `idle` (hatch open, `transport-method-33` guard drops, `max-guard` 8) → `leave`. One gated call added: `(mod-alert-transport-art-hack)` (defined in `traffic-manager.gc`) in `transport-init-by-other` binds the runtime-spawned transport (which has no `entity-actor`) to the active lwide level, where its art is baked, and publishes ctywide's `vehicle-turret` art-group there, so its skeleton and chin turret resolve in a retail boot. |
| `.fr3` merc injection | `jak2_config.jsonc`, `lwide{a,b,c}.gd` | `extra_art_groups_by_dgo` bakes `transport-ag`'s merc geometry (textures resolved via `LPROTECT`'s remap table) into the always-resident `lwidea/lwideb/lwidec.fr3`. Without this the PC merc renderer silently skips the hull (`num_missing_models++`). See [Lisp wiki §3.5](../../../.agents/skills/goal-lisp/wiki/jak2.md#35-merc-geometry--fr3-residency). |

**Cooldown / rate:** the `alert-transport-next-check` timer is set to `current-time + 60 s` at the moment a transport spawns, and case 1 of the `cond` blocks any new spawn while one is still alive — so the effective rate never exceeds **one drop-ship per minute**.

## 2b. Runtime toggle (mandatory procedure)

Per AGENTS.md's golden rules the mod ships **OFF** and is switched from the in-game
Mods menu (**L3 + SELECT**, retail boot): `Mods ▸ transport-ag-alert ▸ Enable`.

| Piece | File | Role |
|---|---|---|
| `*mod-transport-ag-alert-enable*` | `traffic-manager.gc` | `define-perm` (symbol, `#f`). Non-debug, CWI-resident so the gated code links in release builds; the `#t` survives level reloads. |
| gate in `active:post` | `traffic-manager.gc` | `(when *mod-transport-ag-alert-enable* (update-alert-transport self))` — OFF = stock `:post`. |
| gate in `transport-init-by-other` | `transport.gc` | `(mod-alert-transport-art-hack)` wrapped in `(when *mod-transport-ag-alert-enable* ...)` so retail story transports are byte-for-byte stock. |
| `mod-transport-ag-alert-build-menu` + register | `pc/features/transport-ag-alert-menu.gc` (new, retail-safe, in GAME) | Registers the single "Enable" flag with `mods-menu-register`. The toggle flips the flag and sends `'mod-alert-transport-reset` to `*traffic-manager*` (a drop-ship still in the air leaves; no traffic recycle). Wired in `game.gd` after `mods-menu.o`. |

The `traffic-manager` deftype fields, the `update-alert-transport` method and its
init-site assignments are always compiled but are inert dead weight while the
flag is `#f` (the method is never called).

## 3. Rendering — the merc geometry `.fr3` injection

`transport-ag`'s hull geometry only ever shipped in `lprotect/ctykora/forestb/nest`, none of which are resident while free-roaming Haven City. The fix (in `decompiler/config/jak2/jak2_config.jsonc`):

```jsonc
"extra_art_groups_by_dgo": {
  "LWIDEA.DGO": ["transport-ag:LPROTECT.DGO"],
  "LWIDEB.DGO": ["transport-ag:LPROTECT.DGO"],
  "LWIDEC.DGO": ["transport-ag:LPROTECT.DGO"]
}
```

This bakes the geometry into `lwidea.fr3 / lwideb.fr3 / lwidec.fr3` (always borrowed into `ctywide` slot 1 in free-roam). The matching `transport-ag.go` + `tpage-2869.go` entries are added to `lwidea.gd / lwideb.gd / lwidec.gd`. No runtime level-borrow.

> **Requires a re-extraction** (`task extract`) so the three `.fr3` are rebuilt with the injected geometry.

## 4. How to Test

1. **Extract (once):** `task extract` — rebuilds `lwide*.fr3` with `transport-ag`.
2. **Rebuild:** `task repl` then `(mi)` (the `traffic-manager` deftype changed — restart the REPL if `(mi)` complains).
3. **Launch:** `task boot-game`, enter Haven City free-roam.
4. **Enable:** open the Mods menu with **L3 + SELECT** → `Mods ▸ transport-ag-alert ▸ Enable` (OFF by default).
5. **Trigger:** aggro a Crimson Guard to raise the alert level.
6. **Observe:** within a few seconds a transport descends 10–18 m from you, its hatch opens, ~8 guards drop out, and it flies off. The console prints `AT: transport drop spawned ...`.
7. **Rate:** stay on alert — the next transport should not appear until ~60 s after the previous one spawned.

**Key source files:**

- `goal_src/jak2/levels/city/traffic/traffic-manager.gc` — `update-alert-transport` + fields + call site + `*mod-transport-ag-alert-enable*` gate + `mod-alert-transport-art-hack` + `'mod-alert-transport-reset` event.
- `goal_src/jak2/levels/city/traffic/vehicle/transport.gc` — gated `(mod-alert-transport-art-hack)` call.
- `goal_src/jak2/pc/features/transport-ag-alert-menu.gc` — Mods menu (L3 + SELECT) toggle registration.
- `goal_src/jak2/dgos/game.gd` — `"transport-ag-alert-menu.o"` after `mods-menu.o`.
- `decompiler/config/jak2/jak2_config.jsonc`, `goal_src/jak2/dgos/lwide{a,b,c}.gd` — `.fr3` merc injection.

## 5. Current State & Known Tradeoffs

- **Working:** spawn / descent / guard drop / departure / 1-per-minute rate.
- **Not interactive:** the drop-ship cannot be boarded or shot down like a real vehicle — it is the retail scripted actor. For a drivable transport, use [`whozghiar/jak2-mod-transport-ag-traffic`](https://github.com/whozghiar/jak2-mod-transport-ag-traffic).
- **Hull invisible until re-extract:** if `lwide*.fr3` have not been rebuilt with the injection, the transport still spawns and drops guards but has no hull mesh (its `vehicle-turret` chin gun still shows). Not a crash.
- **`jak2_config.jsonc`** also carries a local `rip_levels: true` and a whitespace reformat inherited from the combined branch — harmless, not part of this feature.

---

## 6. Modding Changes Log

| Date | Touched/Created Files | Technical Description | Objective |
| :--- | :--- | :--- | :--- |
| 2026-08-29 → 09-01 | `traffic-manager.gc`<br>`transport.gc`<br>`jak2_config.jsonc`<br>`lwide{a,b,c}.gd` | Ported from the combined `jak2/features/transport_v2` branch: `update-alert-transport` + `alert-transport`/`alert-transport-next-check` fields tie the retail `transport` drop-ship to the city alert level; `(ctywide-entity-hack)` added to `transport-init-by-other`; `transport-ag` merc geometry baked into `lwide*.fr3` via `extra_art_groups_by_dgo`. | Scripted troop transport during alerts, no runtime level borrow. |
| 2026-09-02 | `traffic-manager.gc`<br>`docs/modding/current_mod/transport_alert_readme.md` | **Branch split.** Isolated the alert drop-ship onto its own branch (the drivable traffic gunship moved to `jak2/features/transport_traffic`). Post-spawn cooldown `(seconds 15)` → `(seconds 60)` so the rate is strictly one drop-ship per minute. Removed the `transport-v` traffic-type spawn wiring (`traffic-object-spawn` / `type-from-vehicle-type` cases, `want-count[20]` back to 0). New scoped readme. | One clean feature per branch; enforce the requested 1/minute rate. |
| 2026-09-08 | `traffic-manager.gc`<br>`transport.gc`<br>`pc/debug/transport-ag-alert-menu.gc` *(new)*<br>`dgos/game.gd`<br>`README.md` | **Branch renamed** `transport_alert` → `transport-ag/alert`. **Mandatory Debug ▸ Mods toggle:** `define-perm *mod-transport-ag-alert-enable*` (`#f`), `update-alert-transport` call and the added `(ctywide-entity-hack)` both gated behind it, new `transport-ag-alert-menu.gc` registering the "Enable" flag via `mods-menu-register`, wired in `game.gd`. Fresh install now plays stock Jak 2. | Native non-regression: mod OFF by default, switchable in-game. |
| 2026-09-16 | `traffic-manager.gc`<br>`pc/features/transport-ag-alert-menu.gc` | **Empty-city regression fix.** `reset-actors` calls level `activate-func`s in `*level*` SLOT ORDER, so `ctywide-activate` (-> `traffic-start` -> `init-params` -> `reset-and-init`) can run AFTER `lwide-activate` and wipe everything it installed: `object-type-info-array[0..19].level` back to `#f` (no traffic spawns at all) and `(reset alert-state)` dropping the `target-jak` flag (guards ignore Jak's crimes) plus lwideb's forced war-zone alert. `init-params` now re-runs `lwide-activate` on the active lwide level, after `restore-default-settings`. Also: the drop-ship spawn no longer logs success when `process-spawn` returns `#f`, and the menu toggle no longer sends `'kill-all` + `'spawn-all` (that churned ~150 `*default-dead-pool*` slots in one frozen frame and starved the drop-ship's own spawn). | Restore Haven City's population and guard alerts after a death / checkpoint restart, without the drop-ship going missing. |
| 2026-09-16 | `traffic-manager.gc`<br>`transport.gc` | **Drop-ship invisible in a retail boot.** `skeleton-group->draw-control` resolves an actor's art-group inside `(-> pp level)` and falls back to `art-group-load-check`, whose body is entirely inside `(when *debug-segment* ...)`. With the toggle in the old Debug menu the mod only ever ran in a `-debug` boot, where that fallback loaded `transport-ag` from disk; on the retail-safe Mods menu it returns `#f`, `initialize-skeleton` does `(go process-drawable-art-error "art-group")` and the art-error state prints nothing — a live process with no draw-control, no skeleton and no turret. `ctywide-entity-hack` replaced by `mod-alert-transport-art-hack`, which binds the drop-ship to the active lwide level (where the mod baked `transport-ag` + `tpage-2869`) and publishes ctywide's `vehicle-turret` art-group there so the chin turret still resolves. The spawn log now prints the resulting state. | Make the hull appear in a retail boot, not only under `-debug`. |
| 2026-10-02 | `README.md`<br>`docs/modding/current_mod/transport_alert_readme.md` | **Docs aligned with the code:** header points to the mod repository; current-behaviour text (§2, §2b, §4 key files) describes `mod-alert-transport-art-hack` instead of `(ctywide-entity-hack)` and the retail-safe `pc/features/transport-ag-alert-menu.gc` toggle (L3 + SELECT, sends `'mod-alert-transport-reset`) instead of `Debug ▸ Mods`; sibling mod named by its repository; dead wiki link fixed; How to Test renumbered; player README video thumbnail URL fixed. | Documentation matches the shipped code. |
