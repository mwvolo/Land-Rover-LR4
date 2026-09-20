# Testing

The README records what's known about this signalset. This file records what
isn't — the open questions that can only be answered in the truck, and the
pass/fail criteria for the 51 debug parameters added on 2026-09-20. Work
through it as drives happen; check items off with evidence, not on suspicion.

---

## Open in-vehicle tasks

- [ ] **Re-run `22F187` against module `726`.** Every other module rejected it with NRC 31, but `726` returned a malformed reply carrying VIN bytes instead. Terminal sequence: `ATSH 726`, `ATCRA 72E`, `22F187`.
- [ ] **Confirm which signalset version the Pelican app is actually holding.** Eight commands stopped being polled on 2026-08-30 and engine speed has never been recorded; the app may be running a cached schedule. Force a signalset refresh, then check whether `F40C` starts appearing.
- [ ] **A drive with a ride-height change.** `3B00`, `3B01`, `3B08` and `2B12` have only ever been sampled at one height, so their "static" classification is meaningless. Raise and lower the suspension during a logged drive.
- [ ] **A drive with a terrain-mode change.** `3B4D`, labelled Drive Mode, has returned `0` on all 171 samples ever taken. Cycle through terrain response settings to find out whether the label or the byte offset is wrong.
- [ ] **A rear-differential lock cycle.** `1E88` and `1E89` on module `795` have never moved.
- [ ] **A cold start.** Needed for the warm-up curves on `113F` and `2104`, and to catch the crank dip on `1153`/`1154`.
- [ ] **Identify module `792`.** Its whole `2A3x` block is static and nothing is known about it. Try `22F18C` and `22F191` against it the way the other six unmined modules were probed on 2026-09-19.
- [ ] **Mine the four still-unidentified modules** — `716`, `726`, `737`, `797`. Each returned a distinct ECU serial via `22F18C` but rejected `22F191`, so they are real and separate but their function is unknown.

---

## Terminal cheat sheet

The adapter terminal puts you on the CAN bus as a diagnostic tester. Everything
you type is either **a setting that decides who you are talking to and who you
are listening to**, or **a question you ask**. AT commands are settings, handled
by the adapter itself and never sent to the car. Anything else is a question,
put on the bus verbatim.

### The pattern you keep repeating

```
ATSH 726      set header      - address requests to this module
ATCRA 72E     receive filter  - show only this module's replies
22F187        question
22F18C        question
22F191        question
```

`ATSH hhh` is "Set Header": the 11-bit CAN ID your requests go out with, which
is the module's request address. `ATCRA hhh` is "set CAN Receive Address" — a
filter on what comes back. On this truck every reply address is the request
address plus 8, so `726` is answered by `72E`, `7E0` by `7E8`, `7D3` by `7DB`.

You need both because they do different jobs. Without `ATCRA` you see every
module that felt like answering, and the adapter sits waiting for more replies
before it gives you the prompt back — slow and noisy. With it, the adapter
returns the moment your module answers.

The block repeats because **each pair retargets the conversation**. Ask the same
three questions of `726`, then re-point at `716` and ask again, and so on. The
questions stay the same; only the two settings change.

### The command at the end

`ATAR` is "Automatic Receive" — it undoes `ATCRA` and lets the adapter choose
the receive address again. **Always finish with it.** Leave a filter set and the
next thing you try is still listening to the last module you probed, so a
perfectly healthy request looks dead. `ATCRA` with no argument resets the
filters the same way.

### Asking a question

| You type | Meaning |
|---|---|
| `22 xxxx` | UDS ReadDataByIdentifier — service `22`, then a four-digit DID. This is the one to hunt with. It is read-only. |
| `01 xx` | Legacy OBD-II mode 1 PID. Mostly useless here — the app consumes these internally and they never become recorded signals. |
| `09 02` | VIN. |
| `22F18C` / `22F191` / `22F187` | ECU serial / part number / spare part number. Ask these first of any new module: cheap, and often enough to identify what it is. |

Stay on service `22`. Services like `10 03` (extended session), `2F` (I/O
control) and `31` (routine control) change ECU state rather than read it.

### Reading the reply

A positive answer. This is a real frame from the logs, the reply to `22F405`,
which arrives as `7E80462F40582` with spaces suppressed:

| Part | Meaning |
|---|---|
| `7E8` | who answered |
| `04` | how many bytes follow |
| `62` | `0x22 + 0x40` — "yes, ReadDataByIdentifier" |
| `F405` | echo of the DID you asked for |
| `82` | the payload, `0x82` = 130, so 130 − 40 = 90 °C coolant |

A rejection — `7E8 03 7F 22 31`: `7F` means refused, `22` is the service you
asked for, `31` is the reason — request out of range, i.e. this module has no
such DID. **This is the normal answer when hunting** and not a problem.

`NO DATA` means nothing replied before the timeout: either the module is not
there or the header is wrong.

Long answers arrive split. Module `726` answering `22F18C` gives
`72E101362F18C353232`, then `72E2134323933383038`, then `72E2200000000000000`.
After the address, `10` marks the first frame and `13` is the total byte count
(19); `21` and `22` are continuation frames counting up. Drop the leading byte
of each continuation and join the rest — here that spells the ECU serial
`5224293808`.

### Watching instead of asking

`ATMA` is "Monitor All" — it sends nothing and dumps every frame on the bus.
Good for finding data that modules broadcast without being asked. Reset the
filter with `ATCRA` first or you will only see one ID. Any key stops it.

### The setup line the app sends

You do not normally type these, but they explain the noise at the top of a log.

| Command | Meaning |
|---|---|
| `ATWS` | warm start — reset the adapter without unplugging it |
| `ATE0` | echo off |
| `ATH1` | headers on — **essential**, without it you cannot tell which module replied |
| `ATS0` | drop spaces from the output |
| `ATSP6` | protocol 6: ISO 15765-4 CAN, 11-bit, 500 kbaud |
| `ATAT1` | adaptive timing |
| `ATM0` | memory off |
| `ATFCSM0` | flow control fully automatic |
| `ATDPN` | which protocol is in use — answers `6` here |
| `ATRV` | battery voltage at the socket |

### Hunting recipe

1. Point at the module: `ATSH <hdr>`, `ATCRA <hdr+8>`.
2. Ask `22F18C`, `22F191`, `22F187` first. A serial proves the module exists; a
   part number often names it.
3. Sweep DIDs. Remember the per-ECU offsets — engine proprietary DIDs sit
   +0x300 from base, transmission and rear differential +0x1000 — so an "out of
   range" is often the right DID at the wrong offset.
4. Note the payload width of anything that answers `62`.
5. `ATAR` when you move on.
6. Ask the same DID again under different conditions. A value that never moves
   tells you nothing, and most of this signalset's unknowns are unknown
   precisely because they were only ever read from a parked truck.

---

## Debug parameter criteria

A DID that moves on a real drive earns a proper decode and a real name. One
that stays flat through a full drive cycle — including a ride-height change
and a terrain-mode switch — can be deleted with evidence rather than on
suspicion. The `freq` 600 tier is the least trustworthy classification in the
table below: every sample behind it came from a parked truck, so "never
moved" so far only means "never moved while parked."

| Module | DID | `freq` | What we think it is | What would count as a result |
|---|---|---|---|---|
| `726` | `0202` | 60 | Only DID found on module 726 | Whether it ever leaves 00 |
| `761` | `197C` | 60 | Unknown, 9 distinct values in 14 samples | Most variable unmined DID; any correlation with a cabin or body state |
| `761` | `D11C` | 60 | Unknown, 3A-41 (58-65) | Plausible temperature; compare against F446 ambient |
| `792` | `2A32` | 600 | Undecoded, constant 00062CF0 in 16 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A33` | 600 | Undecoded, constant 00033E0A in 15 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A34` | 600 | Undecoded, constant 000008C0 in 15 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A35` | 600 | Undecoded, constant 00051E28 in 11 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A37` | 600 | Undecoded, constant 000069 in 12 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A38` | 600 | Undecoded, constant 00001A in 12 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A39` | 600 | Undecoded, constant 000002 in 12 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A3A` | 600 | Undecoded, constant 00001F in 14 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A3B` | 600 | Undecoded, constant 00000000 in 7 samples (all parked) | Drop if still flat after a full drive cycle |
| `792` | `2A3C` | 600 | Undecoded, constant 000000 in 7 samples (all parked) | Drop if still flat after a full drive cycle |
| `795` | `1E88` | 600 | Rear diff, always 0000 | Any movement under lock engagement |
| `795` | `1E89` | 600 | Rear diff, always 09C4 (2500) | Looks like a constant or a limit; drop if flat after a lock cycle |
| `7D3` | `3B00` | 600 | Suspension, static at one ride height | Must be sampled across a ride-height change to mean anything |
| `7D3` | `3B01` | 600 | Suspension, static at one ride height | Must be sampled across a ride-height change to mean anything |
| `7D3` | `3B08` | 600 | Suspension, static at one ride height | Must be sampled across a ride-height change to mean anything |
| `7E0` | `1044` | 600 | Undecoded, constant 00 in 2 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `10E0` | 600 | Undecoded, constant 00 in 14 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `112C` | 60 | Unknown, 570-1341, one FFFF | Any correlation with rpm, load or MAF over a drive |
| `7E0` | `1139` | 60 | Unknown 32-bit, 1142-2784 | Does not track rpm at idle; look for correlation with injector or air mass |
| `7E0` | `113F` | 30 | Temperature, 77-80 observed | Should follow a warm-up curve; compare against F405 coolant and 03F3 oil |
| `7E0` | `1151` | 120 | Undecoded, two values seen: 0000, 0001 | Whether a third value appears on a drive |
| `7E0` | `1152` | 600 | Undecoded, constant 00 in 12 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `1153` | 5 | Charging voltage, 1/256 V | Tracks alternator output; must dip on crank and NOT mirror F442 |
| `7E0` | `1154` | 5 | Charging voltage setpoint, 1/256 V | Should sit pinned at 0E00 (14.00 V) and step only under load shedding |
| `7E0` | `1155` | 600 | Undecoded, constant 00 in 12 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `1156` | 120 | Undecoded, two values seen: 05, 52 | Whether a third value appears on a drive |
| `7E0` | `1158` | 600 | Undecoded, constant 05 in 12 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `1159` | 120 | Undecoded, two values seen: 04, 07 | Whether a third value appears on a drive |
| `7E0` | `115B` | 120 | Undecoded, two values seen: 04, 05 | Whether a third value appears on a drive |
| `7E0` | `1160` | 600 | Undecoded, constant 00D9 in 13 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `1164` | 120 | Undecoded, two values seen: AB, AF | Whether a third value appears on a drive |
| `7E0` | `1187` | 600 | Undecoded, constant 88 in 5 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `11BA` | 600 | Undecoded, constant 012A in 14 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `11BB` | 600 | Undecoded, constant 0000 in 9 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `11BD` | 600 | Undecoded, constant 00EB in 17 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `11BE` | 600 | Undecoded, constant 0000 in 17 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `11C4` | 60 | Unknown enum, seen 00/01/03 | Which driving states produce 03 |
| `7E0` | `11C5` | 120 | Undecoded, two values seen: 04, 14 | Whether a third value appears on a drive |
| `7E0` | `11CC` | 600 | Undecoded, constant 00 in 5 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `F408` | 30 | Short term fuel trim bank 2 (PID 08) | Answers at all; then compare against F406 for the bank asymmetry |
| `7E0` | `F409` | 30 | Long term fuel trim bank 2 (PID 09) | Answers at all; pairs with F407 to measure asymmetry directly |
| `7E0` | `F41F` | 30 | Run time since start (PID 1F) | Records as a signal - this is the alias-mechanism test against standard 011F |
| `7E1` | `0302` | 120 | Counter, +145 over six months | Monotonic across drives - candidate operating hours |
| `7E1` | `0303` | 600 | Undecoded, constant F8 in 6 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `0304` | 600 | Undecoded, constant 02 in 6 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `0305` | 600 | Undecoded, constant 00 in 6 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `0805` | 120 | Undecoded, two values seen: 0A, 0B | Whether a third value appears on a drive |
| `7E1` | `101A` | 60 | Signed value, -14 to +35 | Sign flips with drive/overrun; candidate torque or slip |
| `7E1` | `1E6A` | 600 | Undecoded, constant 00 in 27 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `2104` | 30 | Temperature, 56-64 observed | Should track 1E69 gearbox temp with an offset, or diverge as a second sensor |
| `7E1` | `210F` | 120 | Enum, 01/02 | Changes with gear range or drive mode |
| `7E1` | `2B1B` | 600 | Undecoded, constant 0000 in 13 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `2B1C` | 600 | Undecoded, constant 0000 in 13 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `2B1D` | 600 | Undecoded, constant 0000 in 13 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `2B1E` | 600 | Undecoded, constant 0000 in 13 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E1` | `DD01` | 600 | Undecoded, constant 025166 in 1 samples (all parked) | Drop if still flat after a full drive cycle |

---

## How to check results

Sidecar writes zipped SQLite backups to
`~/Library/Mobile Documents/iCloud~com~featherless~apps~electricsidecar/DataBackups/`
on macOS. `scanSessions` holds every raw request and response; `signals` holds
decoded values. Backups lag live data by up to a day, so check the day after a
drive, not immediately after.

The useful check after a drive, per DID: how many distinct payloads came back,
and over what range. A DID that returned the same byte string every time is
still flat. A DID that returned two or three distinct values is worth a second
drive before deciding anything. A DID that swept a range correlated with
something else on the bus (RPM, load, ride height, gear) is ready for a real
decode.

---

## Known-unresolved

Open questions that aren't in-vehicle tasks — they need log analysis or a
decision, not a drive:

- Why eight commands stopped being polled on 2026-08-30, and whether it's tied to the `freq` re-tiering in `f782e4d`/`527a47f`.
- Whether `F408`/`F409` (bank-2 fuel trims) exist on this engine at all — they have never been requested.
- What `761/197C` and `761/D11C` measure — the most variable unmined DIDs found so far.
