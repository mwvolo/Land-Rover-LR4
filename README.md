# Land Rover LR4 (L319) — Signal Reference

OBDb signalset for the 2016 Land Rover LR4 / Discovery 4 — 3.0L supercharged
V6 (AJ126), ZF 8HP70, full-time 4WD with locking rear differential and
electronic air suspension.

Signals here were confirmed against ~13M logged request/response pairs from
this vehicle (VIN `SALAK2V62GA832217`). Anything still named `*_RAW` responds
on the bus but hasn't been decoded yet.

---

## Modules

Thirteen modules answer on this truck. Six have never been mined — they
reject unknown DIDs with a negative response rather than staying silent,
which is how we know they're there. Requests go to `hdr`, replies come back
from `rax`.

| Request | Response | Module |
|---|---|---|
| `7E0` | `7E8` | Engine (ECM) |
| `7E1` | `7E9` | Transmission (TCM) |
| `716` | `71E` | Unmined |
| `726` | `72E` | Unmined — returns the VIN |
| `732` | `73A` | Gear selector |
| `734` | `73C` | Headlamp control module (HCM) |
| `737` | `73F` | Unmined |
| `760` | `768` | ABS / brake control module (probable) |
| `761` | `769` | Body / instrument |
| `792` | `79A` | Unidentified — answers `2A3x` |
| `795` | `79D` | Rear differential |
| `797` | `79F` | Unmined |
| `7D3` | `7DB` | Air suspension (EAS) |

Two addressing quirks are worth knowing:

**Per-ECU DID offsets.** The engine's proprietary DIDs sit `+0x300` from
their base (`00F2` → `03F2`), while the transmission and differential use
`+0x1000` (`0E69` → `1E69`). A DID that returns "out of range" is often just
sitting at the other offset.

**`22F4xx` aliases.** Any standard OBD PID `01XX` is also readable as
`22F4XX` — `010C` (RPM) is the same data as `22F40C`. This matters because
Pelican handles standard PIDs internally, so a signalset entry for `012F`
never surfaces in the app. The `F4xx` alias arrives as a custom signal and
shows up normally. Most of this file uses that trick.

### Module identification (2026-09-19)

A UDS identification probe (`22F18C` ECU serial, `22F191` ECU part number)
was run against all six previously unmined modules:

| Module | `F18C` serial | `F191` part number |
|---|---|---|
| `716` | `0000000004757126` | rejected (NRC 31) |
| `726` | `5224293808` | rejected (NRC 31) |
| `734` | `0080-508580` | `EX53-14C243-AB` |
| `737` | `Q117eeA3516` | rejected (NRC 31) |
| `760` | `1716263MO0507` | `CH32-14C227-AB` |
| `797` | `0-00002C` | rejected (NRC 31) |

`734`'s part number is a Hella `14C243` headlamp levelling / adaptive front
lighting controller, the same base number used across Jaguar Land Rover
platforms of this era (it also shows up as `8W83-14C243-AA` on the Jaguar
XK) — identified as the Headlamp control module (HCM). `760`'s part number
matches the ABS/brake module in a published Discovery 4 (L319) ECU scan
report; that identification rests on a single external scan report, so
treat it as probable rather than confirmed. The other four modules didn't
resolve a part number, but each returned a distinct serial, so they're
confirmed to be real, separate modules even though what they do is still
unknown.

`22F187` was also tried against all six and came back NRC 31 (request out
of range) everywhere except `726`, whose reply was malformed and returned
VIN bytes instead of a rejection — worth a retry.

---

## Engine

| Signal | Address | Notes |
|---|---|---|
| Engine Speed | `7E0` `22F40C` | |
| Calculated Load | `7E0` `22F404` | Percent of available torque at current RPM |
| Absolute Load | `7E0` `22F443` | Percent of *peak possible* airflow — exceeds 100% under boost |
| Coolant Temp | `7E0` `22F405` | Thermostat opens ~88°C; normal running 90–105°C |
| Manifold Pressure | `7E0` `22F40B` | **Absolute**, not gauge. See boost calculation below |
| Barometric Pressure | `7E0` `22F433` | ~101 kPa at sea level, drops with altitude |
| Altitude | `7E0` `22F433` | Barometric converted to feet by lookup table |
| Intake Air Temp | `7E0` `22F40F` | Before the supercharger |
| Charge Air Temp | `7E0` `220520` | **After** the supercharger and intercooler |
| Timing Advance | `7E0` `22F40E` | Degrees before top dead centre |
| Throttle Position | `7E0` `22F411` | Actual plate position |
| Commanded Throttle | `7E0` `22F44C` | What the ECU asked for |
| Accelerator Pedal D / E | `7E0` `22F449` / `22F44A` | Two sensors in one pedal |
| Mass Air Flow | `7E0` `22F410` | Grams of air per second |
| Equivalence Ratio | `7E0` `22F444` | Commanded lambda |
| O2 Lambda B1S1 | `7E0` `22F434` | Measured lambda |
| O2 Sensor Voltage | `7E0` `22F434` | Second half of the same response |
| Catalyst Temp B1S1 / B2S1 | `7E0` `22F43C` / `22F43D` | One per bank |
| Oil Temp | `7E0` `2203F3` | Land Rover's own sensor. The SAE alias `F45C` was dropped — 1,169 samples against this one's 347,015 |
| Oil Level | `7E0` `2203E6` | Millimetres in the sump |
| Oil Volume | `7E0` `2203F2` | |
| O2 Lambda B2S1 | `7E0` `22F438` | Measured lambda, bank 2 — first bank 2 lambda ever read on this truck |
| O2 Sensor Voltage B2S1 | `7E0` `22F438` | Second half of the same response |
| O2 Voltage B1S2 | `7E0` `22F415` | Post-catalyst sensor, bank 1 |
| Relative Throttle Position | `7E0` `22F445` | A third throttle-position reading, alongside Throttle Position and Commanded Throttle |
| Absolute Throttle Position B | `7E0` `22F447` | A fourth |
| O2 Sensors Present | `7E0` `22F413` | Bitmask of which O2 sensor positions are physically fitted |

### Lambda

Lambda expresses air-fuel ratio relative to perfect combustion (14.7:1 for
gasoline).

- **1.00** — stoichiometric, the cruise target
- **below 1.00** — rich, extra fuel
- **above 1.00** — lean, extra air

Under boost the ECU deliberately commands rich (~0.85) because extra fuel
cools the charge and suppresses knock. A value of 2.00 means deceleration
fuel cut — injectors fully off while coasting, sensor reading pure air.

---

## Fuel

| Signal | Address | Notes |
|---|---|---|
| Fuel Level | `7E0` `22F42F` | |
| Fuel Rail Pressure | `7E0` `22033E` | Proprietary; ~60 bar idle, 130+ under load. The SAE alias `F423` was dropped — 158 samples against this one's 15,910 |
| Short Term Fuel Trim B1 | `7E0` `22F406` | Immediate correction, swings constantly |
| Long Term Fuel Trim B1 | `7E0` `22F407` | Learned correction, drifts slowly |
| Short Term Fuel Trim B2 | `7E0` `22F408` | First recorded 2026-09-20 |
| Long Term Fuel Trim B2 | `7E0` `22F409` | First recorded 2026-09-20 |
| Fuel System Status B1 | `7E0` `22F403` | Enum: open loop cold / closed loop / open loop, load or decel fuel cut / open loop, system fault / closed loop with a feedback fault |
| Fuel System Status B2 | `7E0` `22F403` | Second half of the same response; not yet mapped to named states |
| Commanded Evap Purge | `7E0` `22F42E` | Percent duty cycle on the purge valve |
| O2 Trim B1S2 | `7E0` `22F415` | Second half of the same response — post-catalyst trim, bank 1 |
| Long Term Secondary O2 Trim B1 | `7E0` `22F456` | |

---

## Duplicates found and removed

Four signals were checked against another reading of the same physical
quantity, using paired samples across logged history rather than a single
drive. Two turned out to be the same measurement and were dropped; two
turned out to be genuinely separate sensors and were kept.

**PID 67 sensor 1 is the engine coolant PID.** Mean difference 0.0°C over
135 paired samples. Deleted. Sensor 2 of the same PID is kept, renamed to
Charge Cooler Coolant Temp: it ran 11.7°C above ambient and 24°C below
charge air temperature over those same samples, which is where the AJ126's
separate low-temperature intercooler circuit belongs. A genuinely new
signal, not a rename in name only — but it carries no `suggestedMetric`,
so it will never be stored as history by the app; see "Only signals with a
metric are ever recorded" under Polling.

**PID 70 channel A is manifold pressure, at finer resolution — not a
separate boost sensor.** This corrects "F470 was boost pressure all along,"
decoded earlier the same day. PID 70's support byte reads `0x02`; three of
its four defined channels read constant zero across 525 samples and were
dropped. The fourth, channel A, tracks Manifold Pressure to within 1.3 kPa
mean over 525 pairs (r = 0.85) — the same manifold sensor, reported at 1/32
kPa instead of 1 kPa, not a pre-throttle sensor. Kept, renamed to Manifold
Pressure (Fine), slowed to 120s since the extra resolution isn't worth a
faster poll.

**The proprietary oil temperature DID and standard PID 5C agree to 0.7°C**
over 1,135 paired samples once warm, diverging by up to 15°C during
warm-up with PID 5C reading higher. Dropping PID 5C earlier was correct,
and it stays out.

**PID 68 sensor 2 tracks the proprietary charge air temperature DID to
2.7°C mean** — close to redundant, but kept: sensor 1 of the same PID is
the pre-supercharger inlet, running 9.2°C above ambient, and is a real
second reading the truck doesn't report anywhere else. Both kept at 120s
with descriptions.

---

## Drivetrain

| Signal | Address | Notes |
|---|---|---|
| Gear Selector | `732` `22D928` | 0 Park · 1 Reverse · 2 Neutral · 3 Drive · 7 Sport |
| Gearbox Temp | `7E1` `221E69` | ZF 8HP70 sump |
| Locking Diff Oil Temp | `795` `221E8A` | |

---

## Suspension

All four corners report gauge pressure in the air spring.

| Signal | Address |
|---|---|
| Pressure Front Left / Right | `7D3` `223B04` / `223B03` |
| Pressure Rear Left / Right | `7D3` `223B06` / `223B05` |
| Height Offset | `7D3` `222B12` | Signed, millimetres from nominal |
| Compressor Activity | `7D3` `223B07` | ~110 at rest, over 1,300 while the truck raises |
| Ride Height Mode | `7D3` `223B3C` | Enum: 1 Normal, 2 Off-Road, 4 Access, 13 In Transit. Corrected 2026-09-20 — see "Air suspension notes" under Off-road |
| Height Sensor Front / Rear | `7D3` `223B71` / `223B72` | **Inverted** — falls as the truck rises |
| Module Voltage | `7D3` `22D11A` | Should mirror battery voltage |

Normal standing pressures are roughly 40 psi per corner at normal ride
height, rising with load and with raised height modes.

**None of this module's signals carry a `suggestedMetric`, and the app
only ever stores signals that do** — see "Only signals with a metric are
ever recorded" under Polling. Corner pressures, ride height, compressor
activity and module voltage are all live-only: even once `7D3` is
un-retired and answering again, nothing from this table will ever show up
in the app's own history, only in the raw scan logs.

---

## Reading signals together

Individual numbers are useful. Pairs are where the diagnostics live.

### Boost — Manifold Pressure minus Barometric Pressure

The LR4 reports manifold pressure as *absolute*, so at idle it reads well
below atmospheric and looks wrong. Subtract barometric and you get true
boost:

```
boost = Manifold Pressure − Barometric Pressure
```

Negative is vacuum (closed throttle, coasting). Zero is atmospheric. Positive
is supercharger boost. Because barometric is a live reading, this stays
correct as you gain or lose altitude.

### Heat-limited power — Charge Air Temp with Timing Advance

Charge air temp is the truck's honest report of whether the intercooler is
keeping up. Watch it with timing advance:

| Charge air temp | Timing advance | Meaning |
|---|---|---|
| Near ambient | Normal | Healthy, full power available |
| Climbing | Retarding | ECU pulling timing to prevent knock — power is fading |
| High, stable after stopping | — | Heat soak; recovers once moving |

Timing retarding *without* a temperature rise points at fuel quality or knock
sensor activity instead.

### Fueling health — Commanded vs Measured Lambda

Equivalence Ratio is the target, O2 Lambda is reality. They should track
closely. A persistent gap means fueling isn't achieving its target — failing
sensor, vacuum leak, or fuel delivery shortfall.

### Total fuel correction — Short Term plus Long Term Trim

Add them. Their sum is how far off the base fuel map the engine is running.

| Sum | Meaning |
|---|---|
| Within ±5% | Normal |
| Around +20% at warm idle | Adding fuel to compensate — vacuum leak or lazy MAF |
| Persistently negative | Running rich — leaking injector or high fuel pressure |

Short term swinging while long term stays near zero is normal closed-loop
operation. Long term drifting away from zero is the engine *learning* a
problem, which is the one to act on.

### Sensor cross-checks

Several values are measured twice by independent paths. Agreement validates
both; disagreement localises the fault.

| Pair | Should agree within |
|---|---|
| Accelerator Pedal D vs E | A percent or two — they're redundant by design |

Oil Temp and Fuel Rail Pressure each had an SAE alias checked against the
proprietary signal early on (0.7°C and similar magnitude respectively);
both aliases were dropped as duplicates and are gone from this file, so
there's nothing left to cross-check them against day to day.

Pedal D and E disagreeing is significant: the ECU compares them itself and
will enter limp mode if they diverge.

### Bank imbalance — Catalyst Temp B1 vs B2

The two banks should run within ~30°C of each other. A persistent split means
one bank is working harder — a misfire, an injector, or an O2 sensor on the
cooler side.

### Load distribution — the four suspension pressures

Front and rear pairs should be near-symmetrical left to right. A single
corner reading consistently low is either a load imbalance or an air spring
losing pressure. Compare against Height Offset: if pressure is low but height
is nominal, the system is compensating for a slow leak.

### Warm-up sequence — Coolant, Oil, Gearbox

These heat at different rates, and the order tells you about circulation.
Coolant reaches operating temperature first, oil follows several minutes
later, gearbox last. Oil temp lagging far behind coolant on a long drive
suggests restricted flow. Gearbox temperature climbing above normal while
towing is the signal to back off.

### Airflow sanity — MAF against RPM and Load

At idle, roughly 3–8 g/s. Under full boost, ten times that or more. MAF
staying flat while load and RPM climb means the sensor is under-reporting —
which the fuel trims will then try to correct, showing up as a rising
positive trim.

---

## Computed signals

The signalset defines eight synthetic signals. Seven are a **ratio between
two readings that should hold a known value**, which makes them suited to
a display: you learn the normal number once, and anything else is a
signal. The eighth, Fuel Rate, is a different shape — a real physical
quantity rather than a 1.0-normal ratio — and is described separately
below. (An earlier eighth ratio, Rail Pressure Crosscheck, was removed
when its SAE rail-pressure alias was dropped as a duplicate; Fuel Rate
took its slot.)

| Signal | Ratio | Normal | Meaning when it moves |
|---|---|---|---|
| Boost Pressure Ratio | MAP / Barometric | 1.0 idle, up to ~1.8 | Above 1.0 is supercharger boost, self-correcting for altitude |
| Lambda Tracking | Measured / Commanded lambda | 1.0 | Fuelling isn't hitting its target — sensor, leak, or delivery |
| Bank Balance | Cat temp B1 / B2 | 1.0 | One bank working harder — misfire or injector on the low side |
| Pedal Agreement | Pedal D / Pedal E | 1.0 | Redundant pedal sensors disagreeing; the ECU limps if they diverge |
| Throttle Tracking | Actual / Commanded throttle | 1.0 | Plate not following orders — sticky or carbonned throttle body |
| Suspension Balance Front | Front left / right pressure | 1.0 | A corner losing air, before the dash warns |
| Suspension Balance Rear | Rear left / right pressure | 1.0 | Same, rear axle |

All seven sit at **1.0 when healthy**, so a single glance covers fuelling,
ignition balance, pedal and throttle integrity, and air springs.

Boost Pressure Ratio is the exception and the one to watch for fun: it's the
closest thing to a boost gauge this vehicle exposes. The schema's only
operation is division, so true gauge boost (`MAP − Barometric`) isn't
expressible as a synthetic — but the ratio carries the same information and
needs no altitude correction.

### Fuel rate, now computed

The ECM's own supported-PID bitmasks (`0100 = BFBFACD3`, `0120 = A007B119`,
`0140 = FED08511`, `0160 = 07010000`) say PIDs `5E` and `9D` are both
unsupported. The truck does not report fuel rate. Any app showing one is
computing it, and now this signalset does too — the eighth synthetic,
added 2026-09-20.

```
Litres per hour = MAF (g/s) / (Lambda × 14.7 × 745 / 3600)
                = MAF / (Lambda × 3.0421)
```

The schema's synthetic operation is a plain ratio with no constant, so the
constant was folded into a hidden operand instead: `LR4_FUEL_DIVISOR` reads
Commanded Equivalence Ratio (`22F444`) with `div` set to `10771.53`
(`32768 / 3.0421`), which yields `Lambda × 3.0421` directly. `LR4_FUEL_RATE`
is then Mass Air Flow divided by that, already in litres per hour, and
carries `suggestedMetric: fuelRate`.

Validated against the 2026-09-20 drive: integrating the formula over 1,480
MAF samples gives 11.21 L burned. The tank gauge fell from 87.8% to 76.9%
over the same drive, which on the 86.3 L tank is 9.48 L — agreement to
about 18%, on the pessimistic side, over 91.5 km (19.2 mpg by the
formula). It's an estimate, not a measurement: the gauge is 8-bit (one step
is 0.34 L), and float angle off-road makes it worse. Treat Fuel Rate as
directionally useful, not a trip computer.

---

## Altitude and mountain driving

### Reading altitude

There's no altitude signal, but **Barometric Pressure is one** — air thins
predictably with height. Read `22F433` directly:

| Altitude | Barometric | In psi |
|---|---|---|
| Sea level | 101 kPa | 14.7 |
| 2,000 ft | 94 kPa | 13.7 |
| 4,000 ft | 88 kPa | 12.7 |
| 6,000 ft | 81 kPa | 11.8 |
| 8,000 ft | 75 kPa | 10.9 |
| 10,000 ft | 70 kPa | 10.1 |
| 12,000 ft | 64 kPa | 9.3 |

Roughly 1 kPa per 300 ft near sea level, stretching to about 340 ft
per kPa above 8,000 ft. Weather shifts it a few kPa, so treat
it as approximate unless you calibrate against a known elevation.

### Measured on this truck

From the September 2025 mountain trip, barometric pressure ranged **65–76 kPa,
median 70** — roughly 7,800 to 11,800 ft, spending most of its time near
10,000. Those runs give real baselines to compare against:

| Signal | Mountains, Sept 2025 | Notes |
|---|---|---|
| Charge Air Temp | 6–81°C, median 47 | Wide swing with grade and airflow |
| Gearbox Temp | 8–85°C, median 64 | Never troubling |
| Diff Temp | 0–60°C, median 35 | Cooler than a hot-weather lowland drive |
| Coolant Temp | up to 103°C | Normal ceiling on sustained climbs |
| Mass Air Flow | peak 216 g/s | Full-load airflow |

Worth noting the differential ran *hotter* on flat August driving (median
71°C) than in the mountains (35°C). Ambient temperature matters more to it
than terrain does.

### Fuel trim climbs with altitude

Long term fuel trim tracks altitude clearly across ~76,000 logged samples:

| Barometric | Altitude | Median LTFT |
|---|---|---|
| 100 kPa | ~400 ft | +8.6% |
| 80 kPa | ~6,400 ft | +7.8% |
| 75 kPa | ~7,900 ft | +12.5% |
| 70 kPa | ~9,400 ft | +14.1% |
| 65 kPa | ~10,900 ft | +14.8% |

Altitude adding fuel trim is expected — the mass air flow sensor tends to
under-report in thin air and the ECU compensates. **The part worth watching
is the low-altitude baseline of around +8%.** A healthy engine sits within
±5%, so there's a mild lean condition present before altitude adds anything,
and the two stack: near +15% at 11,000 ft is approaching where a lean code
would set.

Most likely cause on this engine is a small unmetered air leak — the
supercharged V6 has a lot of intake plumbing and hose joints. Not urgent, but
if the truck ever feels flat at altitude, this is the reason to look at
first.

### What altitude does to the engine

A naturally aspirated engine loses about 3% power per 1,000 ft. Yours loses
much less, because the supercharger is a fixed-displacement blower geared to
the crank — it keeps stuffing the same volume in regardless of ambient
pressure. What you'll see instead:

- **Boost Pressure Ratio holds steady.** It's normalised against barometric,
  so it reads the same at 10,000 ft as at sea level. That's the point of
  using the ratio rather than raw manifold pressure.
- **Mass Air Flow drops** at the same throttle and RPM. Thinner air, less
  mass, less fuel, less power — the loss the blower can't fully erase.
- **Fuel trims may drift positive** as the ECU adapts. Mild drift climbing a
  long pass is normal, not a fault.

### What to watch climbing

Sustained high load is the hardest thing you'll ask of the truck.

| Signal | Watch for |
|---|---|
| Charge Air Temp | Climbing on a long pull, with Timing Advance retarding — the ECU protecting itself, and where your power goes |
| Gearbox Temp | The one that matters. Sustained climbing in a low gear heats the ZF fast |
| Coolant Temp | Should stay near 100°C. Rising past that on a grade means the cooling system is at its limit |
| Absolute Load | Pinned near 100% for long stretches means you're asking for everything available |

### What to watch descending

Engine braking sends heat somewhere different.

- **Lambda goes to 2.00** on a closed-throttle descent. That's deceleration
  fuel cut, working exactly as intended — you're using no fuel at all.
- **Gearbox Temp still climbs** in a low gear, even off-throttle.
- **Coolant Temp can rise** on a long descent despite low load, because
  airflow is low and the engine is being driven by the wheels.

---

## Off-road

### The mountain drive that tested off-roading collected no off-road data

Session 2525 (2026-09-20, 15:14–17:32, 137.5 minutes, 91.5 km / 56.9 mi at
7,800–11,000 ft) deliberately included light off-roading to exercise the
suspension and rear differential. It produced none of that data. Every
signal the test was aimed at lives on a module the app had already stopped
addressing before the drive started: ride height, height sensors, corner
pressures, compressor activity and Terrain Response all live on `7D3`,
which the app retired at 14:07 that day, over an hour before the drive
began. The two rear-differential-lock candidates lived on `795`, which the
app was still addressing, but neither DID was among the 23 commands it
chose to poll that session — see "The app retires ECUs permanently, one
session at a time" under Polling for why.

Everything below about air suspension and the rear differential was
recovered from earlier sessions already present in the same log file, not
from this drive. Until the app's learned vehicle profile is reset, no
suspension or gear-selector data will be collected on a drive no matter
what this signalset asks for.

And even once it is reset: none of the suspension signals carry a
`suggestedMetric`, so none of them will ever be recorded as *history* by
the app regardless — see "Only signals with a metric are ever recorded"
under Polling. Un-retiring `7D3` restores live readouts and raw scan-log
coverage, not a trip history for ride height or corner pressure.

### Air suspension

The four corner pressures are the most useful thing you have off-road. On
uneven ground the truck articulates, and pressures diverge as weight shifts.

- **Suspension Balance Front / Rear** — the two synthetic ratios. At 1.0 the
  axle is evenly loaded. Away from 1.0, weight has moved to one side.
- A corner going very low while its opposite goes high means that wheel is
  unloading — the first hint of lifting a wheel.
- **Height Offset** shows where you are relative to normal ride height.

Back on pavement both balance ratios should settle at 1.0. If one doesn't,
you've either shifted your load or picked up a slow leak.

### Watch on the trail

| Signal | Why |
|---|---|
| Gearbox Temp | Low-range crawling generates heat with almost no airflow. The number to respect |
| Locking Diff Oil Temp | Same reason, and it climbs fast when the diff is locked and working |
| Coolant Temp | Low speed means low airflow; the fan is doing the work alone |
| Suspension Balance | Articulation and weight transfer, live |
| Gear Selector | Confirms what the transmission thinks it's in |

A drive on 2026-09-20 that changed ride height confirmed Ride Height, the
height sensors and the corner pressures all move together and in the
directions their labels predict — see "Air suspension notes" below. `3B4D`,
once considered for Drive Mode or Terrain Response, is neither: a later
audit across 236 samples spanning many drives and several terrain modes
found it constant at `0x00` throughout, and it has been deleted. Terrain
Response itself remains unidentified on this truck — nothing currently in
the signalset is known to carry it.

### Before you go

Air suspension raises the truck at low speed and drops it as you speed up. If
you're crawling and it lowers unexpectedly, watch Height Offset and the
corner pressures — an EAS fault normally means the truck sits down, which
matters a lot more with rocks under it than in a parking lot.

---

## Undecoded

These respond on the bus but their meaning isn't established. They're in the
signalset so their values get logged during normal driving, which is how the
formulas eventually get worked out.

| Address | Signals |
|---|---|
| `7E1` / `7E9` | `1E68`, `DD01` — `1E68` is a neighbour of the gearbox temp and varies most of anything unmined on this module; `DD01` shares its DID with the engine's odometer but isn't one |
| `761` / `769` | `197C`, `D11C` — the most active unmined DIDs on the truck, four distinct values each in four samples |
| `7D3` / `7DB` | `3B01`, `3B02`, `3B0B` — `3B01` looks like the ride-height state word (see "Air suspension notes" under Off-road); `3B02` answers four single-byte values that look like one per corner |

Nineteen commands that used to sit in this section were deleted on
2026-09-20, on evidence across all logged history rather than a single
drive:

| Command | Evidence | Verdict |
|---|---|---|
| `726` / `0202` | One value (`0x00`) in 13 samples | Dead |
| `792` / `2A32`–`2A3C` (eleven commands) | Nine return a single constant, two return two values each; the module itself was never identified | Dead. Moved to `TESTING.md` as a manual probe |
| `795` / `1E88` | Constant `0x0000` in 6 samples | Dead. One of the two rear-diff-lock candidates |
| `795` / `1E89` | `0x09C4` in 24 of 25 samples | Dead. The other diff-lock candidate |
| `7E1` / `1E6A` | Constant `0x00` in 29 samples | Dead |
| `7D3` / `3B00` | `0x00000003` in 54 of 55 samples | Dead |
| `7D3` / `3B08` | Constant `0x00` in 64 samples | Dead |
| `7D3` / `3B4D` | Constant `0x00` across 236 samples spanning many drives and several terrain modes | Dead. This was the Terrain Response candidate — it isn't, and appears to be nothing. See "Terrain Response and 3B4D" below |
| `7E0` / `F458` | Returns a `7F` negative response | Dead |

Deleting a command this way — evidence across the full log history rather
than one drive — is the standard this file now holds new deletions to. A
DID that stays flat through a real height change, a real terrain-mode
cycle, and everything in between doesn't need a second chance.

**The suspension decodes check out.** Replaying every logged sample through
the formats in this file gives corner pressures of 34-50 psi, a ride height
offset correctly signed at -91 to +50 mm, and module voltage of 9.12-14.56 V,
all physically sensible.

**Terrain Response and `3B4D`.** `3B4D` was labelled Drive Mode and was the
candidate for Terrain Response. Across all logged history it returns a
constant `0x00` — 236 samples, spanning many drives and several terrain
modes, not just the one 2026-09-20 drive that first flagged it as flat.
That's not "unconfirmed," it's answered: `3B4D` is not Terrain Response and
appears to be nothing. It has been deleted rather than kept as a probe —
see "Nineteen commands deleted" above.

What the audit could not do is say anything about suspension health. Across
fourteen months only 36 minutes have all four corner pressures captured
together, because the `7D3` module receives one or two requests per drive
despite its `freq` of 10.

**Tire pressures remain unsolved, but they exist.** Module `751` returns no
data to any request, and nothing found so far resembles four tire pressures.
The decisive clue is the spare: this truck warns when the *spare* is low, and a
spare does not rotate — so there is no wheel-speed difference to infer it from.
That rules out an indirect system and means real pressure sensors are fitted and
reporting to some module. Six modules answer negative responses but have never
been mined — `716`, `726`, `734`, `737`, `760`, `797` — and each is now probed
with the Jaguar tire-pressure DID as a locator. A positive response from any of
them finds the module.

### Air suspension notes

A 33-minute drive on 2026-09-20 that changed ride height twice gave the first
real test of these fields, and most of them passed it.

`3B71` and `3B72` track the height sensors, and they read **inverted** — the
number falls as the truck rises. Observed: 115/115 at normal height, 93/101
raised; on the 2026-09-20 drive `3B71` fell from `6A` to `56` on the raise,
then rose to `85` by the end. Treat a falling value as the truck going up.

The four corner pressures (`3B03`-`3B06`) tracked the same drive unambiguously:
around `00C5` cruising, down to about `00AC`, up to about `00FE` at 16:22:42
UTC as the truck raised, then down to `0096`-`009F` at 16:23:49 as it lowered.
Compressor activity (`3B07`) jumped from around `00A0` to `05F0` and `0617`
during those changes.

The driver believed the low "access" height didn't register. It did — the
corner pressures and height sensors both recorded a third, lower level at
16:23:49. What was wrong was the map, not the sampling, and a later audit
across all logged history settled it.

**The ride height map was wrong, and that is why Access never registered.**
`3B3C` across all logged history returns `0x01` (216 samples), `0x02` (11),
`0x04` (5) and `0x0D` (4). The map in this file was ordinal, 0 through 3,
with 0 labelled Access — but the value 0 never once occurs, and `0x04` and
`0x0D` weren't in the map at all. Pinned against the front height sensor,
which is inverted so a falling value means the truck is rising:

| Value | Front height sensor mean | State |
|---|---|---|
| `0x01` | 111.4 (range 91–122) | Normal |
| `0x02` | 90.4 (85–95), highest the truck sits | Off-Road |
| `0x04` | 133.0, lowest the truck sits | Access |
| `0x0D` | 120.5, between Normal and Access | In Transit |

The map is now `1, 2, 4, 13`. `3B01` carries the same state in parallel:
`0x100` pins to a front mean of 111.6 (Normal) and `0x800` to 133.0
(Access). `0x400` has been seen once and never alongside a height sample, so
it's Off-Road by elimination — unconfirmed, unlike the other three.

The two balance ratios respond to cargo, not just faults. With the load area
full, front balance read 0.99 and rear read 0.92 — the rear axle carrying more
on one side. Check the ratios unloaded before reading a low number as a leak.

---

## Polling

### The budget was measured against the wrong ceiling

For most of this file's history the budget was set against **13
request/response round trips per second**, treated as the adapter's
ceiling. That number is real, but it's the whole bus, and most of it
belongs to the app, not to this signalset. Measured on 2026-09-20 (session
2525): total adapter throughput was 13.43 req/s, splitting into AT
commands (adapter setup, header changes, filter changes) at 3.27/s, the
app's own internal mode-01/mode-09 polling — which never surfaces as a
recordable signal — at 5.94/s, and this signalset's UDS service-`22`
traffic at 4.22/s.

Across seven sessions on 2026-09-19 and 2026-09-20, the UDS share stayed
pinned between 4.1 and 5.1 req/s no matter how large the signalset was.
Session 2504 polled 108 distinct DIDs and got 4.19 req/s. Session 2525
polled 23 distinct DIDs and got 4.22 req/s — five times fewer commands,
identical throughput. **The signalset's real ceiling is about 4.2 req/s,
not 11.** This file had been asking for 10.53.

Service `22` is read-only. Polling cannot harm anything; it can only
compete with itself, and with the app's own traffic, for a share that
doesn't grow no matter how much is asked of it.

**`freq` is a minimum interval in seconds, not a rate.** Pelican documents it as
the maximum frequency at which a command may be sent, expressed in seconds, so
a *smaller* number polls *harder*. There is no priority field anywhere in the
v3 format — command count and `freq` are the only levers.

The budget in this file, cut to fit the measured ceiling on 2026-09-20:

| Interval | Commands | Contents |
|---|---|---|
| 2s | 2 | Engine speed, vehicle speed |
| 5s | 6 | Calculated load, throttle position, manifold pressure, mass air flow, commanded equivalence ratio, gear selector |
| 10s | 3 | Lambda B1S1, fuel rail pressure, ride height mode |
| 15s | 9 | Timing advance, absolute load, both pedal sensors, coolant temp, short term trim B1, both height sensors, `3B01` |
| 30s | 15 | Remaining temperatures (oil, charge air, gearbox, diff), fuel trims bank 2, commanded throttle, fuel level, MAF A/B, all four corner pressures, compressor activity |
| 60s | 13 | Height offset, module voltage, lambda B2S1, ambient, barometric/altitude, battery, catalyst temps, IAT sensors, `3B02`, `3B0B`, `1E68` |
| 120s | 11 | Slower diagnostics: fuel system status, evap purge, O2 voltage B1S2, secondary O2 trim, relative/absolute throttle, IAT 1/2, manifold pressure (fine), run time, `197C`, `D11C` |
| 300s | 1 | Distance since codes cleared |
| 600s | 6 | Odometer, oil level, oil volume, distance with MIL on, warm-ups since codes cleared, `DD01` |

That's **66 commands at 3.922 req/s**, down from 85 commands at 10.528 —
comfortably under the measured 4.2 req/s ceiling, where the old figure had
looked safe against 11 req/s but was actually overdrawing the real one by
more than double. Engine speed is back on a real cadence (2s) and carries
the `engineSpeed` metric; it had not been polled since 2026-08-30 — see
below for why.

### The app retires ECUs permanently, one session at a time

Headers the app actually addressed, by session, on 2026-09-20:

| Time | Headers addressed |
|---|---|
| 09:57 | `726`, `732`, `761`, `792`, `795`, `7D3`, `7DF`, `7E0`–`7E7` — everything this file names |
| 10:59 | `726` and `761` gone |
| 12:40 | `732` and `792` gone |
| 14:07 | `7D3` gone |
| 15:14 (session 2525, the off-road drive) | Unchanged from 14:07 |

Once a module is dropped it is never retried. This is a ratchet, not a
fluctuation, and it isn't about the modules going bad: in the 12:40 session
every `7D3` command returned a valid response right up until the app
stopped asking.

During session 2525's 137-minute drive, 61 of this file's 84 commands (the
count before today's cut to 66) were never sent once — not throttled, not
degraded, never requested. The 23 that were polled all start at minute 0.1
or 0.6 and run continuously to minute 137.4, so this isn't a mid-drive
dropout either. It also isn't the stale-signalset explanation from
2026-08-30, reframed below: no commit in this file's history matches the
polled set, and four commands added earlier the same day were among the 23
being polled — a stale copy of the file wouldn't include same-day
additions.

**A command's presence in this file does not mean it is being collected.**
Check the scan logs before relying on one. Nothing in this repo can
un-retire a module — the app has learned that `7D3`, `792`, `732`, `726`
and `761` don't answer and has stopped addressing them, and that has to be
reset from inside the app (probably by removing and re-adding the
vehicle), not from this file. Whether staying under the 4.2 req/s ceiling
stops the ratchet from taking more modules is untested; the next drive is
the test.

### Eight commands went quiet for three weeks — the first sighting of the ratchet, not a separate incident

This section originally explained an isolated 2026-08-30 incident as a
stale copy of the signalset on the app's side. It wasn't isolated. It was
the first visible symptom of the retirement ratchet described above, and
at the time nobody knew the ratchet existed. The history below is kept
as-is because the reasoning in it — ruling out the request budget and the
`freq` tiers before landing on "the app must be holding stale data" — was
a reasonable read of the evidence available that day. It just wasn't the
right answer.

Between 2026-08-30 and 2026-09-19, eight commands in this file were never
requested on a drive: `F40C`, `F411`, `F40E`, `F443`, `F449`, `F44A`, `F407`
and `033E`. The cost was real — `engineSpeed` is wired to `F40C`, and no
engine speed value reached the signal database in fourteen months. Two
synthetics could not compute either, `LR4_THROTTLE_TRACKING` needing `F411`
and `LR4_PEDAL_AGREEMENT` needing `F449` and `F44A`.

**Looked resolved on 2026-09-20, briefly.** Every command in the file was
polled that morning. `F40C` was requested 172 times and answered 172
times, and engine speed was recorded for the first time in fourteen
months. At the time this was credited to the app picking up a fresher copy
of the signalset. Later the same day, session 2525 showed the real
pattern: the app hadn't fixed anything, it had simply not yet retired the
modules carrying those eight commands. It went on to retire four more
modules that same day.

Getting to the "stale signalset" explanation meant ruling out the two
obvious causes, and both remain worth knowing even though the explanation
built on them was wrong. It was not the request budget: demand was 10.21
req/s against the 11-13 the adapter appeared to deliver — appeared to,
because that 11-13 figure was the whole-bus number, not this signalset's
real share. It was not the `freq` values either, because the dead commands
shared tiers with live ones — `F40C` and `F40D` are both `freq` 1 and only
`F40D` ran; `F411`, `F443`, `F449` and `F44A` sat at `freq` 3 alongside
`F404`, `F434` and `F444`, which all ran.

The lesson that outlives the incident, revised: **a command's presence in
this file means nothing until the scan logs confirm the app is actually
addressing its module this session.** Before concluding a command is
unsupported, or that a fix worked, check which headers the app is
addressing — not just which commands answer when you send them by hand.

### The seven probes, and how they turned out

Seven commands were added on 2026-09-20 to be watched rather than trusted.
The 2026-09-20 drive settled all of them.

**`7E0/F408` and `7E0/F409` answered — the bank-2 fuel trims exist.** They
had never been requested on this vehicle before. They are now real signals
rather than probes, and what they show matters: see below.

**`7E0/1153`, `7E0/1154`, `7E0/113F` and `7E1/2104` returned `7F 22 31`.**
They are session-gated, not dead — see the manual-probe list in
`TESTING.md`. The charging-voltage reading behind `1153`/`1154` still
stands (`1154` pinned at exactly `0E00`, 14.00 V, with both dipping to
5.7-10.1 V on crank, tracking nothing that `F442` does), but it can only be
read by hand in an extended diagnostic session, so those commands have been
removed from this file.

**`726/0202` answered and stayed at `00`** across twelve samples.

`F41F` also recorded, which was the incidental test of whether the `F4xx`
alias mechanism works at all against a standard PID the app already polls.
It does.

### Both banks are running lean, and they agree with each other

First bank-2 data from this truck, measured over the 2026-09-20 drive:

| | Bank 1 | Bank 2 |
|---|---|---|
| Short term | -0.55% (n=305) | -5.36% (n=7) |
| Long term | **+9.31%** (n=13) | **+7.32%** (n=8) |

Two points apart on the long-term trims is not an asymmetry. Both banks are
correcting lean together by 7-9%, which points at something shared — fuel
delivery, MAF calibration, or unmetered air upstream of where the intake
splits — rather than a fault on one side. Any earlier reading of this as a
bank asymmetry came from having only bank 1 to look at.

Sample sizes are small: one drive, and single digits on three of the four
figures. Treat the direction as real and the magnitude as provisional.

`F403`, `F42E` and `F438`, added to the file below, exist specifically to
test this: whether the engine was in closed loop when the trims were
measured, whether a stuck-open purge valve is pulling in unmetered air, and
what bank 2's own oxygen sensor is doing behind its trim.

### The debug batch, and what the first drive did to it

On 2026-09-20, 51 commands went in — one for every DID this truck had ever
answered that nothing here decodes — each a raw scalar at its observed byte
width, named `LR4_<DID>_RAW`. The first real drive cut that back hard.

**Thirty-six of them are session-gated.** The whole `7E0` `10xx`/`11xx`
family and the `7E1` `0302`-`0305`, `0805`, `101A`, `2104`, `210F`,
`2B1B`-`2B1E` block returned `7F 22 31`. In fourteen months of logs they
have answered in exactly two sessions: one in December 2025 that sent
`10 03` to open an extended diagnostic session, and one in June 2025 doing
unusual protocol and flow-control setup. Every session since that did not
open an extended session has been rejected.

The app polls in the default session, so it can never reach them. All 36
are out of this file and listed in `TESTING.md` as manual-probe-only. This
also rewrites the history: `1cf20db` dropped many of them as "dead probes",
and they were never dead — they were session-gated, and nobody had noticed
the December session opened `10 03` first.

What survived, and how it was re-tiered after the drive:

| Command | Change | Why |
|---|---|---|
| `7D3/3B01` | `freq` 600 → 10 | Returned three single-bit values; looks like the ride-height state word |
| `7D3/3B00`, `7D3/3B08` | `freq` 600 → 30 | Still flat through a real height change, but on 3 samples each — one more look before deletion |
| `761/197C`, `761/D11C` | `freq` 60 → 30 | Four distinct values in four samples each; the most active unmined DIDs on the truck |
| `792/2A32`, `792/2A37` | unchanged | Each ticked by a small amount, consistent with counters. The rest of the `2A3x` block stayed flat |
| `726/0202`, `795/1E88`, `795/1E89` | unchanged | Still flat; the diff-lock cycle is what will decide the `795` pair |

The file is now 73 commands at 10.55 req/s.

A DID that moves on a real drive earns a proper decode. One that stays flat
through a full cycle — including a ride-height change and a terrain-mode
switch — can be deleted for good rather than on suspicion. Do not read a
`7F 22 31` as "unsupported" without checking whether an extended session
would have changed the answer.

### Sixteen more, found by reading the ECM's own PID list, and how they did

The ECM's `0100`/`0120`/`0140`/`0160` bitmasks report 54 supported mode-01
PIDs. Twenty-five had no entry here; after removing metadata PIDs and the
two SAE aliases already dropped, sixteen went in on 2026-09-20 and were
tested the same day. **All sixteen answered.** Ten earn their place.

| Command | Result |
|---|---|
| `22F466` | **Two mass airflow sensors.** A 2.09 g/s against B 1.78 g/s, summing to 3.88 against `F410`'s 4.16 a second later. One idle sample — see the correction below |
| `22F467` | **Two coolant temperature sensors**, 61 °C and 35 °C |
| `22F468` | **Two intake air temperature sensors**, 36 °C and 59 °C |
| `22F403` | Fuel system status, two states over 14 samples: open loop on load or decel ×9, closed loop ×5 |
| `22F42E` | Evap purge duty, 0 to 72.2% |
| `22F438` | Bank 2 lambda 0.853-1.985 over 29 valid samples; railed at `FFFF` on 23 of 52 |
| `22F415` | Post-catalyst O2, 0.13-0.93 V and switching |
| `22F445` / `22F447` | Relative throttle 2.7-22.7%, absolute throttle B 12.5-22.4% |
| `22F470` | Decoded against SAE J1979 PID 70: channel A reads 24.72 kPa at idle. Later shown to be manifold pressure at finer resolution, not a separate boost sensor — see "Duplicates found and removed" above |

**The dual mass airflow sensors exist, but the 17.5% disagreement claim
above was wrong — it rested on a single idle sample.** With 1,474 paired
samples from the 2026-09-20 drive, the least-squares slope of (A+B) against
the single MAF PID through the origin is 0.88, not 1.0, and individual
pairs scatter badly: one pair reads 7.03 g/s against the single PID's
18.16 g/s at the same instant. The split between the two channels is not
trustworthy at this polling rate. Use the single MAF PID for anything
quantitative; treat A and B as present but unvalidated.

`F403` matters for the same investigation: the engine does reach closed
loop, so the 7-9% long-term trims are real measurements rather than
open-loop artifacts.

**Four were declared wrong and recorded nothing useful.** `F466`, `F467`,
`F468` and `F470` were added at `len` 8 because their payload widths were
unknown. The first byte of each turns out to be a sensor-support bitmask,
not data, so all four logged a constant 3 or 2. The figures above came from
decoding raw frames by hand. Widths are now known and declared properly.

**Three did nothing and are gone.** `F416`, `F419` and `F41A` return `FFFF`,
not-available, on every sample. The ECM's bitmask claims them; the values do
not exist.

**One did its whole job in a single reply and is also gone.** `F413`
returned `0x77`: bank 1 sensors 1, 2 and 3 and bank 2 sensors 1, 2 and 3 are
present. That can never change, so it is recorded here instead of polled.

Two still needed samples at the time: `F456` (three samples, all near
zero) and `F458` (one sample, `7F`, meaning unknown). `F458` is now
resolved — it returns a `7F` negative response consistently and has been
deleted; see "Nineteen commands deleted" under Undecoded. `F456` stayed in
at `freq` 120.

`F470` was read against the SAE J1979 definition of PID 70: its ten bytes
are commanded boost and measured boost for two channels, at 1/32 kPa. The
support byte is `0x02`, flagging channel A as present, and channel A read
24.72 kPa at idle. At the time this looked like the first direct boost
measurement on this truck. A wider check on 2026-09-20 against 525 paired
samples found channel A tracking Manifold Pressure to within 1.3 kPa mean
(r = 0.85) — it is the same manifold sensor at finer resolution, not a
separate boost sensor. See "Duplicates found and removed" above. Boost is
still only available by subtracting Barometric Pressure from Manifold
Pressure, as below.

Checking the rest of the signalset the same way found nothing else. Against
the canonical SAE signalset, the only other unread fields in replies we
already make are sensor-present bits and channels this engine does not have:
`F468` carries six intake air temperature slots and answers `FF` on four,
`F466` and `F467` expose support bits for sensors we already read, and
`F408`/`F409` reserve a second byte for bank 4, which a V6 does not have.

Barometric pressure moved from `freq` 1 to `freq` 5 to pay for this. It does
not change at 1 Hz, and it was consuming 0.9 req/s of a budget with no slack.

### From 89 commands back down to 66

The narrative above tracks the file growing to 89 commands at 10.93 req/s
over the course of 2026-09-20. Later the same day, session 2525 showed
that budget had been measured against the wrong ceiling — 11 req/s that
was never really available, against a true UDS share of about 4.2 req/s —
and that four more ECUs had been permanently retired in the meantime. The
response was the nineteen deletions listed under "Undecoded" (dead across
all logged history, not one drive), the duplicate signals resolved under
"Duplicates found and removed" (PID 67 sensor 1 deleted outright, three of
PID 70's four channels dropped as dead alongside it), and a re-tiering of
everything that survived. The result is the 66-command, 3.922 req/s budget
at the top of this section.

### Only signals with a metric are ever recorded

This is the most consequential thing measured on 2026-09-20, and it sets the
boundary on what this whole project can deliver.

**The app writes a signal to its database only if that signal carries a
`suggestedMetric`. Everything else is requested, answered, decoded and
thrown away.**

Across fifteen months of backups, from 2025-06-29 to 2026-09-20, the signal
database has held exactly 15 distinct signals. Not 84, not the 98 the file
carried at its peak. Fifteen.

The 137-minute drive on 2026-09-20 recorded 14, and they are precisely the
metric-carrying signals whose commands were being polled. The signalset held
16 metrics that day. The two missing from the database are `engineSpeed` and
`absoluteEngineLoad`, and they are missing for the unrelated reason that
their commands, `F40C` and `F443`, were among the 61 never sent. The match
is exact in both directions: every metric-carrying polled signal was stored,
and no signal without a metric ever was.

The other two stores in the backup are not alternatives. `records` holds
manual service and fuel-up entries and is empty. `tripLogger` holds the
phone's own CoreLocation trace — latitude, longitude, altitude, speed — with
no OBD content in it.

What that costs, stated plainly: the metric enum has no slot for manifold
pressure, ride height, suspension pressure, corner pressure, charge air
temperature, barometric pressure, ambient temperature, rear differential
temperature or boost. **None of those can ever be looked at historically
through the app.** They exist live, or in the raw scan logs, and nowhere
else.

That includes the suspension work in its entirety. Even once the retired
`7D3` module is polled again, no ride height and no corner pressure will be
recorded as history. It includes the charge cooler coolant temperature newly
decoded from PID 67 sensor 2 — a real signal, genuinely new, that will never
persist.

One limit on the claim: what was measured is that these signals are never
*stored*. Whether the app renders them live is a separate question the logs
cannot answer. This repo has always assumed that an `F4xx`-aliased signal
does show up live in the app, and nothing here contradicts that; it is still
an assumption, and it is one the owner can confirm from the app in a few
seconds.

The practical rule: a fast `freq` on a signal with no metric buys a live
readout and nothing else. That can be worth paying for. It should be a
deliberate choice rather than an accident.

### Metric slots, seventeen of thirty-six

The schema defines 36 `suggestedMetric` values — the slots Pelican treats as
connectable. Per the section above they are also the only signals that get
recorded at all, which makes this table the boundary of what is knowable
about this truck over time, not just a list of gauges. This signalset fills
17, up from 16 on 2026-09-20 when
`fuelRate` was added as a computed signal (see "Fuel rate, now computed"
above). The remaining 19 break down as:

| Slots | Why they stay empty |
|---|---|
| 10 | Electric and hybrid only: state of charge and health, traction battery, charging, electric range, CVT deterioration |
| 8 | Tire pressure and temperature, which this truck has not yet given up |
| 1 | `fuelRange` is a proprietary Ford DID |

`fuelRate` looked like it belonged in this list too — the schema's only
formula operation is `ratio`, a plain a divided by b with no constant and no
scaling — but the constant can be folded into a hidden operand on one side
of the ratio instead of the ratio itself, which is how it got filled. See
above for the mechanism. Tire pressure is the one still worth chasing: eight
of the nineteen empty slots are tires, and `TESTING.md` records what the
Jaguar signalset does and what has never been tried here.

### Hidden signals

Eleven of the file's 84 signals (including synthetics) carry `hidden: true`:
the remaining `*_RAW` probes and the `3B02` byte splits. Hiding keeps them
from cluttering the app with values nobody can interpret yet. Note that none
of them is recorded either way — none carries a metric, so `hidden` changes
only what is shown, never what is kept. A probe earns its way out of hiding
by being decoded and named.

Eighteen signals carry a `description`, concentrated on the probes and on
the decoded signals with a catch — the inverted height sensors, the bank 2
lambda that rails at `FFFF`, altitude being a barometric lookup rather than
GPS.

### Safety

Service `22` is read-only by definition and cannot change vehicle state. It
is safe to send anything at it.

Service `10 03` (extended diagnostic session) does change ECU state. It's a
standard service that auto-reverts after a few seconds, but run it parked —
some modules reduce normal messaging while an extended session is open.

Avoid entirely unless you know exactly what you're doing: `2F` (actuator
control), `31` (routine control), `2E` (write), `11` (ECU reset), `14` (clear
codes), `27` (security access). On this vehicle `2F` and `31` sent to the
suspension module can physically raise or lower the truck, and `14` erases
emissions readiness monitors, which means failing inspection until a full
drive cycle completes.
