# ApexLogger vehicle profiles

This repository serves **apexlogger.com**, including the vehicle profiles the ApexLogger and
ApexOBD apps download at runtime from `https://apexlogger.com/profiles/`.

Profiles describe how to talk to a vehicle: CAN addressing, which OBD-II PIDs to poll, how to
convert each reply into a value, and the alert thresholds. The apps ship a copy of every profile
inside the app, and also check this site, so **a profile can be changed here — for example while
troubleshooting a car — and reach the app without releasing a new build.**

## Layout

```
profiles/
├── profiles.json            index of every profile the apps should fetch
├── generic/mode1.json       fallback for any car (Mode 01 only)
├── bmw/n54/tested.json      <brand>/<engine-or-model>/<status>.json
├── bmw/n54/mode22.json
└── …
```

- **`profiles/profiles.json`** lists every profile. A file that isn't listed here is never
  downloaded.

  ```json
  {
    "schemaVersion": 3,
    "profiles": [
      { "path": "bmw/n54/tested.json", "sha256": "" }
    ]
  }
  ```

  `path` is relative to `profiles/`. `sha256` is optional: when it holds the SHA-256 of the file,
  the app skips re-downloading an unchanged copy; leave it `""` to always re-check.
- Inside the app, each profile is stored under its path with `/` replaced by `_`
  (`bmw/n54/tested.json` → `bmw_n54_tested.json`).

## When does a downloaded profile take effect?

Every profile has an integer **`revision`** (missing = 0).

- A profile whose `id` the app doesn't ship is always added.
- A profile whose `id` the app **does** ship replaces the built-in copy **only if its `revision`
  is higher** than the built-in one.

So:

- **Live troubleshooting change:** edit the file here and **increase `revision`** above the
  version shipped in the app. The app downloads changed profiles in the background at launch;
  the change takes effect when you tap the profile-update prompt in Settings, tap ↻ in the
  vehicle-profile picker (downloads and applies at once), or on the next launch. The first launch
  after installing a new app build skips the download once, so a fresh build starts from its own
  profiles.
- **Shipping a change in an app release:** increase `revision` in the app's copy and publish the
  same file, with the same `revision`, here. A newer app build always wins over an older copy
  on this site.

Editing a file here **without** raising `revision` changes nothing for profiles the app already
ships.

## Profile file format

```json
{
  "id": "bmw_n54",
  "name": "N54 MODE1 · 3.0L Twin-Turbo I6",
  "shortName": "N54",
  "revision": 1,
  "status": "tested",
  "cylinderCount": 6,
  "firingOrder": [1, 5, 3, 6, 2, 4],
  "isTurbo": true,
  "displacementL": 2.979,
  "vinPrefixes": ["WBA", "WBS", "WBX", "WBN"],
  "tcm": { "request": "7E3", "response": "7EB" },
  "canConfig": { "protocol": "6", "txHeader": "7E0", "rxHeader": "7E8" },
  "manufacturer": { "name": "BMW", "ecu": "MSD80", "mode": "22" },
  "polling": { "lowLatencyMs": 200, "highLatencyMs": 3000, "batchMode01": true },
  "sensors": {
    "speed":    { "pid": "010D", "formula": "A_LINEAR", "unit": "kmh", "latency": "low" },
    "coolantTemp": {
      "pid": "0105", "formula": "A_LINEAR", "offset": -40, "unit": "C", "latency": "high",
      "optimalF": 207, "warnF": 230, "criticalF": 248
    },
    "boostPressure": { "warnPSI": 18, "criticalPSI": 22 }
  }
}
```

### Top-level fields

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Unique, stable identifier. Must match the app's copy for a download to replace it. |
| `name`, `shortName` | yes | Full and compact display names. |
| `revision` | no (0) | Content version; see *When does a downloaded profile take effect?* |
| `status` | no (`testing`) | `tested`, `testing` or `custom`. |
| `cylinderCount`, `firingOrder`, `isTurbo` | yes | Engine layout. |
| `displacementL` | no | Litres. Only used to choose between profiles that share `vinPrefixes`. |
| `vinPrefixes` | no (`[]`) | VIN prefixes (WMI) used to pick this profile automatically on connect. |
| `tcm` | no | Transmission module `request`/`response` CAN ids. |
| `canConfig` | no | Adapter setup: OBD `protocol` (`ATSP` value), `txHeader` (`ATSH`), `rxHeader` (`ATCRA`). |
| `manufacturer` | no | `name`, `ecu`, and `mode`: the service byte prepended to manufacturer PIDs (e.g. `"22"`). |
| `polling` | no | `lowLatencyMs` (default 350), `highLatencyMs` (default 3000), `batchMode01` (default `true`; set `false` for ECUs that drop PIDs from combined Mode 01 requests). |
| `sensors` | no | Map of sensor key → sensor definition (below). |

### Sensor definition

| Field | Default | Meaning |
|---|---|---|
| `pid` | — | Hex PID. Standard Mode 01 PIDs include the mode (`"010D"`); manufacturer PIDs list only the register (`"1151"`) and get `manufacturer.mode` prepended. Omit for a threshold-only entry. |
| `formula` | — | How to read the reply bytes (A = first data byte, B = second): `A_LINEAR` = A × scale + offset; `AB_LINEAR` = (A×256+B) × scale + offset; `A_PERCENT` = A × 100 / 255; `AB_LAMBDA` = (A×256+B) / 32768. |
| `scale`, `offset` | 1.0, 0.0 | Applied by the linear formulas. |
| `unit` | `""` | e.g. `C`, `PSI`, `kPa`, `kmh`, `pct`, `V`, `lambda`, `deg`, `rpm`. |
| `latency` | `high` | `low` or `high` polling tier (see `polling`). |
| `visible` | `true` | Whether a gauge is shown. Hidden sensors are still polled and still drive alerts. |
| `tolerateResponsePending` | `false` | Keep waiting through ISO 14229 `7F xx 78` "response pending" replies. |
| `optimalF`, `warnF`, `criticalF` | — | Temperature thresholds (°F). |
| `warnPSI`, `criticalPSI` | — | Pressure thresholds. |
| `warnDeg`, `criticalDeg` | — | Angle thresholds. |
| `heatSoakDeltaF` | — | Heat-soak alert delta (°F). |
| `overboostGraceSec`, `overboostGraceTPS` | — | Hold the critical boost alert this many seconds while throttle ≥ this %. Both required. |
| `cylinders` | — | `timingCorrections` only: `[{ "pid": "1101", "index": 1 }, …]`, one per cylinder. |
| `blockFields` | — | Turns the sensor into a **block sensor** (below). |

**Sensor keys** the apps understand: `speed`, `rpm`, `throttle`, `engineLoad`,
`manifoldPressure` (or `manifoldAbsolutePressure`), `barometricPressure`, `boostPressure`,
`coolantTemp`, `intakeAirTemp`, `oilTemp`, `oilTempMode1`, `timingAdvance`, `lambda`,
`batteryVoltage`, `transmissionTemp` (or `transmissionOilTemp`), `highPressureFuelPump`,
`lowPressureFuelPump`, `fuelRailPressure`, `timingCorrections`, and the block sensors `did4000`,
`did2000`, `didC000`.

### Block sensors (`blockFields`)

One request whose multi-frame reply is split into many named values by byte position. Byte
offsets count from the start of the full reply, so for `22 40 00` the echo `62 40 00` is bytes
0–2 and data starts at byte 3.

```json
"did4000": {
  "pid": "4000", "latency": "low", "visible": false, "tolerateResponsePending": true,
  "blockFields": [
    { "id": "rpm", "byteOffset": 10, "byteWidth": 2, "unit": "rpm", "dispatchId": "rpm" },
    { "id": "candidate_20_timing", "byteOffset": 20, "byteWidth": 1,
      "scale": 0.375, "offset": -35.625, "unit": "deg", "visible": false },
    { "id": "word_05_06", "byteOffset": 5, "byteWidth": 2, "signed": true, "visible": false }
  ]
}
```

| Field | Default | Meaning |
|---|---|---|
| `id` | — | Name of the value (also its column in diagnostic log exports). |
| `byteOffset`, `byteWidth` | — | Position and length (1–3 bytes, big-endian). |
| `signed` | `false` | Read as two's complement. |
| `scale`, `offset` | 1.0, 0.0 | value = raw × scale + offset. |
| `unit` | `raw` | Display unit. |
| `visible` | `true` | Whether it's shown as its own value. |
| `dispatchId` | — | Also feed this value into a normal gauge (e.g. `rpm`, `speed`, `coolantTemp`). Only for confirmed formulas. |

## Editing tips

- Keep the JSON valid — a file that fails to parse is ignored, and the app keeps what it had.
- Add new profiles to `profiles/profiles.json`, or they won't be fetched.
- `profile-editor.html` on this site can help build a profile.
