# PallyPowerTW

Paladin blessing, aura, and seal management addon for WoW 1.12 / TurtleWoW. Coordinates assignments across multiple Paladins in a raid, tracks buff status per class, and provides one-click Greater/regular blessing casting.

## Screenshots

<img src="https://raw.githubusercontent.com/ivanovlk/PallyPowerTW/refs/heads/master/ScreenShot.png" float="left" align="left" width="100%">

*Main assignment UI*

<img src="https://raw.githubusercontent.com/ivanovlk/PallyPowerTW/refs/heads/master/ScreenShotAura.png">

*BuffBar horizontal layout with hidden default Aura frame*

## Installation

Download the zip file and rename to PallyPowerTW, or use the Turtle WoW Launcher.

## Usage

### BuffBar (Casting)
- **Left-click**: Casts a Greater Blessing (30 min) on the class.
- **Smart Rebuff**: When enabled (default), Left-clicking when only 1 player in the class is missing the buff automatically casts a normal 10-minute blessing on that player instead, saving Symbol of Kings and mana.
- **Right-click**: Casts a normal (10 min) blessing. If individual player blessings are assigned, right-click applies those first.
- **Pre-Cast Validation**: Checks for required reagents (Symbol of Kings) and mana before attempting to cast, alerting you with a clear message and sound if reagents or mana are insufficient.

### Assignment Grid
- **Left-click** on a class cell: Cycle through blessings for that Paladin. Never gets stuck even when duplicate blessings exist in the raid.
- **Shift + Left-click**: Assigns the blessing to ALL classes for that Paladin.
- **Right-click** on a class cell: Clears the assignment.
- **Mouse Wheel**: Cycle blessings forwards/backwards.
- **Left-click class header**: Expands the individual player roster for that class.
- **Right-click individual player**: Assigns an individual 10-minute blessing override to that specific player (e.g. assign Wisdom to a healer Shaman while melee Shamans get Might).
- **Middle-click player name**: Toggles **Tank** status. Tanks are automatically exempted from Greater Blessing of Salvation (BuffBar stays green without casting Salv on tanks).
- **"S" Column**: Assigns Seals / Judgement duties for each Paladin (Crusader, Wisdom, Light, Justice) to coordinate Judgements on bosses.
- **"A" Column**: Assigns Paladin Auras.

## Slash Commands

| Command | Description |
|---------|-------------|
| `/pp` or `/pallypower` | Toggle the main assignment UI frame |
| `/pp compact` | Toggle compact BuffBar mode (hides classes that are 100% buffed) |
| `/pp report` | Print current blessing/aura assignments to raid or party chat |
| `/pp buff` or `/pp autobuff` | Auto-buff all assigned blessings on nearby players |
| `/pp lock` | Toggle locking/unlocking frame positions |
| `/pp debug` | Toggle debug mode (prints diagnostic info to chat) |

## Features

- **Smart Single-Target Rebuff**: Left-clicking a class with only 1 unbuffed member casts a 10-min single blessing, saving Symbol of Kings.
- **Compact BuffBar Mode**: Option to auto-hide classes that are 100% buffed, shrinking the BuffBar and keeping the screen clean (`/pp compact`).
- **Pre-Cast Reagent & Mana Validation**: Clear audio-visual warning when missing Symbol of Kings or insufficient mana.
- **Clean Talent Display**: Talents are formatted clearly as `Rank (+Talent)` (e.g. `1 (+5)`) instead of `1+5`.
- **Automatic Roster Refresh**: Cleanly detects when a Paladin leaves the group and refreshes assignments without ghost locks.
- **Robust Single-Click Cycle**: Fallback cycle pass ensures grid clicks never get stuck even when duplicate buffs or smartbuff restrictions are active.
- **Tank Exemption (Middle-Click)**: Middle-click a player's name in the grid to exempt them from Salvation.
- **Seal & Judgement Coordination ("S" Column)**: Assign and track Paladin Seals / Judgement duties.
- **Individual Blessing Overrides**: Assign custom 10-minute blessings to specific players within a class.
- **Warrior & Pet Protection**: Prevents conflicting Greater Blessing casts between Warriors and Hunter pets.
- **Line-of-Sight Check**: Integrated LoS checks via UnitXP_SP3 (if loaded).
- **Save Presets**: Save and swap blessing and aura profiles via the Minimap icon.

## Known Issues

- "[Patch FR] Turtle WoW en francais + Pack de voix francaise corrige pour VoiceOver" is not supported. PallyPowerTW does not work correctly when this mod is installed.

## Notes

- Hunter pets and Warriors share the same class ID in vanilla WoW, so Greater Blessings affect both (not a bug).

## Additional Resources

[PallyPowerTW Addon Wiki](https://github.com/ivanovlk/PallyPowerTW/wiki/PallyPowerTW-Addon-Wiki)

## Changelog
- 06.09.26 - **Major Quality of Life & Bugfix Update**:
  - **Fixed Single-Click Cycle Bug**: Added fallback pass so left-click cycling never gets stuck at -1 or locked out when duplicate blessings exist.
  - **Pre-Cast Reagent & Mana Checks**: Added pre-cast checks for Symbol of Kings and mana with `PlaySound("igQuestFailed")` and localized screen errors.
  - **Smart Single-Target Rebuff**: Left-clicking a class where only 1 member needs a buff automatically casts a 10-min small blessing to save Symbol of Kings and mana.
  - **Compact BuffBar Mode**: Added `/pp compact` and Options checkbox to collapse fully buffed classes from the BuffBar.
  - **Talent Display Formatting**: Ranks and talents now format cleanly as `Rank (+Talent)` (e.g. `1 (+5)`).
  - **Auto-Refresh on Leave**: Enhanced `PallyPower_PaladinLeftGroup` to scan `AllPallys` so leaving Paladins without assignments are immediately detected and cleared.
- 09.04.26 - Add paladin selector to Assignment Grid top-left area. This way you can display specific paladin's individual blessings assignment (fix for feature request #25) 
- 01.03.26 - Event-driven scanning replaces timer-based polling (zero CPU cost when idle). TurtleWoW realm detection for correct blessing durations (also vanilla 1.12.1 compatible). New `/pp lock` slash command for toggling frame positions. Scan frequency repurposed as UI debounce interval (clamped to max 2s). Removed obsolete "Poll Per Frame" setting. Slash commands documented in README.
- 25.08.25 - If Salvation is assigned, user is tank, and no individual blessings, do not count against need (so the BuffBar button stays green even with tank missing Salvation).
- 25.08.25 - Assign/clear raid icon when player is marked as tank if we are Raid Leader/Assist or party leader.
- 22.08.25 - Allow assignments of seals for each paladin.
- 22.08.25 - Mark as tank reflects to pfUI tank assignment (if available). Don't allow GB on pets if Warriors assignment differs from pets assignment. If same assignment, mark both as GB-blessed.
- 09.08.25 - Warriors and hunter pets share same class so if they have same blessing assigned and you cast Greater Blessing, PP marks both Warriors and pets as blessed in BuffBar.
- 15.07.25 - Fix: When casting Greater Blessings and several targets are out of range, addon assumes they got the buff and does not allow re-cast of GB. Now those targets are correctly marked as needing blessing and allow re-cast of GB.
- 15.07.25 - Aura assignment is also saved in presets.
- 15.07.25 - Allow marking a player as a tank and sync with other paladins.

