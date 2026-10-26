# Cryo Dune (1992) compact findings

Source basis:

- madmoose `dune-chani` annotation database, `DNCDPRG.EXE` v3.7.
- OpenRakis `DNCDPRG_RECENT.ASM`.
- Desert Frost Engine ScummVM implementation and notes.
- Observed DOS save files (not verified executable behavior unless noted).

Clock:

- PIT divisor `0x1745 = 5957`, about 200.3 Hz; code at seg000:e85c.
- Countdown `ds:46db`; reload `ds:146e = 12000`; period about 59.9 s.
- `game_time` at `ds:2`; `new_hour_flag` at `ds:46dd`; suspension at `ds:2788`.
- `day = time >> 4`; `period = time & 15`; 16 periods per day.
- Event dispatcher seg000:1b23; occupancy seg000:6c6f; per-period table seg000:1db3.
- Targets: WAIT FOR MORNING 04:30, WAIT FOR EVENING 22:30.

Time word encoding:

```
time_word = (day * 16 + slot_from_midnight - 19) & 0xffff
```

Preceding word is `rand_bits` at `ds:0000`; do not zero it when shifting only time.

Recorded display limits:

- Date indicator cycles after Day 365 22:30.
- Results display reaches Day 999 22:30.
- 16-bit time word last pre-wrap evening Day 4096 22:30.

Files and formats:

- DOS CD: `DUNE.DAT` plus `DNCDPRG.EXE`; DOS floppy: `*.HSQ` plus `DUNEPRG.EXE`.
- `.SAL` rooms; 4/8-bit sprites; HSQ compression.
- Amiga converted on load; Sega CD separate host.

Unknowns:

- Full battle algorithms.
- Some saturation behaviour in release-specific route tables.
- Sega CD video, sound, flight, and battle porting.
