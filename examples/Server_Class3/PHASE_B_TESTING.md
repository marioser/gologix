# Phase B testing — gologix server STRING handling (issue #58)

This document tells the automation team **exactly** how to validate the fix on
branch `fix/server-string-handling`. The fix touches how the gologix CIP server
serializes and parses Logix `STRING` UDTs (`LEN: DINT, DATA: SINT[82]`,
StructTypeCRC `0x0FCE`) when external CIP clients (Studio 5000 MSG, Kepware,
Ignition, pylogix) read from or write to a gologix server.

Phase A (this branch) is already validated against pylogix as an external
client (`pylogix_interop.py`). Phase B is what you are about to do: replace
pylogix with a **real Rockwell controller** sending MSG instructions, and
optionally a SCADA stack (Kepware / Ignition / FactoryTalk).

If anything in this doc is unclear, ping back with the failing step instead of
guessing. The criteria below are binary — pass or fail — by design.

---

## 1. What you need

### Hardware

- 1 × ControlLogix or CompactLogix controller running v20+ firmware.
- Network reachable between the controller and the host that will run the
  gologix server (any laptop / PC / VM is fine).
- Studio 5000 Logix Designer (for loading the L5X and running MSG online).
- *(Optional, only if you also want to validate SCADA path)* Kepware
  KEPServerEX, Ignition with the Allen-Bradley Logix driver, or
  FactoryTalk Linx — any one is enough.

### Software on the gologix host

- Go 1.21 or later (`go version` should print 1.21+).
- This branch checked out:
  ```bash
  git clone https://github.com/marioser/gologix
  cd gologix
  git checkout fix/server-string-handling
  ```

### Network

- The gologix server listens on **TCP/UDP 44818** (standard EtherNet/IP).
  No firewall between the controller and the host on that port.
- The host running gologix needs a static IP the controller can reach.
- The example uses CIP path `1,0` (backplane, slot 0) for the first
  provider and `1,1` (backplane, slot 1) for the second. The MSG
  instruction's **Connection Path** is what selects which provider, not
  the IP.

---

## 2. Tags to create on the controller

Open the L5X import dialog and add (or create manually) these tags. The names
and types are not negotiable — the validation steps reference them literally.

| Tag name              | Type        | Initial value          | Used for                  |
|-----------------------|-------------|------------------------|---------------------------|
| `gologixTestString`   | `STRING`    | `"Hello World"`        | Read test (scalar STRING) |
| `gologixWriteTarget`  | `STRING`    | empty                  | Write test (round-trip)   |
| `msgRead`             | `MESSAGE`   | see config below       | Triggers CIP Data Read    |
| `msgWrite`            | `MESSAGE`   | see config below       | Triggers CIP Data Write   |

### MSG instruction config — `msgRead`

Configure on the **Configuration** tab:

| Field                   | Value                                |
|-------------------------|--------------------------------------|
| Message Type            | CIP Data Table Read                  |
| Source Element          | `teststring`                         |
| Number Of Elements      | `1`                                  |
| Destination Element     | `gologixTestString`                  |

On the **Communication** tab:

| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| Path             | `<gologix_host_ip>, 1, 0`                                      |
| Connected        | ☑ (checked — class 3 connected MSG)                            |
| Cache Connections| ☐ (unchecked is fine; either works)                            |

Replace `<gologix_host_ip>` with the IP of the machine running `go run .`.

### MSG instruction config — `msgWrite`

| Field                   | Value                                |
|-------------------------|--------------------------------------|
| Message Type            | CIP Data Table Write                 |
| Source Element          | `gologixWriteTarget`                 |
| Number Of Elements      | `1`                                  |
| Destination Element     | `writestring`                        |

Communication tab: same as `msgRead`.

---

## 3. Start the gologix server

On the host machine, from the repo root:

```bash
cd examples/Server_Class3
go run .
```

Expected console output every 5 seconds (sanity ticker):

```
2026/05/17 10:00:00 Data 1: map[testdint:12 testint:3 teststring:Hello World testtag1:12345 testtag2:543.21 testtag3:[1 2 3 4 5 6 7 8 9 10] writestring:]
2026/05/17 10:00:00 Data 2: map[]
```

Leave this terminal running. If you see `bind: address already in use`, port
44818 is busy — close any other CIP server / pylogix script / Wireshark
capture-with-replay process and retry.

---

## 4. Validation scenarios

Run these **in order**. Each one has a binary pass/fail. If a step fails, stop
and report it with the failing tag value, the gologix server log line, and a
Wireshark capture if you have one (filter `enip`).

### Scenario B1 — Scalar STRING read from a real controller

1. In Studio 5000, set the controller to **Run mode**.
2. Trigger `msgRead` (force the rung true, or manually toggle the `.EN` bit).
3. **Pass criteria:**
   - `msgRead.DN` becomes 1 (done).
   - `msgRead.ER` stays 0 (no error).
   - `gologixTestString` in the controller shows `Hello World` (LEN=11, DATA
     contains the ASCII bytes, the rest of the 82-byte DATA buffer is 0x00).
4. **Fail criteria:**
   - `msgRead.ER == 1` with `.EXERR` set. Capture the EXERR hex value.
   - `gologixTestString.LEN != 11` or DATA bytes differ.

### Scenario B2 — Scalar STRING write from a real controller

1. In Studio 5000, set `gologixWriteTarget.LEN` and `.DATA` to a non-empty
   value (e.g. `"phase-b-roundtrip"` — LEN=17).
2. Trigger `msgWrite`.
3. **Pass criteria:**
   - `msgWrite.DN == 1`, `.ER == 0`.
   - In the gologix server console, the next 5-second tick shows
     `writestring:phase-b-roundtrip` inside the `Data 1: map[...]` log line.
4. **Fail criteria:**
   - `.ER == 1` (capture EXERR).
   - The server log shows `writestring:` empty or with truncated/garbled
     content.

### Scenario B3 — Write then read-back round-trip

1. After B2 succeeds, run `msgRead` but with **Source Element** temporarily
   changed to `writestring` (instead of `teststring`).
2. **Pass criteria:**
   - The controller's `gologixTestString` now shows the value you wrote in
     B2 (`phase-b-roundtrip`).
3. **Fail criteria:**
   - Mismatch between what you wrote and what comes back.

### Scenario B4 — Stress / repeated triggers

1. Trigger `msgRead` 20 times in quick succession (a counter + loop, or a
   timed rung).
2. **Pass criteria:**
   - All 20 messages report `.DN == 1`, no `.ER`.
   - No memory growth or hang on the gologix server (it should respond
     under 50ms per request on a local network).
3. **Fail criteria:**
   - Any single message ER==1, or the server becomes unresponsive.

### Scenario B5 *(optional, only if SCADA is in scope)* — SCADA client read

Configure your SCADA stack (Kepware/Ignition/FactoryTalk) to point at the
gologix host IP, slot 0. Add tags `teststring` and `writestring`. Subscribe
both.

- **Pass criteria:** SCADA shows the current string values, updates within
  the polling interval after a `msgWrite`.
- **Fail criteria:** SCADA reports a quality bad / type mismatch / connection
  error.

---

## 5. How to report results

Open a comment on the internal ticket **SMBX-270** with:

1. Pass/fail per scenario (B1–B5).
2. Controller model + firmware version (`Properties → General` in Studio 5000).
3. gologix host OS + Go version.
4. For any failure: the failing tag value, the relevant gologix server log
   line, and (ideally) a `.pcapng` capture of the failing exchange.

If everything passes, that's the green light to push the PR upstream to
`danomagnum/gologix`. **Do not comment on the upstream issue #58 directly** —
all coordination stays on SMBX-270 until we open the PR.

---

## 6. Quick reference — wire details the fix enforces

For Wireshark / packet-analysis sanity:

- A Logix `STRING` UDT on the wire is **`type segment (4 bytes: 0xA0 0x02
  0xCE 0x0F)` + `LEN (DINT, 4 bytes)` + `DATA (SINT[82], 82 bytes)` = 90
  bytes total per element**.
- StructTypeCRC for `STRING` is `0x0FCE` (little-endian on the wire: `CE 0F`).
- The read response service code must echo the request service: `0xCC`
  (Read|Response) for `0x4C` Read, `0xD2` (FragRead|Response) for `0x52`
  FragRead. A mismatch will cause MSG `.ER` with EXERR `0x16` (Object Does
  Not Exist) or similar on stricter stacks.

If a capture shows DATA shorter than 82 bytes, or LEN > 82, that's a wire
bug — file it under SMBX-270.
