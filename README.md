# TcSparkplug

### A Sparkplug B serialization library Beckhoff's TwinCAT PLC

TcSparkplug implements the [Sparkplug B](https://sparkplug.eclipse.org/) specification for TwinCAT 3 PLCs. It handles MQTT connection management, the full NBIRTH/NDEATH/DBIRTH/DDATA/DDEATH message lifecycle, protobuf encoding, seq/bdSeq sequencing, and report-by-exception change detection — so application code only needs to describe its data.

**Dependencies:** Beckhoff TF6701 IoT Communication (Tc3_IotBase), TwinCAT 3.1.4026 or later.

---

## Quick start

### 1. Declare variables

```pascal
PROGRAM MAIN
VAR
    // Session manager — handles MQTT connection and Sparkplug lifecycle.
    // nBdSeq MUST be RETAIN so it survives power cycles (Sparkplug B requirement).
    fbSpkSession  : FB_SparkplugSession;
    nBdSeq        : ULINT; // {attribute 'TcInitSymbol'} or mark RETAIN in GVL

    bSpkEnable    : BOOL := TRUE;

    // One FB_SparkplugDevice per logical device (motor, drive, sensor, etc.)
    fbMotor1      : FB_SparkplugDevice;
    bDeviceInit   : BOOL;

    // Variables whose values will be reported as Sparkplug metrics.
    // These can be read from hardware I/O, linked to an axis, etc.
    nMotorSpeed   : UDINT := 0;
    bMotorRunning : BOOL  := FALSE;
    fMotorTemp    : REAL  := 0.0;

    tDataInterval : TON;
END_VAR
```

### 2. Call every cycle

```pascal
// Session must run first so GroupId/NodeId are set before AddDevice reads them.
fbSpkSession(
    sGroupId    := 'PlantA',
    sNodeId     := 'PLC1',
    sBrokerHost := '192.168.1.10',
    bEnable     := bSpkEnable,
    nBdSeq      := nBdSeq);

// ── Initialization ────────────────────────────────────────────────────────────

// Register the metrics you want to publish. The alias number becomes the
// protobuf field number — keep it stable across firmware versions.
// The Sparkplug data type is inferred automatically from the IEC variable type.
IF NOT bDeviceInit THEN
    fbMotor1.AddMetric('Speed',       1, nMotorSpeed);    // UDINT  → UInt32
    fbMotor1.AddMetric('Running',     2, bMotorRunning);  // BOOL   → Boolean
    fbMotor1.AddMetric('Temperature', 3, fMotorTemp);     // REAL   → Float

    // Register the device with the session.
    // The session will call fbMotor1.OnBirth() automatically on every birth cycle.
    fbSpkSession.AddDevice('Motor1', fbMotor1);
    bDeviceInit := TRUE;
END_IF

// ── Cyclic device call (required) ─────────────────────────────────────────────

// Must be called every cycle. Runs change detection and, with bAutoPublish := TRUE,
// automatically publishes a DDATA containing only the metrics that changed.
fbMotor1(bAutoPublish := TRUE);

// ── Optional periodic full refresh ────────────────────────────────────────────

// PublishData() with no arguments sends all metrics regardless of change —
// useful as a keep-alive or belt-and-suspenders refresh.
tDataInterval(IN := NOT tDataInterval.Q, PT := T#30S);
IF tDataInterval.Q THEN
    fbMotor1.PublishData();
END_IF
```

---

## How it works

```
PLC cycle
  │
  ├─ FB_SparkplugSession()       ← runs the MQTT state machine every cycle
  │     │
  │     ├─ Idle → Connecting     ← initiates MQTT connect with NDEATH as Will
  │     ├─ Connecting → Birthing ← subscribes to NCMD topic
  │     ├─ Birthing              ← publishes NBIRTH (seq=0)
  │     │     └─ calls OnBirth() on each registered FB_SparkplugDevice
  │     │           └─ device publishes DBIRTH (seq=1, 2, ...)
  │     ├─ Online                ← polls inbound NCMD; rebirth on Node Control/Rebirth
  │     └─ Reconnecting          ← 5 s backoff, then retry; bdSeq increments each attempt
  │
  ├─ fbMotor1()                  ← runs change detection every cycle
  │     └─ bAutoPublish = TRUE   → publishes DDATA with only changed metrics on any change
  │
  └─ fbMotor1.PublishData()      ← explicit full publish (e.g. periodic refresh)
```

**Sequence numbers** (`seq`, 0–255 wrapping) are managed entirely by the session via `GetNextSeq()`. Device FBs never need to track seq themselves.

**bdSeq** must be declared `RETAIN` in the calling program. It increments on every ungraceful disconnect so the host can detect missed NDEATH messages.

**DBIRTH** is published automatically — the session calls `OnBirth()` on every registered device at the start of each birth cycle (including after reconnects and NCMD rebirth commands). The application does not need to detect or handle birth cycles manually.

**Change detection** runs inside the FB body every cycle. Each metric's live value is compared against its last-published shadow using byte-exact comparison (discrete types) or a configurable deadband (REAL/LREAL). The shadow only advances when a publish succeeds, so `bDataPending` stays TRUE on a failed publish and the data is retried next cycle.

---

## Supported metric types

| IEC 61131-3 type | Sparkplug B type |
|---|---|
| `BOOL` | Boolean |
| `SINT` / `INT` / `DINT` | Int8 / Int16 / Int32 |
| `LINT` | Int64 |
| `USINT` / `UINT` / `UDINT` | UInt8 / UInt16 / UInt32 |
| `ULINT` | UInt64 |
| `BYTE` / `WORD` / `DWORD` / `LWORD` | UInt8 / UInt16 / UInt32 / UInt64 |
| `REAL` | Float |
| `LREAL` | Double |
| `STRING(n)` | String |

---

## Report by exception

`FB_SparkplugDevice` tracks each metric independently. When `bAutoPublish := TRUE`, only metrics whose value has changed since the last successful publish are included in the DDATA — not the whole payload.

### Deadband (REAL / LREAL only)

Float metrics support a deadband threshold to suppress noise. Set a device-wide default at init time, or override per metric by alias after `AddMetric`:

```pascal
// Device-wide default — applied to all REAL/LREAL metrics at AddMetric time
fbMotor1(bAutoPublish := TRUE, fDefaultDeadband := 0.5);

// Per-metric override — e.g. tighter threshold on a precision sensor
fbMotor1.SetDeadband(3, 0.1);  // alias 3 = Temperature
```

A deadband of `0.0` (the default) publishes on any change.

### Publishing modes

| Call | What gets sent |
|---|---|
| `fbMotor1(bAutoPublish := TRUE)` | Only changed metrics, automatically |
| `fbMotor1.PublishData()` | All metrics (full refresh) |
| `fbMotor1.PublishData(bChangedOnly := TRUE)` | Only changed metrics, on demand |

`bDataPending` is available as a `VAR_OUTPUT` if you prefer to drive the publish decision yourself instead of using `bAutoPublish`.

---

## Multiple devices

Add as many `FB_SparkplugDevice` instances as needed — the session manages up to 16.

```pascal
fbDrive.AddMetric('Velocity',  1, fDriveVelocity);
fbDrive.AddMetric('Torque',    2, fDriveTorque);
fbSpkSession.AddDevice('Drive1', fbDrive);

fbSensor.AddMetric('Pressure', 1, fPressure);
fbSensor.AddMetric('Flow',     2, fFlow);
fbSpkSession.AddDevice('Sensor1', fbSensor);
```

---

## Publishing a DDEATH

Call `PublishDeath()` before intentionally taking a device offline (e.g. maintenance mode). The session remains connected; only that device is marked dead.

```pascal
IF bMaintenanceRequested THEN
    fbMotor1.PublishDeath();
END_IF
```
