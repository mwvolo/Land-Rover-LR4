# Land Rover LR4 (L319) OBDb Signalset

This repo defines the OBD-II/UDS signalset for a 2016 Land Rover LR4 /
Discovery 4 (L319) — 3.0L supercharged V6 (AJ126), ZF 8HP70, full-time 4WD,
electronic air suspension. It follows the [OBDb](https://github.com/OBDb)
v3 signalset format and is consumed by the Sidecar/Pelican OBD-II app.

For signal-level detail (what each DID means, how to read boost, altitude
behavior, off-road usage, module identification) and the ongoing findings
log, read `README.md` — it is the primary reference and this file does not
repeat it.

## Layout

- `signalsets/v3/default.json` — the signalset itself, 109 commands. The
  only file the app actually consumes.
- `tests/test_cases/2016/command_support.yaml` — manifest of which commands
  each ECU supports, used by the test suite.
- `README.md` — signal reference, polling budget, and the findings log.

## Architecture facts

- The vehicle is **11-bit CAN** throughout. Every module answers on a
  `0x7xx` request header with its response at `request + 8` (e.g. `7E0` →
  `7E8`). No 29-bit/extended addressing anywhere on this truck.
- **The `F4xx` convention.** `22 F4<nn>` returns standard OBD-II PID `01<nn>`
  with standard scaling, as a UDS request instead of a mode-01 request. This
  exists because Pelican consumes standard mode-01 PIDs internally and they
  never surface as recordable signals in the app, whereas the same data
  requested via the `F4xx` alias arrives as a custom signal and does show
  up. Most of the signalset uses this trick rather than requesting the
  standard PID directly.
- **Per-ECU DID offsets.** The engine's proprietary DIDs sit `+0x300` from
  their base (e.g. `00F2` → `03F2`). The transmission and rear differential
  sit `+0x1000` (e.g. `0E69` → `1E69`). A DID that comes back "out of range"
  is often just addressed at the wrong offset for that module.
- **`freq` is an interval in seconds, not a rate.** A lower number polls
  harder. Total demand is `sum(1/freq)` across all commands and must stay
  under roughly 11 req/s, which is what the adapter actually delivers.
  Current demand is 10.99 — there is almost no headroom. Adding a command
  means trading it against a deletion or giving it a slow cadence (120s or
  600s).

## Validation

CI (`.github/workflows/presubmits.yml`) validates `signalsets/v3/` against
the schema in the separate `OBDb/.schemas` repo, which is not checked into
this repo. To validate locally:

```
curl -sfL https://raw.githubusercontent.com/OBDb/.schemas/main/signals.json -o /tmp/signals.schema.json
python3 -c "
import json, jsonschema
schema=json.load(open('/tmp/signals.schema.json'))
inst=json.load(open('signalsets/v3/default.json'))
errs=list(jsonschema.Draft7Validator(schema).iter_errors(inst))
print('errors:', len(errs))
for e in errs[:5]: print(list(e.path), e.message[:200])
"
```

`path` is constrained by the schema to a fixed enum (`Engine`, `Fuel`,
`Drivetrain`, `Suspension`, `Transmission`, `Trips`, `ECU`, `Electrical`,
`Tires`, `Battery`, `Climate`, `Movement`, and a few others) — `Diagnostics`
is not a valid value; use `ECU` for undecoded/diagnostic raw signals.

Print total polling demand:

```
python3 -c "
import json
d=json.load(open('signalsets/v3/default.json'))
print(sum(1/c['freq'] for c in d['commands']))
"
```

Check that the test manifest YAML parses and that no entry silently became
a float or other non-string type (see trap 1 below):

```
python3 -c "
import yaml
d=yaml.safe_load(open('tests/test_cases/2016/command_support.yaml'))
for ecu, entries in d['supported_commands_by_ecu'].items():
    for e in entries:
        if not isinstance(e, str):
            print('NON-STRING ENTRY:', ecu, repr(e))
print('ok')
"
```

The pytest workflow (`.github/workflows/response_tests.yml`) runs inside the
`ghcr.io/obdb/devcontainer` image, and the test sources it runs live inside
that image, not in this checkout. Running `pytest tests/` locally will
report "no tests ran" — that is expected, not a failure. There is no way to
run the actual response tests outside CI.

## Traps

These have all actually bitten:

1. **YAML float trap.** In `command_support.yaml`, a plain-scalar entry made
   up only of digits and dots (e.g. `726.220202`) parses as a YAML float,
   not a string, unless it's quoted. Any entry with no hex letter A–F
   anywhere in it must be double-quoted. After editing this file, verify no
   entry parsed as a non-string (see the validation command above).
2. **Deleting a command can orphan a synthetic.** The top-level `synthetics`
   block in `default.json` references signal ids by name (e.g.
   `LR4_BOOST_RATIO` reads `LR4_MAP` and `LR4_BARO`). Removing the command
   that defines one of those signal ids leaves a dangling reference that is
   still valid JSON but semantically broken — the synthetic will never
   compute. Grep `synthetics` for a signal id before deleting the command
   that defines it.
3. **Entry shape differs per ECU in `command_support.yaml`.** Most ECUs use
   `<hdr>.22<DID>` (e.g. `795.221E89`). `7E1` in the `supported_commands_by_ecu`
   section uses `<hdr>.<rax>.22<DID>` (e.g. `7E1.7E9.221E69`). There is no
   reason for the difference — `7E1`'s `rax` is `7E9`, exactly `hdr+8`, the
   same as everywhere else. It is just how that block was written. Match
   whatever shape the surrounding block already uses; don't normalize
   across ECUs.
4. **Agent worktrees branch from `main`, not from the current branch.** Work
   that depends on unmerged commits on the current branch must be done in
   the main checkout, or the agent must be told explicitly to check out that
   branch — otherwise it silently operates on stale files and produces
   confident, wrong results.
5. **Eight commands have not been polled since 2026-08-30** — `F40C`,
   `F411`, `F40E`, `F443`, `F449`, `F44A`, `F407`, `033E` — despite being
   present in the signalset with sane `freq` values and the request budget
   having headroom at the time. Engine speed has therefore never actually
   been recorded as a signal, even though the command is in the file and
   the underlying standard PID is being polled constantly. Cause unknown;
   see the "Eight commands stopped being polled" section in `README.md`.
   **A command's presence in this file does not mean it is actually being
   collected** — check the scan logs before relying on one.

## Where the data comes from

On macOS, the Sidecar/Pelican app writes zipped SQLite backups to
`~/Library/Mobile Documents/iCloud~com~featherless~apps~electricsidecar/DataBackups/`,
in four folders:

- `scanSessions` — every raw request/response pair (large, on the order of
  a gigabyte or more). This is the ground truth for whether a command
  answers and what it actually returns.
- `signals` — decoded signal values.
- `tripLogger` — trip-level data.
- `records` — other recorded data.

The backups lag live data by up to a day, so a command that looks dead in
the backups may have started responding again since the last sync.
