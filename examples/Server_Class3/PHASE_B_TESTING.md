# Phase B testing — gologix server STRING handling (issue #58)

This document tells the automation team **exactly** how to validate the fix
on branch `fix/server-string-handling`. The fix touches how the gologix CIP
server serializes and parses Logix `STRING` UDTs (`LEN: DINT, DATA: SINT[82]`,
StructTypeCRC `0x0FCE`) when external CIP clients (Studio 5000 MSG, Kepware,
FactoryTalk, Ignition, pylogix) read from or write to a gologix server.

Phase A is already validated against `pylogix` as an external client (see
`pylogix_interop.py`). Phase B is what you are about to do: replace pylogix
with a **real Rockwell controller** sending MSG instructions, and optionally
a SCADA stack.

Each scenario has a binary pass / fail. If a step fails, stop and report it
with the failing tag value, the gologix server log line, and (ideally) a
Wireshark `.pcapng` filtered on `enip`. Do **not** patch the server code
locally — send the evidence back and we iterate.

---

## 1. What you need

### Hardware

- 1 × ControlLogix or CompactLogix controller with firmware ≥ 20.
- Network reachable between the controller and the host that will run the
  gologix server.
- Studio 5000 Logix Designer v32 or later (for importing the L5X).
- *(Test C only)* FactoryTalk View / Ignition / any SCADA stack with an
  EtherNet/IP driver.

### Software on the gologix host

- Go 1.21 or later (`go version`).
- This branch checked out:
  ```bash
  git clone https://github.com/marioser/gologix.git
  cd gologix
  git checkout fix/server-string-handling
  go build -o /tmp/gologix-server ./examples/Server_Class3/
  ```
  For Windows: `GOOS=windows GOARCH=amd64 go build -o gologix-server.exe ./examples/Server_Class3/`.

### Network

- TCP 44818 and UDP 2222 open between the controller, the SCADA host, and
  the gologix host.
- The gologix host needs a static IP reachable from the PLC. Note this IP
  — it goes in the MSG Communication path.
- The gologix host **cannot** have another EtherNet/IP server or a Logix
  emulator on the same machine; the server binds 44818 unconditionally.
  Use a VM or LXC if FactoryTalk Linx already owns 44818 on your main
  workstation.

---

## 2. Start the gologix server

```bash
/tmp/gologix-server 2>&1 | tee /tmp/gologix-server.log
```

Expected output in the first few seconds:

```
INFO Listening on TCP port 44818
INFO Listening on UDP port 2222
... (every 5s) Data 1: map[testdint:12 testint:3 teststring:Hello World ... writestring:]
```

Leave this running. Record the host IP — the MSG path uses it.

---

## 3. Import the L5X into Studio 5000

The L5X file `gologix_phase_b_tags.L5X` ships attached to the internal
ticket (SMBX-270 for the SGS automation team; if you do not have access,
request it from the contributor).

1. Open your PLC project.
2. **File → Import → Import Component → select `gologix_phase_b_tags.L5X`**.
3. Import as **Controller Tags**. The L5X adds:
   - `gologix_src_string` (STRING) — pre-loaded with the test value
     `pylogix-round-trip-from-PLC` (LEN=27).
   - `gologix_dst_string` (STRING) — empty buffer for read responses.
   - `msg_read_test_string`, `msg_write_writestring`, `msg_read_writestring_back`
     (MESSAGE) — three MSG control tags.
   - `trig_msg_read_test_string`, `trig_msg_write_writestring`,
     `trig_msg_read_writestring_back` (BOOL) — three trigger latches to fire
     each MSG manually from the online editor.

---

## 4. Configure each MESSAGE tag

Right-click each MSG tag → **Configure**. **Configuration** tab:

| Tag MSG | Message Type | Source Element | # Elements | Destination Element |
|---|---|---|---|---|
| `msg_read_test_string` | CIP Data Table Read | `teststring` | 1 | `gologix_dst_string` |
| `msg_write_writestring` | CIP Data Table Write | `gologix_src_string` | 1 | `writestring` |
| `msg_read_writestring_back` | CIP Data Table Read | `writestring` | 1 | `gologix_dst_string` |

**Communication** tab (same for all three):

| Field | Value |
|---|---|
| Path | `<gologix_host_ip>, 1, 0` (e.g. `192.168.1.50, 1, 0`) |
| Communication Method | `CIP` |
| Connected | ☑ |
| Cache Connections | ☑ |

The `1, 0` after the IP is the internal CIP path the example uses to
distinguish providers on the same server (virtual backplane, slot 0). The
`Server_Class3` example puts everything in that slot.

---

## 5. Add three trigger rungs to a continuous task routine

```ladder
| trig_msg_read_test_string         MSG(msg_read_test_string)         |
| trig_msg_write_writestring        MSG(msg_write_writestring)        |
| trig_msg_read_writestring_back    MSG(msg_read_writestring_back)    |
```

Download to the PLC. Put it in **Run**.

---

## 6. Validation scenarios

Run **in order**. Each scenario produces evidence — keep screenshots and
log excerpts attached to the ticket.

### Test A — Read a STRING from the gologix server

1. Confirm `gologix_dst_string` is empty (`LEN=0`, DATA all `$00`).
2. Set `trig_msg_read_test_string = 1`.
3. Wait for `msg_read_test_string.DN = 1` (should be immediate).
4. **Capture** Studio 5000 showing `gologix_dst_string` with `LEN = 11`
   and DATA = `Hello World`.
5. If `.ER = 1`: capture `msg_read_test_string.ERR` and `.EXERR` (CIP
   status codes).
6. Return the trigger to 0.

**Pass criteria:** `gologix_dst_string == "Hello World"`, `LEN = 11`,
`.ER = 0`.

### Test B — Write a STRING + read it back

1. Set `trig_msg_write_writestring = 1`. Wait for `.DN`.
2. In the `gologix-server.log` console, the next 5-second tick must show:
   ```
   Data 1: map[... writestring:pylogix-round-trip-from-PLC ...]
   ```
   `writestring` flipped from empty to the value that was pre-loaded in
   `gologix_src_string`.
3. **Capture** the server log line.
4. Return the trigger to 0.
5. Set `trig_msg_read_writestring_back = 1`. Wait for `.DN`.
6. **Capture** Studio 5000 showing
   `gologix_dst_string = "pylogix-round-trip-from-PLC"`.

**Pass criteria:** server log shows the written value, MSG read-back
returns the same value, all three `.ER = 0`.

### Test C — SCADA Rockwell *(optional but recommended)*

1. In FactoryTalk View Studio (or your SCADA), create a new Data Server /
   Topic pointing at the gologix host IP with path `1, 0`.
2. Browse the tags. At minimum `teststring`, `writestring`, `testdint`,
   `testint`, `testtag1`, `testtag2`, `testtag3` (DINT array) must show
   up.
3. Create a display with an input field bound to `writestring`.
4. From the SCADA, write a distinct value, e.g. `scada-write-test`.
5. **Capture** the server log showing `writestring:scada-write-test`.
6. **Capture** the SCADA reading `writestring` back and showing the new
   value.

**Pass criteria:** SCADA reads and writes complete with good quality, no
type-mismatch errors on either side.

### Test D — Stress *(optional, useful if you have time)*

Trigger `msg_read_test_string` 20 times in quick succession (counter +
timed rung, or manual toggling).

**Pass criteria:** all 20 messages `.DN = 1`, no `.ER`, gologix server
keeps responding under ~50 ms per request on a local network.

---

## 7. Collect evidence

Attach to the internal ticket (or shared folder, depending on team
process):

- `gologix-server.log` from server start through the last test.
- Studio 5000 screenshots (Tests A, B).
- FactoryTalk / SCADA screenshots (Test C).
- Any `.ERR` / `.EXERR` values for failed MSGs.
- *(Optional)* `.pcapng` capture filtered on `enip` if something fails on
  the wire.

---

## 8. Acceptance criteria

- [ ] Test A: `gologix_dst_string == "Hello World"` after the MSG read.
- [ ] Test B: server log shows `writestring:pylogix-round-trip-from-PLC`
      and read-back returns the same value.
- [ ] Test C: SCADA reads and writes `writestring` cleanly.
- [ ] Zero `.ER = 1` on any of the three MSG tags.
- [ ] Zero `ERROR`-level lines in `gologix-server.log` during the tests.

When all four boxes are green, this is the signal to push the PR upstream
to `danomagnum/gologix`. **Do not comment on upstream issue #58 directly**
— all coordination stays on the internal ticket until we open the PR.

---

## 9. Wire-level reference (for `.pcapng` triage)

If you capture packets:

- A Logix `STRING` UDT on the wire is **4-byte type segment
  (`0xA0 0x02 0xCE 0x0F`) + `LEN (DINT, 4 bytes)` + `DATA (SINT[82], 82
  bytes)` = 90 bytes per element**.
- StructTypeCRC for `STRING` is `0x0FCE` (little-endian on the wire: `CE
  0F`).
- The read response service code must echo the request service: `0xCC`
  (Read | Response) for `0x4C` Read, `0xD2` (FragRead | Response) for
  `0x52` FragRead. Mismatch causes `.ER` with EXERR `0x16` (Object Does
  Not Exist) on stricter stacks.

DATA shorter than 82 bytes, or LEN > 82, is a wire bug — report it.
