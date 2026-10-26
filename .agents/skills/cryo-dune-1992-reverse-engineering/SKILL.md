---
name: cryo-dune-1992-reverse-engineering
description: Reverse-engineering methodology and recovered facts for Cryo Interactive's Dune (1992) across its DOS floppy, DOS CD v3.7, Amiga, and Sega CD releases. Use when inspecting Cryo Dune binaries, data formats, clock, saves, campaign rules, or porting behavior to the Desert Frost ScummVM engine.
metadata:
  version: 1
---

# Cryo Dune (1992) reverse engineering

This skill distills the work behind Desert Frost Engine: a native ScummVM
engine for Cryo Interactive's *Dune* (1992) that runs one build against all four
releases. Keep observations, inferences, and unknowns separate, and tie claims to
the executable offsets or save observations that support them.

## Scope and sources

Game variants:

- DOS floppy: loose `*.HSQ` files plus `DUNEPRG.EXE`.
- DOS CD v2.1 / v2.3 / v3.7: `DUNE.DAT` archive plus `DNCDPRG.EXE`.
- Amiga: three disk images converted on load into the DOS layouts.
- Sega CD / Mega CD: own host layout and initial game data.

Primary sources, in descending weight:

1. madmoose's `dune-chani` annotation database for `DNCDPRG.EXE` v3.7:
   routine and data names, struct layouts, comments.
   <https://github.com/madmoose/dune-chani>
2. OpenRakis' annotated `DNCDPRG_RECENT.ASM`, read at the offsets named by the
   database; OpenRakis labels seg000 offset `x` as `sub_1x` / `loc_1x`, and
   data names as `0x1F4B0 + ds offset`.
3. The initial data segment of the player's `DNCDPRG.EXE` (file offset 63152).
4. Lionel Debroux' `odrade` and the cited dune2k forum threads for
   troop/NPC/smuggler table offsets.

Addresses in this skill are seg000 code offsets or ds data offsets of CD v3.7
unless stated otherwise. "Verified" means read in disassembly. "Annotated"
means taken from database comments without re-reading every instruction.

## General workflow

1. Identify the exact executable/data release and version string first. CD
   variants differ in clock details, route tables, and save compression.
2. Separate code from initial data: the data segment begins at file offset
   63152 in `DNCDPRG.EXE`.
3. Use the annotation database for names and struct layouts, then confirm key
   routines in disassembly rather than trusting every comment.
4. Keep release-specific behavior explicit: Amiga data is converted to DOS
   layouts on load; Sega CD remains a separate host.
5. Validate recovered rules with engine-assisted checks and, where possible,
   compare against original memory or save files.
6. Record unknowns instead of forcing a complete static model.

## Recovered clock

The PIT is reprogrammed to divisor `0x1745 = 5957`:
`1193182 / 5957 = 200.3 Hz` (seg000:e85c; the divisor is the offset of
`unk_20BF5`).

The PIT handler at seg000:ef6a:

- decrements countdown `ds:46db` each tick;
- on underflow reloads it from `ds:146e` (= 12000 in the data segment);
- increments `game_time` (`ds:2`);
- raises `new_hour_flag` (`ds:46dd`).

One time period is `12000 / 200.3 = 59.9 s` of real time. The clock stops
while `game_suspend_count` (`ds:2788`) is nonzero (dialogues, menus, load).

Day/period split: `day = time >> 4`, `period = time & 15`
(seg000:1ac5/1ae0). A day has 16 periods of 90 in-game minutes.

`run_events_for_current_time_period` (seg000:1b23) runs once per period:
date indicator, sky palette, new-day hook, troop occupation events
(`run_troop_occupation_events`, seg000:6c6f), vegetation walk, and the per-period
action table at seg000:1db3 (period 3 spice shipment counter, period 4
Harkonnen raid scheduler, period 8 shipment reminder, period 15 Harkonnen
growth). WAIT FOR EVENING / WAIT FOR MORNING advance to a target time while
running every skipped period's events.

Reported save-time limits:

- Day 1 00:00 is the minimum.
- Day 1 04:30 is zero/default time.
- Day 1 09:00 is gameplay start.
- WAIT FOR MORNING targets 04:30.
- WAIT FOR EVENING targets 22:30.

Display limits: ordinary date indicator cycles after Day 365 22:30; results
display reaches Day 999 22:30; the 16-bit time value's last pre-wrap evening is
Day 4096 22:30. The CD date routine at seg000:1ad1 adds three with 16-bit
arithmetic before shifting by four; seg000:1a6b applies the 365-day display
cycle and adds one.

For a desired day and 90-minute slot counted from midnight, encode the time
word as:

```python
def encode_time(day, slot_from_midnight):
    assert 1 <= day <= 4096
    assert 0 <= slot_from_midnight < 16
    time_word = (day * 16 + slot_from_midnight - 19) & 0xffff
    return time_word.to_bytes(2, "little")
```

Important state-layout correction: the word immediately before `game_time` is
`rand_bits` at `ds:0000`; `game_time` is at `ds:0002`. Preserve `rand_bits`
when only editing the date. The real timer countdown is the separate `ds:46db`.

## Save format notes

- DOS CD saves use compressed signatures and run-length encoding. Decode the RLE
  body before locating fields; `F7 count` tokens are not fixed-position
  separators.
- `rand_bits` / `game_time` word order in file observations is little-endian.
- Do not apply DOS compressed signatures or time encoding to native Amiga or
  Sega CD files.
- Save observations are not independently verified executable behavior. Keep
  that distinction in reports.

## Data and assets

- `DUNE.DAT` archive, loose floppy files, and HSQ compression.
- `.SAL` room descriptions; 4-bit/8-bit sprite sheets.
- Sky/sand colours by time of day and day of the week.
- Fonts, pointer, intro videos in executable-defined order.
- Rooms, place tables, exits and characters read from the executable.
- Amiga M1/M2/M3 tracker music through Paula; DOS HERAD AdLib music through
  OPL. Amiga files, sheets, rooms, palettes, copper sky and data segment are
  converted to DOS layouts on load.
- Sega CD has its own host: disc index, text, tile screens, initial game data,
  room screens, panel, conversations with portraits, and map with travel.

## Recovered mechanics (selected)

- Places: names, types, and room behaviour recovered from `placeData[]` /
  `roomsArray[]`.
- Troop hiring and rallying follow executable rules; contact range and map menu
  interactions have verified routines.
- Occupations run each period through `run_troop_occupation_events`.
- Spice shipments and prospecting use the verified per-period action table.
- Ecology route includes wind traps, bulbs, irrigation, vegetation spread,
  fortress capture, and modify equipment.
- Scripted scenes and story progression include Jessica's lessons, visions,
  Comm room bargaining, Stilgar, and sietch chief lines.
- Results screen and smuggler/battle structures are documented but some battle
  details remain incomplete.

Unknown areas remain: complete battle algorithms, some release-specific route
table saturation behaviour, and Sega CD video/sound/flight/battle porting.

## Reusable skill checklist for similar old games

When applying this work to another classic game:

1. Pin the exact binary version and collection/archive format.
2. Capture original disassembly annotations and data-segment offsets.
3. Recover timing from timer hardware, not just observed wall-clock behaviour.
4. Separate display limits from underlying variable limits.
5. Record both verified executable facts and observed save-file facts.
6. Preserve released data invariants when porting (Amiga/Sega CD conversions).
7. Validate end-to-end with assisted playthroughs and regression checks.
8. Leave explicit unknown lists so later passes can close them.

See [references/findings.md](references/findings.md) for compact facts and
source pointers.
