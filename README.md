# Knight Kill Count

A small Crusader Kings III mod that tracks every character's lifetime named
kills and displays the running total as a compact badge overlaid on the
bottom-right corner of the vanilla **View Kills** icon in the character
window (the icon in the vertical strip to the left of the portrait).

## What this branch does (`display-number`)

![mock](docs/screenshot.png)

Rather than adding a new widget under the portrait (that is what the
`main` branch does), this branch re-uses the existing **View Kills**
button that CK3 already shows for any character with attributed named
kills, and puts the running total directly on it. You now see the count
at a glance without having to open the kill list and count the rows by
hand.

## What is counted

Every time a character dies and the engine sets `scope:killer` on the
`on_death` on_action (`game/common/on_action/death.txt`), this mod
increments a persistent character variable (`kc_kill_count`) on the
killer. That covers every place the vanilla game attributes a death to
a specific named character:

- Knight-vs-knight and knight-vs-commander kills in combat-phase events
  (`game/common/combat_phase_events/00_knight_phase_events.txt`).
- Commander-vs-commander kills.
- Successful murder schemes.
- Duels that end in a death.
- Executions (one increment per executed character).
- Any other `death = { killer = X }` call in vanilla or another mod.

It does **not** count anonymous levy / men-at-arms casualties - those
numbers are shown in the post-battle **Knights** tab but they are
produced by a C++ GUI datafunction
(`RegimentCombatStats.GetTotalKills` / `BattleSummaryWindow.GetKnightKills`)
that is only used inside `game/gui/window_battle_summary.gui` and is
not exposed to the scripting layer, so no mod can capture them.

## Why not use the `every_killed_character` iterator?

An earlier version of this branch computed the badge number from a
script value:

```
kc_total_kills_value = {
    value = 0
    every_killed_character = { add = 1 }
}
```

That *looks* cleaner (you get a count of the full vanilla kill list for
free, including kills that happened before the mod was installed) but
in practice it over-counts on characters that have done any executions
- e.g. 1 battle kill + 2 executions produces a count of 5 instead of 3.
The engine's internal "killed character" list keeps extra bookkeeping
entries for execute_opinion cascades, so `every_killed_character` is
not a reliable 1:1 kill counter. We therefore fall back to explicit
per-death tracking via `on_death`, which is guaranteed to increment
exactly once per attributed death.

The trade-off: the badge only counts kills that happen **after** the
mod is enabled. Characters that already had kills in your save before
installing the mod show no badge until their next kill ticks the
variable to 1.

## How it works

```
on_death (ROOT = dying character)
  -> if exists = scope:killer
     -> scope:killer = { kc_increment_kill_count_effect = yes }
        -> set_variable kc_kill_count = 0 (if missing)
        -> change_variable kc_kill_count += 1
           -> rendered by gui/window_character.gui inside
              button_normal name = "open_kill_list":
              text_single name = "kc_kill_count_badge"
                visible = GreaterThan_CFixedPoint(
                    Character.MakeScope.Var('kc_kill_count').GetValue, 0)
                text    = Character.MakeScope.Var('kc_kill_count').GetValue
```

The badge is `parentanchor = bottom|right` inside the existing
`open_kill_list` button, so it floats in the bottom-right corner of
the icon. It is hidden when the variable is unset or zero, so
characters the mod has never seen a kill for look vanilla.

## Repository layout

```
ck3-knight-killcount.mod                     # launcher descriptor
ck3-knight-killcount/
  descriptor.mod
  common/
    on_action/
      kc_knight_killcount_on_actions.txt     # appends to on_death
    scripted_effects/
      kc_knight_killcount_effects.txt        # kc_increment_kill_count_effect
  gui/
    window_character.gui                     # vanilla copy + badge overlay
  localization/
    english/
      kc_knight_killcount_l_english.yml      # KC_KILL_COUNT_BADGE_TOOLTIP
```

No new graphics are shipped; the badge uses the vanilla
`gfx/interface/component_tiles/solid_black_label.dds` background so it
matches the rest of the UI.

## Installation

1. Clone this repo.
2. Copy (or symlink) both `ck3-knight-killcount.mod` and the
   `ck3-knight-killcount/` folder into your CK3 mod directory:

   ```
   %USERPROFILE%\Documents\Paradox Interactive\Crusader Kings III\mod\
   ```

   The `.mod` file must end up directly under `mod\`, and the folder
   it points to (`path = "mod/ck3-knight-killcount"`) must sit next to
   it.
3. Launch CK3, enable **Knight Kill Count** in the launcher, play.

The mod is flagged as `supported_version="1.18.*"` (CK3 1.18.x).

## Validation

With the mod enabled and `-debug_mode` in the CK3 launch options:

1. Open a freshly-started character's window - no badge on the
   **View Kills** icon (the variable hasn't been set yet, and vanilla
   may already hide the icon outright for characters with no known
   kills).
2. Select any character and run in the debug console:

   ```
   effect = { kc_increment_kill_count_effect = yes }
   ```

   The **View Kills** icon should appear with a small "1" in the
   bottom-right corner. Running the effect again bumps it to "2".
3. Fight a small battle, kill an enemy knight during its combat-phase
   event, and check the surviving knight - their badge should have
   ticked up by 1.
4. Execute a prisoner and check the executioner's badge - exactly +1.
5. Check
   `Documents\Paradox Interactive\Crusader Kings III\logs\error.log`
   for any entries referencing `kc_knight_killcount_*` or
   `window_character.gui` - there should be none from this mod.

## Difference from `main`

- `main` ships a new widget under the portrait (next to the vassal-
  stance icon). Nothing is overlaid on existing UI.
- `display-number` (this branch) does not add a portrait widget. It
  overlays the count on the existing **View Kills** button instead.

Both branches use the exact same on_death / scripted-effect pipeline,
so the number shown is the same - only the rendering differs.

## Compatibility

- **Script side**: no vanilla files are replaced. The mod only
  *appends* to the `on_death` on_action (CK3 merges multiple mods'
  entries for the same on_action) and defines a new scripted effect
  in its own namespace. Save-game compatible: the variable just
  starts appearing on new kills.
- **GUI side**: `gui/window_character.gui` is a full-file override
  (CK3 does not support partial `.gui` patching). Other mods that also
  override this file - Knight Manager Continued, Better UI Scaling,
  RUI: Character Window, etc. - will conflict and need a manual patch.
  If another mod's override is the one you want to keep, pull its
  copy into `ck3-knight-killcount/gui/window_character.gui` and
  re-apply the `text_single name = "kc_kill_count_badge"` block
  inside `button_normal name = "open_kill_list"` on top of it.

## Upgrading after a CK3 patch

If a patch changes `game/gui/window_character.gui`, re-copy it and
re-apply the `text_single` badge edit inside the `open_kill_list`
button. The diff is ~30 lines.
