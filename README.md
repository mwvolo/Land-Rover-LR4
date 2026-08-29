# Land Rover LR4 (L319) — Signal Reference

OBDb signalset for the 2016 Land Rover LR4 / Discovery 4 — 3.0L supercharged
V6 (AJ126), ZF 8HP70, full-time 4WD with locking rear differential and
electronic air suspension.

Signals here were confirmed against ~13M logged request/response pairs from
this vehicle (VIN `SALAK2V62GA832217`). Anything still named `*_RAW` responds
on the bus but hasn't been decoded yet.

---

## Modules

Seven modules answer on this truck. Requests go to `hdr`, replies come back
from `rax`.

| Request | Response | Module |
|---|---|---|
| `7E0` | `7E8` | Engine (ECM) |
| `7E1` | `7E9` | Transmission (TCM) |
| `732` | `73A` | Gear selector |
| `761` | `769` | Body / instrument |
| `795` | `79D` | Rear differential |
| `792` | `79A` | Unidentified — answers `2A3x` |
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
| Intake Air Temp | `7E0` `22F40F` | Before the supercharger |
| Charge Air Temp | `7E0` `220520` | **After** the supercharger and intercooler |
| Timing Advance | `7E0` `22F40E` | Degrees before top dead centre |
| Throttle Position | `7E0` `22F411` | Actual plate position |
| Commanded Throttle | `7E0` `22F44C` | What the ECU asked for |
| Accelerator Pedal D / E | `7E0` `22F449` / `22F44A` | Two sensors in one pedal |
| Mass Air Flow | `7E0` `22F410` | Grams of air per second |
| Equivalence Ratio | `7E0` `22F444` | Commanded lambda |
| O2 Lambda B1S1 | `7E0` `22F434` | Measured lambda |
| Catalyst Temp B1S1 / B2S1 | `7E0` `22F43C` / `22F43D` | One per bank |
| Oil Temp | `7E0` `2203F3` | Land Rover's own sensor |
| Oil Temp (SAE) | `7E0` `22F45C` | Standard PID, same physical sensor |
| Oil Level | `7E0` `2203E6` | Millimetres in the sump |
| Oil Volume | `7E0` `2203F2` | |

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
| Fuel Rate | `7E0` `22F45E` | Litres per hour |
| Fuel Rail Pressure | `7E0` `22033E` | Proprietary; ~60 bar idle, 130+ under load |
| Fuel Rail Pressure (SAE) | `7E0` `22F423` | Standard PID, same rail |
| Short Term Fuel Trim B1 | `7E0` `22F406` | Immediate correction, swings constantly |
| Long Term Fuel Trim B1 | `7E0` `22F407` | Learned correction, drifts slowly |

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

## Undecoded

These respond on the bus but their meaning isn't established. They're in the
signalset so their values get logged during normal driving, which is how the
formulas eventually get worked out.

| Address | Signals |
|---|---|
| `792` / `79A` | `2A32`–`2A3F` — `2A36`, `2A37`, `2A38`, `2A3A` return data; the rest are a sweep |
| `795` / `79D` | `1E88`, `1E89`, `1E8B` — neighbours of the differential temp |
| `7E1` / `7E9` | `1E68`, `1E6A`, `1E6B`, `1E70` — neighbours of the gearbox temp |
| `761` / `769` | `197C`, `D11C` |
| `7D3` / `7DB` | `3B07` — reads lower than the four corner pressures |

**Tire pressures remain unsolved.** Module `751` returns no data to any
request. Both Toyota-Tacoma and Jeep-Grand-Cherokee reach their tire modules
through addressing modes never tried here — Tacoma via ISO-TP extended
addressing (`eax`/`tst`/`fcm1`), Jeep via a 29-bit header (`DAC7`). The
extended-address variant is in this signalset as `LR4_TPMS_EAX2A`.

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
