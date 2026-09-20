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
| Ride Height Mode | `7D3` `223B3C` | 1 normal, 2 raised — both confirmed. Two more values, `0D` and `04`, turned up during a ride-height change on 2026-09-20 and aren't explained yet |
| Height Sensor Front / Rear | `7D3` `223B71` / `223B72` | **Inverted** — falls as the truck rises |
| Module Voltage | `7D3` `22D11A` | Should mirror battery voltage |

Normal standing pressures are roughly 40 psi per corner at normal ride
height, rising with load and with raised height modes.

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
| Oil Temp vs Oil Temp (SAE) | A few degrees |
| Fuel Rail Pressure vs SAE equivalent | Similar magnitude at steady idle |
| Accelerator Pedal D vs E | A percent or two — they're redundant by design |

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

The signalset defines eight synthetic signals. Each is a **ratio between two
readings that should hold a known value**, which makes them suited to a
display: you learn the normal number once, and anything else is a signal.

| Signal | Ratio | Normal | Meaning when it moves |
|---|---|---|---|
| Boost Pressure Ratio | MAP / Barometric | 1.0 idle, up to ~1.8 | Above 1.0 is supercharger boost, self-correcting for altitude |
| Lambda Tracking | Measured / Commanded lambda | 1.0 | Fuelling isn't hitting its target — sensor, leak, or delivery |
| Bank Balance | Cat temp B1 / B2 | 1.0 | One bank working harder — misfire or injector on the low side |
| Pedal Agreement | Pedal D / Pedal E | 1.0 | Redundant pedal sensors disagreeing; the ECU limps if they diverge |
| Throttle Tracking | Actual / Commanded throttle | 1.0 | Plate not following orders — sticky or carbonned throttle body |
| Rail Pressure Crosscheck | Proprietary / SAE rail pressure | steady | Drift means one sensor path is wrong |
| Suspension Balance Front | Front left / right pressure | 1.0 | A corner losing air, before the dash warns |
| Suspension Balance Rear | Rear left / right pressure | 1.0 | Same, rear axle |

Six of the eight sit at **1.0 when healthy**, so a single glance covers
fuelling, ignition balance, pedal and throttle integrity, and air springs.

Boost Pressure Ratio is the exception and the one to watch for fun: it's the
closest thing to a boost gauge this vehicle exposes. The schema's only
operation is division, so true gauge boost (`MAP − Barometric`) isn't
expressible as a synthetic — but the ratio carries the same information and
needs no altitude correction.

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

A drive on 2026-09-20 that changed ride height confirmed Ride Height, Ride
Height Front/Rear and the corner pressures all move together and in the
directions their labels predict — see "Air suspension notes" below. `3B4D`,
once labelled Drive Mode, is not: it read `0` on all 48 samples that drive,
straight through every height change. Terrain Response mode itself is still
untested, since the drive changed height rather than terrain setting — if a
future drive cycles through Terrain Response settings, watch `3B4D` and note
what it reads in each one.

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
| `7E0` / `7E8` | `F416`, `F419`, `F41A`, `F458`, `F466`, `F467`, `F468`, `F470` — eight single-byte probes added 2026-09-20. The ECM's own supported-PID bitmask says these PIDs exist; nothing more is known |
| `792` / `79A` | `2A32`–`2A3A` — eight live values, none resembling tire pressures |
| `795` / `79D` | `1E88`, `1E89` |
| `7E1` / `7E9` | `1E68`, `1E6A` — neighbours of the gearbox temp |
| `761` / `769` | `197C`, `D11C` |
| `726` / `72E` | `0202` — returns `00` |
| `7D3` / `7DB` | `3B00`, `3B01`, `3B02`, `3B08`, `3B0B` — `3B02` answers four single-byte values that look like one per corner |

`726`'s `0202` came out of a sweep of DIDs `0000`–`03FF` against that module
(916 requests) — the one hit, and the first non-identification DID ever
found there.

**The suspension decodes check out.** Replaying every logged sample through
the formats in this file gives corner pressures of 34-50 psi, a ride height
offset correctly signed at -91 to +50 mm, and module voltage of 9.12-14.56 V,
all physically sensible. The exception is `3B4D`: it was labelled Drive Mode,
but it returned `0` on all 171 samples recorded before 2026-09-20 and on all
48 samples taken on a 2026-09-20 drive that changed ride height twice — flat
straight through the one state change it was tested against. The signal has
been renamed `LR4_3B4D_RAW`. A terrain-mode change hasn't been tested yet, so
the field is unidentified rather than proven useless.

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
16:23:49. The state fields did not obviously distinguish it:

`3B3C` is the height mode and increments upward: mode 1 was normal and mode 2
was raised, both confirmed before this drive. The map also labels 0 as Access
and 3 as Extended on the assumption the ordering continues. On the 2026-09-20
drive it also returned `0D` and `04`, neither of which fits that scheme, and
none of the four values it returned cleanly picked out the access-height
moment. Correct the 0/3 labels if they read wrong.

`3B01` returned three distinct values on the same drive — `00000400`,
`00000100`, `00000800` — each a single bit set, which looks like a ride-height
state word rather than an enum. It was at `freq` 600 and caught only 3
samples that drive, so it's been raised to `freq` 10 for another look.

Most likely, `3B01` and `3B3C` simply weren't sampled often enough (`freq` 600
and 10) to catch the access-height moment, which is why the corner pressures
and height sensors saw it and the state fields didn't.

The two balance ratios respond to cargo, not just faults. With the load area
full, front balance read 0.99 and rear read 0.92 — the rear axle carrying more
on one side. Check the ratios unloaded before reading a low number as a leak.

---

## Polling

The adapter sustains about **13 request/response round trips per second**,
measured across many hours of logging. That is the ELM327's ceiling, not the
CAN bus — 13 requests per second is negligible against 500 kbit/s, and
diagnostic identifiers in the `7xx` range are low priority by CAN arbitration,
so they yield to powertrain traffic automatically. Service `22` is read-only.
Polling cannot harm anything; it can only compete with itself.

**`freq` is a minimum interval in seconds, not a rate.** Pelican documents it as
the maximum frequency at which a command may be sent, expressed in seconds, so
a *smaller* number polls *harder*. There is no priority field anywhere in the
v3 format — command count and `freq` are the only levers.

The budget in this file:

| Interval | Commands | Contents |
|---|---|---|
| 1s | 4 | Manifold pressure, barometric, engine speed, vehicle speed |
| 3s | 8 | Throttle, pedals, timing, lambda, load |
| 5s | 10 | Temperatures, fuel trims bank 1, mass air flow, gearbox and diff temp, gear selector |
| 10s | 14 | Air suspension, fuel system status B1/B2, O2 lambda/voltage B2S1 |
| 20s | 2 | Commanded evap purge, O2 voltage/trim B1S2 |
| 30s | 19 | Fuel trims bank 2, catalyst temperatures, battery, fuel level, and the undecoded probes shown to move |
| 60s | 4 | `726/0202`, long term secondary O2 trim B1, relative throttle position, absolute throttle position B |
| 300s | 8 | Eight raw single-byte probes from the ECM's own supported-PID bitmask, payload width unknown |
| 600s | 20 | Odometer, oil level, oil volume, O2 sensors present, and undecoded probes that have never moved |

That totals 10.93 requests per second of demand against roughly 11 available
once protocol overhead is removed — 89 commands, up from 73 after the
sixteen added from the ECM's own supported-PID bitmask (see below). The four
one-second commands exist so the boost calculation stays responsive.

### Eight commands went quiet for three weeks. The app had a stale signalset

Between 2026-08-30 and 2026-09-19, eight commands in this file were never
requested on a drive: `F40C`, `F411`, `F40E`, `F443`, `F449`, `F44A`, `F407`
and `033E`. The cost was real — `engineSpeed` is wired to `F40C`, and no
engine speed value reached the signal database in fourteen months. Two
synthetics could not compute either, `LR4_THROTTLE_TRACKING` needing `F411`
and `LR4_PEDAL_AGREEMENT` needing `F449` and `F44A`.

**Resolved on 2026-09-20.** Every command in the file was polled that drive.
`F40C` was requested 172 times and answered 172 times, and engine speed is
now recorded. Nothing in this file changed to cause that — the app had
simply been running an older copy of the signalset, and picked up the
current one.

Getting there meant ruling out the two obvious causes, and both remain worth
knowing. It was not the request budget: demand was 10.21 req/s against the
11-13 the adapter delivers. It was not the `freq` values either, because the
dead commands shared tiers with live ones — `F40C` and `F40D` are both
`freq` 1 and only `F40D` ran; `F411`, `F443`, `F449` and `F44A` sat at
`freq` 3 alongside `F404`, `F434` and `F444`, which all ran.

The lesson that outlives the incident: **a command's presence in this file
means nothing until the app has actually fetched the file.** Before
concluding that a command is unsupported, confirm the app is holding the
version you think it is.

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

### Sixteen more, found by reading the ECM's own PID list

Every request in this file so far came from watching the bus and guessing.
On 2026-09-20 the ECM's own mode-01 replies to `0100`, `0120`, `0140` and
`0160` were decoded instead — these are bitmasks of which mode-01 PIDs the
module actually supports, and this ECM claims 54 of them. Cross-referencing
those 54 against this file found 25 with no entry. After excluding metadata
PIDs (the support bitmasks themselves, monitor status, OBD standard, fuel
type) and the two SAE aliases already dropped as duplicates (`F423`, `F45C`
— see Fuel and Engine above), sixteen were worth adding.

Eight got real decodes:

| Command | PID | Signal | Why it was worth adding |
|---|---|---|---|
| `F403` | `03` | Fuel System Status B1 / B2 | Open loop vs. closed loop — fuel trims only mean something in closed loop |
| `F438` | `38` | O2 Lambda B2S1 / O2 Voltage B2S1 | Bank 1 lambda has been read via `F434` all along; there was no bank 2 equivalent |
| `F42E` | `2E` | Commanded Evap Purge | A stuck-open purge valve is a classic cause of both banks running lean at once |
| `F415` | `15` | O2 Voltage B1S2 / O2 Trim B1S2 | Post-catalyst sensor, bank 1 |
| `F456` | `56` | Long Term Secondary O2 Trim B1 | Further trim context |
| `F445` | `45` | Relative Throttle Position | Throttle redundancy |
| `F447` | `47` | Absolute Throttle Position B | Throttle redundancy |
| `F413` | `13` | O2 Sensors Present | Bitmask of which O2 sensor positions are physically fitted |

The other eight are raw single-byte probes, because their meaning isn't
established: `F416`, `F419`, `F41A`, `F458`, `F466`, `F467`, `F468`, `F470`.
All that's known is that the ECM claims to support the underlying PIDs.
They're declared at `len` 8 because the payload width is unknown — reading
the first byte is always safe, and each should be widened once a real reply
shows its actual width. See `TESTING.md` for what would count as a result
for each.

This batch was picked to test the lean-trim finding above rather than at
random: `F403` says whether the engine was in closed loop when the +9%/+7%
trims were measured, `F42E` tests the stuck-purge-valve hypothesis directly,
and `F438` gives bank 2's own oxygen sensor behind the bank 2 trim. Together
they move that finding from an observation toward something diagnosable.

The file is now 89 commands at 10.93 req/s.

**Two known inefficiencies**, neither fixable from a signalset:

Requests carry no expected-response count. The ELM327 datasheet documents that
appending a digit (`22F405 1`) lets the adapter return the instant it has that
many responses instead of waiting out the full timeout, and shows it nearly
doubling throughput. No request in this vehicle's history uses it.

Sparse headers cost triple. Reading one value from a module means `ATSH`, then
a receive filter, then the read — three round trips for one number. The engine
module amortises this well at 0.12 setup commands per read; a header carrying a
single command pays 2.0.

Multi-DID requests would help most — a single frame holds seven payload bytes,
enough for `22` plus three identifiers — but the schema caps `cmd` at one
identifier, so the format cannot express it.

---

## Terminal probing

Sidecar's terminal is the fastest way to test a DID.

```
ATSH 7E0        set request address
ATCRA 7E8       set response filter
22F42F          read a DID
ATAR            restore automatic addressing
```

**Always finish with `ATAR`.** A left-over `ATCRA` filter blocks replies from
every other module, which makes the whole app look broken.

Reading a reply — `7E8 04 62 F42F EC` is address, length, positive response,
echoed DID, then data. `7E8 03 7F 22 31` is a negative response: the DID
doesn't exist here. `NO DATA` means nothing answered at all.

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
