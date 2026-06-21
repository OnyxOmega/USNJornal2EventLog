# EvtxECmd Maps — USN Journal Monitor → Timeline Explorer / ELE / Velociraptor

These maps let [EvtxECmd](https://ericzimmerman.github.io/) parse this tool's
`FileSystem` events into clean, named columns for **Timeline Explorer** and
**Event Log Explorer**, so USN file-change and removable-device events can be
correlated alongside Sysmon (and other) sources in one timeline. **Velociraptor**
reads the same events natively (see below).

## Files (one map per Event ID — 21 total)

| File | Event | Category | Shape |
|------|:-----:|----------|-------|
| `..._100.map` | 100 | File Create | file |
| `..._101.map` | 101 | File Modify | file |
| `..._102.map` | 102 | File Delete | file |
| `..._103.map` | 103 | File Rename | file |
| `..._104.map` | 104 | Security Change | file |
| `..._105.map` | 105 | Other Change | file |
| `..._106.map` | 106 | Range Change (V4) — **RESERVED, never emitted** | file |
| `..._500.map` | 500 | **Device Connected (NEW)** | device |
| `..._501.map` | 501 | **Device Removed** | device |
| `..._503.map` | 503 | **Known Device Re-attached** | device |
| `..._504.map` | 504 | **Known Device ALTERED** (VSN changed) | device |
| `..._904.map` | 904 | Archive Written (rotation complete) | operational |
| `..._914.map` | 914 | Engine Started | operational |
| `..._915.map` | 915 | Engine Stopped | operational |
| `..._916.map` | 916 | Drive/Journal Failure | operational |
| `..._918.map` | 918 | Archive Completeness Gap | operational |
| `..._919.map` | 919 | Volume Present, No Active USN Journal | operational |
| `..._920.map` | 920 | Volume Present, Unsupported Filesystem | operational |
| `..._921.map` | 921 | Volume Present, Remote/Network Share | operational |
| `..._922.map` | 922 | Archive Sign/Hash Failure | operational |
| `..._923.map` | 923 | Resume Gap (cursor) | operational |

(All files are named `FileSystem_USNJournalMonitor_<EventID>.map`. For the full
meaning of every Event ID — including the 800-series test markers that have no maps —
see [`EVENT_ID.md`](EVENT_ID.md).)

## Install / parse

Copy all `*.map` files into EvtxECmd's `Maps` folder, then:

```
EvtxECmd.exe -f "C:\FileSystem_Archives\FileSystem_2026-06_1.evtx" --csv "C:\out" --csvf usn.csv
```

(Use `-d <folder>` to process all archives at once.) Open the CSV in Timeline Explorer.

## Three event shapes

This log carries three structurally different event families. The maps reflect that.

### 1. File events (100–105; 106 RESERVED) — full field array

Emitted via the engine's `emit_event` path: one readable blob in `Data[1]` plus the
broken-out fields in `Data[2..21]`.

> **106 is RESERVED and never emitted.** The engine reads V0/V2 journal records and
> does not request the V4 range records that 106 would represent (V4 carries only
> changed byte-ranges, no content/before/after — no forensic value beyond the Modify
> it accompanies). The `..._106.map` file is retained, valid and ready, in case a
> future non-forensic use enables V4. It will simply never match any event today.

| Column | Field(s) (Data[N]) |
|--------|--------------------|
| PayloadData1 | TargetFilename `Data[2]` — primary correlation key |
| PayloadData2 | UtcTime `Data[3]` |
| PayloadData3 | MachineGuid `Data[4]` |
| PayloadData4 | HWID `Data[5]` + VSN `Data[6]` (blank on file events) |
| PayloadData5 | VolumeSerial `Data[7]` + Hostname `Data[8]` |
| PayloadData6 | Category `Data[11]` + Reason `Data[12]` + Usn `Data[13]` + JournalId `Data[14]` |

### 2. Device events (500/501/503/504) — full field array

Same `emit_event` path; the device-identity fields carry the values, VolumeSerial is
blank (this is a device event, not a file event).

| Column | Field(s) |
|--------|----------|
| PayloadData1 | Drive `Data[2]` |
| PayloadData2 | UtcTime `Data[3]` |
| PayloadData3 | MachineGuid `Data[4]` |
| PayloadData4 | HWID `Data[5]` |
| PayloadData5 | VSN `Data[6]` — **504 also packs PrevVSN `Data[21]`** (reformat/tamper delta) |
| PayloadData6 | Category `Data[11]` + Hostname `Data[8]` |

`501` (Removed) carries the **last-known** HWID/VSN of the drive that left, looked up
from device state — so a detach records *which* device, not just a bare letter.

### 3. Operational events (904, 914–916, 918–923) — single message string

**Operational 9xx events are structurally different.** They are emitted via
`emit_operational`, which writes **one insertion string only** — a human-readable
status message in `Data[1]`. There are **no** `Data[2..]` key/value fields on these
events. The maps therefore surface exactly one column:

| Column | Field |
|--------|-------|
| PayloadData1 | Message `Data[1]` — the full operational status text |

Time and host are read from EvtxECmd's **native** `TimeCreated` and `Computer`
columns (no need to duplicate them). 9xx events describe the **state of the
monitoring host itself** — a volume that can't be journaled, a rotation completing, a
resume gap, an engine start/stop. They are acted on at the machine (locally or via
remote-in), not parsed for correlation, so a single message column is sufficient and
correct.

> **Note for anyone upgrading older maps:** earlier `918`/`922` maps referenced
> `Data[2..12]` as if operational events carried the full field array. They do not —
> those references resolve to null. The `918`/`922` maps in this set are corrected to
> the single-`Data[1]` shape.

## SourceIP and other blob-only fields

`SourceIP` is intentionally NOT carried into a column (it remains in the evtx
EventData / Payload for ELE and Velociraptor). Everything not given a column on file
and device events (SchemaVersion, FQDN, Domain, MachineSID, MAC, OSBuild, MachineId,
SourceIP, and the test-only HiResUtc/Qpc) remains in the full **Payload**. Hostname is
also in EvtxECmd's native **Computer** column.

`HiResUtc` and `Qpc` (appended to the blob in schema 2.1 at `Data[22]`/`Data[23]`) are
sub-millisecond test-instrumentation fields, not investigation fields. They are
deliberately **not mapped** to a column — they remain in the Payload for anyone who
needs them.

## How the fields are sourced (positional `Data[N]`)

Classic `ReportEvent` on a custom channel emits **one `<Data>` node per insertion
string**. File and device events front-load eight join/identity fields so two tools
can address the SAME strings two ways:

- **EvtxECmd / Timeline Explorer:** 1-based, `Data[1]` = the readable `Key: value`
  blob, `Data[2..]` = fields. The `.map` files use `/Event/EventData/Data[N]`.
- **Event Log Explorer (ELE):** 0-based, `PARAM[0]` = the blob, `PARAM[1..]` = fields.
  ELE custom columns only reliably reach **PARAM[1]..PARAM[6]**, so the six
  join-critical fields are front-loaded into those slots.

The emission order for file/device events (= the contract, matches
`EVENT_FIELD_NAMES` in `usn_common.py`):

```
Data[1]/PARAM[0]  = blob (Key: value; also the Payload fallback)
Data[2]/PARAM[1]  = TargetFilename   <- ELE-reachable join field
Data[3]/PARAM[2]  = UtcTime          <- ELE-reachable
Data[4]/PARAM[3]  = MachineGuid      <- ELE-reachable
Data[5]/PARAM[4]  = HwSerial (HWID)  <- ELE-reachable
Data[6]/PARAM[5]  = Vsn              <- ELE-reachable
Data[7]/PARAM[6]  = VolumeSerial     <- ELE-reachable (last reliable ELE slot)
Data[8]/PARAM[7]  = Hostname         <- emitted; also in native Computer column
Data[9]/PARAM[8]  = SourceIP         <- emitted; blob-only
Data[10]+         = SchemaVersion, Category, Reason, Usn, JournalId, FQDN, Domain,
                    MachineSID, MAC, OSBuild, MachineId, PreviousVsn(21)
Data[22..23]      = HiResUtc, Qpc    <- schema 2.1, test-only, unmapped
```

Operational 9xx events do **not** follow this layout — they carry only `Data[1]`.

## Velociraptor

Velociraptor doesn't use `.map` files — it parses evtx natively with VQL
(`parse_evtx()`), reading the EventData by position. For file/device events:

```sql
SELECT EventData.Data[1] AS TargetFilename,   -- Data[2] 1-based
       EventData.Data[4] AS HwSerial,         -- Data[5] 1-based
       EventData.Data[5] AS Vsn               -- Data[6] 1-based
FROM parse_evtx(filename=Archive)
WHERE System.EventID.Value IN (500, 503, 504)
```

(VQL arrays are 0-based, so `Data[N-1]` = the 1-based positions above.) For
operational 9xx events, read `EventData.Data[0]` (the single message string).

## Correlating with Sysmon

Parse Sysmon with EvtxECmd's bundled maps (EIDs 11/23/26), parse this tool's archive
with these maps, load both in Timeline Explorer, and pivot on **TargetFilename** +
**UtcTime** — the two fields these maps deliberately mirror from Sysmon. USN catches
changes Sysmon's minifilter can miss and carries retroactive history; together they
close gaps neither has alone. The device events add the "what removable media was
involved, and was it tampered" dimension Sysmon doesn't provide at all.

## Validate

EvtxECmd flags map problems with `had validation errors`. These maps are YAML-valid;
always test against a real archive from your build before relying on them.
