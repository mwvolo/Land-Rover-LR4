# Testing

The README records what's known about this signalset. This file records what
isn't — the open questions that can only be answered in the truck, and the
pass/fail criteria for the debug parameters added on 2026-09-20 (51 in the
first batch, 8 more added later that day from the ECM's own supported-PID
bitmask). Work through it as drives happen; check items off with evidence,
not on suspicion.

---

## Open in-vehicle tasks

- [ ] **Get the suspension module back into the polling rotation.** `7D3` carries no metric-bearing command, and the app converges on polling only what it stores, so it drifts out of the rotation between signalset changes. It was last addressed at 14:07 on 2026-09-20 and the 15:14 off-road drive collected no suspension data at all. Coverage does come back on its own whenever the signalset changes, so merging a PR is the cheap lever: check the first drive after a merge. Do **not** delete and re-add the vehicle — nothing is stuck, and an earlier version of this task wrongly said it was.
- [ ] **Verify the cut-down signalset on the next drive.** The polling budget was cut from 10.53 to 3.922 req/s (85 commands down to 66) because measurement showed the app only ever delivers about 4.2 req/s to signalset traffic, no matter how many commands are in the file. Asking for 10.53 against a 4.2 ceiling meant the app picked which 40% to serve; asking for 3.922 means it does not have to pick. Two pass criteria: (1) all 66 commands are actually polled, and (2) coverage on a long drive beats 23 distinct DIDs, which is the best any long session has managed in six weeks.
- [ ] **Re-run `22F187` against module `726`.** Every other module rejected it with NRC 31, but `726` returned a malformed reply carrying VIN bytes instead. Terminal sequence: `ATSH 726`, `ATCRA 72E`, `22F187`. (Unaffected by the profile-reset issue above — this is a hand-typed terminal probe, not app polling.)
- [x] **Confirm which signalset version the Pelican app is actually holding.** Done 2026-09-20: the app was holding a stale signalset. A forced refresh brought all 109 commands into the poll cycle — `F40C` answered 172 of 172 requests, and engine speed was recorded for the first time.
- [x] **A drive with a ride-height change.** Done 2026-09-20: `3B01`, `3B3C`, all four corner pressures, both height sensors and the compressor all moved. Access height did register — in the pressures and height sensors — contrary to what it looked like from the driver's seat. Update 2026-09-20 (log review): the reason Access looked unreliable was ours — `3B3C` is a bitfield, not an ordinal. Observed values across all history are `0x01` (216 samples), `0x02` (11), `0x04` (5), `0x0D` (4); the old map ran 0-3 with 0 labelled Access, and 0 never occurs. Pinned against the inverted front height sensor, where a falling value means the truck is rising: `0x01` (front mean 111.4) is Normal, `0x02` (front mean 90.4, truck highest) is Off-Road, `0x04` (front mean 133.0, truck lowest) is Access, `0x0D` (front mean 120.5) is in transit. Map corrected to `1, 2, 4, 13`. Access should register cleanly once `7D3` is being polled again — see the profile-reset task above.
- [ ] **Confirm `3B01` bit `0x400`.** `0x100` is pinned to Normal and `0x800` to Access, both measured against the height sensor. `0x400` has been seen once and never alongside a height sample — Off-Road by elimination, unconfirmed. Needs a deliberate Off-Road selection, once `7D3` is answering again.
- [ ] **Catch the rear differential locking while `1E88` and `1E89` are being sampled.** Both look constant — `1E88` flat `0000`, `1E89` flat `09C4` — but on 6 and 25 samples respectively, none of them taken with the diff actually locked. That is not evidence they are dead, it is evidence nobody was watching at the right moment. Both are now polled at 60s. Lock the diff deliberately, hold it, and check the scan logs afterwards. If they stay flat through a real lock cycle, *then* they are dead and a manual probe sweep of `795` is the next move.
- [ ] **A cold start.** Needed for the warm-up curves on `113F` and `2104`, and to catch the crank dip on `1153`/`1154`.
- [ ] **Identify module `792`.** Dropped from the signalset entirely on 2026-09-20 — its eleven `2A3x` DIDs came back nine constants and two near-constants, and the module itself has never been identified. Not worth polling any more; still worth one terminal sweep. Try `22F18C` and `22F191` against it the way the other six unmined modules were probed on 2026-09-19.
- [ ] **Mine the four still-unidentified modules** — `716`, `726`, `737`, `797`. Each returned a distinct ECU serial via `22F18C` but rejected `22F191`, so they are real and separate but their function is unknown. (`726`'s only known DID, `0202`, has stayed `00` across 13 samples, which is far too few to call it — it is parked at 600s rather than removed. The module itself is still unidentified.)
- [ ] **Sweep module `760` for wheel speeds.** `760` was identified as the ABS/brake module on 2026-09-19 and has never been mined. Individual wheel speeds would enable dragging-brake detection, a tire-size mismatch check, and a better speed reference than the single `F40D` value. A parked terminal sweep — no drive needed.
- [ ] **Read fault codes.** This file has no DTC coverage at all — modes `03` (stored), `07` (pending), `0A` (permanent), plus UDS service `19` for the non-OBD modules. Pending codes are the earliest warning available on a vehicle this age and have never been looked at. This probably can't be expressed in an OBDb signalset, so it stays a terminal task rather than something to add to `default.json`.
- [ ] **Sanity-check the computed fuel-rate synthetic against real fill-ups.** PID `5E` isn't supported per the ECM's own supported-PID bitmask, so fuel rate can't be read directly — it's now derived from MAF and commanded lambda instead. Validated to within about 18% of the tank gauge over one 91.5 km drive; comparing against a few real fill-ups is the only way to actually calibrate it.

---

## Tire pressures: what the Jaguar signalset suggests

Jaguar reads all four tire pressures and temperatures, and fills eight Pelican
metric slots this truck leaves empty. It does it on header `751`, response
`759`, with DIDs `2076`-`2079` for pressure at 1/0.01373 bar and `2A0A`-`2A0D`
for temperature at -50 °C offset.

On this truck `751` is silent. It has been asked `2076`-`2079` 1,386 times
each and answers `NO DATA` every time, and it does not answer `0100` or
`0902` either, so nothing is listening at that address.

But the temperature DIDs have never been tried anywhere useful. `2A0A`-`2A0D`
have only ever gone to header `710` and to `FC00F1`, both of which are silent.
They have never been sent to a module that actually answers.

- [ ] **Sweep `2A0A`-`2A0D` across every module that responds** — `716`,
  `726`, `732`, `734`, `737`, `760`, `761`, `792`, `795`, `797`, `7D3`. About
  45 requests, parked, a couple of minutes. `2076` has already been tried on
  `716`, `726`, `734`, `737`, `760` and `797` and drew `7F 22 31` from the
  live ones, meaning those modules are there but do not carry that DID.
- [ ] **Try `2076`-`2079` on `761` and `7D3`**, the two answering modules that
  were skipped in the earlier round.

The spare tire warning proves the sensors exist. Something is receiving them.

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
| `761` | `197C` | 30 | Unknown, 4 distinct values in 4 samples on the 2026-09-20 drive — with `D11C`, the most active unmined DIDs on the truck | Any correlation with a cabin or body state |
| `761` | `D11C` | 30 | Unknown, 3A-41 (58-65); 4 distinct values in 4 samples on the 2026-09-20 drive — with `197C`, the most active unmined DIDs on the truck | Plausible temperature; compare against F446 ambient |
| `7D3` | `3B01` | 10 | Suspension, three values seen on the 2026-09-20 ride-height change (`00000400`, `00000100`, `00000800`) — looks like a ride-height state word | Needs enough samples to map each value to a height |
| `7E0` | `F41F` | 30 | Run time since start (PID 1F) | Records as a signal - this is the alias-mechanism test against standard 011F |
| `7E1` | `DD01` | 600 | Undecoded, constant 025166 in 1 samples (all parked) | Drop if still flat after a full drive cycle |
| `7E0` | `F466` | 300 | Raw single-byte probe from the ECM's own supported-PID bitmask decode; no data read yet | Whether it answers at all, and what payload width comes back |
| `7E0` | `F467` | 300 | Raw single-byte probe from the ECM's own supported-PID bitmask decode; no data read yet | Whether it answers at all, and what payload width comes back |
| `7E0` | `F468` | 300 | Raw single-byte probe from the ECM's own supported-PID bitmask decode; no data read yet | Whether it answers at all, and what payload width comes back |
| `7E0` | `F470` | 300 | Raw single-byte probe from the ECM's own supported-PID bitmask decode; no data read yet | Whether it answers at all, and what payload width comes back |

**Deleted and restored 2026-09-20:** `726/0202`, all eleven of `792`'s `2A3x` DIDs (`2A32`-`2A3C`), `795/1E88`, `795/1E89`, `7E1/1E6A`, `7D3/3B00`, `7D3/3B08`, `7D3/3B4D` and `7E0/F458` — nineteen commands, removed for being constant and then put straight back. The evidence did not hold up: most had between 6 and 30 samples, and `1E88`'s six samples were all taken with the differential unlocked. Only `3B4D` (236 samples across several terrain modes) was ever conclusive, and it stays in anyway at 600s because that costs 0.0017 req/s. `F458` was simply misread — `62F458 7F` is a positive reply carrying `0x7F`, not a negative response, and it is now decoded as long term secondary O2 trim bank 2. **The rule going forward: a probe needs enough samples to have covered the event it would report, not merely a lot of samples. When in doubt, park it at 600s.**

---

## Set up for a long drive with off-roading

A two-hour drive with light off-roading on 2026-09-20 was set up to exercise
the features that short commuter runs never touch. What was re-tiered for it,
and what to look for afterwards:

| Command | `freq` | What the drive was meant to produce |
|---|---|---|
| `7E0/F466` | 5 | Whether the 17.5% disagreement between the two mass airflow sensors holds up under load, or was an idle artifact |
| `795/1E88`, `795/1E89` | 30 | First data through a rear differential lock cycle. Both have been flat at `0000` and `09C4` on every sample ever taken |
| `7D3/3B4D` | 10 | Still returns `0` on every sample. A Terrain Response mode change is the only untried thing that would identify it |
| `7D3/3B01` | 10 | Three single-bit values seen so far. Enough samples across repeated height changes should map bit to height |
| `7E0/F470` | 15 | Ten undecoded bytes on a supercharged engine, under sustained load for the first time |
| `7E0/F42E` | 10 | Purge duty against fuel trims over two hours, which is the cleanest test of the stuck-purge-valve hypothesis |

Barometric pressure dropped from `freq` 1 to `freq` 5 to pay for the above.
It does not change at 1 Hz and was taking 0.9 req/s of a budget that had no
slack. Total demand is 10.53 req/s across 85 commands.

**Outcome, found in the scan logs afterward:** this is the drive that caused
the working-set discovery described at the top of this file. The app addressed
every module at 09:57, right after a signalset change, then narrowed
through 54, 58 and 19 distinct DIDs as the day went on, and was down to a
23-command working set by the time the 15:14 off-road segment started.
10.53 req/s across 85 commands was never a demand the app could sustain; it
only ever delivers about 4.2 req/s to signalset traffic. The suspension and
differential re-tiering above (`7D3/3B01`, `7D3/3B4D`, `795/1E88`,
`795/1E89`) collected nothing, because none of those commands were in that
working set. Worth keeping in proportion: 23 distinct DIDs is the widest
coverage any long session has managed since 09-05, so the drive was not an
unusually bad one. The budget
has since been cut to 3.922 req/s across 66 commands — see the verification
task in "Open in-vehicle tasks" above.

Three rows in that table are now answered, from earlier sessions in the same
log rather than from the drive itself. `7D3/3B4D` is dead: constant `0x00`
across 236 samples spanning several terrain modes, so a mode change was not
the missing ingredient. It stays in at 600s regardless. `795/1E88` and
`795/1E89` are *not* settled on the same evidence — six and 25 samples, none
with the differential locked — and both are back at 60s to catch a real
lock cycle. `7E0/F466`'s
17.5% disagreement did not survive contact with data: across 1,474 samples
the two channels sum to 0.88 of the single mass airflow PID by least squares,
with individual pairs scattering far too widely to trust the split. The
17.5% figure came from one idle sample. `7E0/F470` turned out to be manifold
pressure at 1/32 kPa rather than a separate boost sensor, tracking the
manifold pressure PID to within 1.3 kPa over 525 pairs.

## Manual-probe-only DIDs

The 2026-09-20 drive's biggest finding: 36 DIDs return `7F 22 31` (request
out of range) on every normal poll and always have. Across fourteen months
of logs they have answered only twice — once in December 2025, in a session
that had sent `10 03` (extended diagnostic session), and once in June 2025,
in a session with unusual protocol and flow-control setup. Every session
since that did not open an extended session first has been rejected.

**They are session-gated, not dead.** The app polls in the default session
and can never reach them. All 36 have been removed from the signalset —
this is why the command count dropped from 109 to 73 on this drive — and
they are now readable only by hand.

`7E0`: `1044`, `10E0`, `112C`, `1139`, `113F`, `1151`, `1152`, `1153`,
`1154`, `1155`, `1156`, `1158`, `1159`, `115B`, `1160`, `1164`, `1187`,
`11BA`, `11BB`, `11BD`, `11BE`, `11C4`, `11C5`, `11CC`

`7E1`: `0302`, `0303`, `0304`, `0305`, `0805`, `101A`, `2104`, `210F`,
`2B1B`, `2B1C`, `2B1D`, `2B1E`

The two most interesting: `1153` and `1154`, a charging-voltage pair at
1/256 V. `1154` sits pinned at exactly `0E00` (14.00 V); both dip to
5.7-10.1 V on crank.

### Reading them

`10 03` is Diagnostic Session Control, sub-function `03` — extended
diagnostic session. **This is the one place in this document where a
command changes ECU state instead of reading it.** Do it deliberately:
parked, engine running, never while driving — and close the session
afterward with `10 01` (return to default session).

The extended session times out after a few seconds of silence unless kept
alive with `3E 00` (tester present). That matters here: probe a long list
without it and the session drops back to default between questions, and
every DID on it goes back to answering `7F 22 31`.

```
ATSH 7E0      set header
ATCRA 7E8     receive filter
10 03         open extended diagnostic session
3E 00         tester present - repeat every few seconds to hold the session
22 1153       question
22 1154       question
...           more questions, with 3E 00 between any pause
10 01         return to default session
```

Retarget `ATSH`/`ATCRA` to `7E1`/`7E9` for the second module and open a
fresh `10 03` there — the extended session is per-module, not global.

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

- Both banks are correcting lean together by 7-9% long term (B1 +9.31%, B2 +7.32%), which points at fuel delivery, MAF calibration or an unmetered air leak rather than a per-bank fault. One drive, single-digit sample counts — needs confirming.
- What `761/197C` and `761/D11C` measure.
