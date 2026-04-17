# Knight Kill Count

A small Crusader Kings III mod that tracks every character's lifetime named
kills and displays the running total, with a kill icon, next to the
vassal-stance icon under their portrait in the character window.

## What is actually counted

CK3 displays per-knight casualty numbers in the post-battle summary's
**Knights** tab, but those numbers are produced by a C++ GUI datafunction
(`RegimentCombatStats.GetTotalKills` / `BattleSummaryWindow.GetKnightKills`)
that is only ever referenced in `game/gui/window_battle_summary.gui`. They
are not exposed to the scripting layer and no GUI-to-script writer exists in
vanilla, so there is no way for a mod to capture those exact numbers. This
is verified by grepping the entire CK3 install and the CK3 wiki (see
[docs/](docs/)) - no `on_action`, scripted trigger, scripted effect or
script value references them.

What this mod counts instead is **named-character kills**: every time a
character dies and the engine sets `scope:killer` on the `on_death`
on_action, we increment a persistent character variable (`kc_kill_count`)
on the killer. That covers:

- Knight-vs-knight and knight-vs-commander kills during combat-phase
  events (`game/common/combat_phase_events/00_knight_phase_events.txt`).
- Commander-vs-commander kills.
- Successful murder schemes.
- Duels that end in a death.
- Any other place where the engine attributes a death to a specific named
  character.

It does **not** count anonymous levy/men-at-arms casualties. Those are
exactly the numbers in the battle-summary Knights tab, and they are
inaccessible to script.

## How it works

```
on_death (ROOT = dying character)
  -> if exists = scope:killer
     -> scope:killer = { kc_increment_kill_count_effect = yes }
        -> set_variable kc_kill_count = 0 (if missing)
        -> change_variable kc_kill_count += 1
           -> rendered in gui/window_character.gui
              [Character.MakeScope.Var('kc_kill_count').GetValue|V0]
```

The widget is added to the `blockoverride "portrait_opinion"` inside
`portrait_character_view_main` in the character window, so it sits in the
same little strip as the vassal-stance icon beneath the main portrait. It
hides itself on characters whose `kc_kill_count` is 0 or unset.

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
    window_character.gui                     # vanilla copy + one blockoverride edit
  localization/
    english/
      kc_knight_killcount_l_english.yml      # tooltip text (UTF-8 with BOM)
```

No new graphics are shipped; the widget uses the vanilla
`gfx/interface/icons/icon_kill.dds`.

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

## Validation

With the mod enabled and `-debug_mode` in the CK3 launch options:

1. Open a freshly-started character's window - no kill-count widget
   should be visible (the variable hasn't been set yet).
2. Select any character and run in the debug console:

   ```
   effect = { kc_increment_kill_count_effect = yes }
   ```

   The widget should appear with "1" next to the kill icon. Running it
   again bumps it to "2". Plain `change_variable = { name = kc_kill_count
   add = 1 }` on a character that has never had the variable set will
   error out with `Variable not of the 'value' scope type. Type: empty`;
   that is why the scripted effect initialises the variable first.
3. Fight a small battle, kill an enemy knight during its combat-phase
   event, and check the surviving knight - `kc_kill_count` should have
   ticked up by 1.
4. Check `Documents\Paradox Interactive\Crusader Kings III\logs\error.log`
   for any entries referencing `kc_knight_killcount_*` or
   `window_character.gui` - there should be none from this mod.

## Compatibility

- **Script side**: no vanilla files are replaced. The mod only *appends*
  to the `on_death` on_action (CK3 merges multiple mods' entries for the
  same on_action) and defines a new scripted effect in its own namespace.
  Save-game compatible: the variable just starts appearing on new kills.
- **GUI side**: `gui/window_character.gui` is a full-file override
  (CK3 does not support partial `.gui` patching). Other mods that also
  override this file - Knight Manager Continued, Better UI Scaling,
  RUI: Character Window, etc. - will conflict and need a manual patch.
  If another mod's override is the one you want to keep, pull its copy
  into `ck3-knight-killcount/gui/window_character.gui` and re-apply the
  `blockoverride "portrait_opinion"` edit on top of it.

## Upgrading after a CK3 patch

If a patch changes `game/gui/window_character.gui`, re-copy it and
re-apply the `blockoverride "portrait_opinion"` edit. The exact patch is
the `hbox` containing `vassal_stance_icon` plus the
`kc_kill_count_display` hbox inside the `portrait_opinion` blockoverride
of `portrait_character_view_main`.
