# FB_V90_TableIndex — S7-1200 G2 ⇄ SINAMICS V90 PN, positioning by table index

SCL function block that runs a V90 PN servo in **basic positioner (EPOS)** mode over
PROFINET with **SIEMENS telegram 111** (PZD-12/12). The PLC picks a table index; the block
runs the drive enable / fault sequence, sends the move and reports status.

> **Status: draft for commissioning.** Not compiled in TIA Portal and not run against a real
> V90. A few telegram bits are tagged `[VERIFY]` — check them against the V90 PN manual
> before the axis moves (see [Verify before first move](#verify-before-first-move)).

## Files

| File | Purpose |
|---|---|
| `typeV90TableEntry.scl` | PLC data type: one table row (position, velocity, acc, dec, abs/rel) |
| `FB_V90_TableIndex.scl` | The function block |
| `DB_V90_Table.scl` | Global DB holding a 16-entry table per axis |

Import in TIA Portal: *PLC → External source files → Add new external file*, then
*Generate blocks from source* in the order **type → FB → DB**.

## Two ways to use "table index"

| `xUseDriveTable` | Where the row lives | What goes on the wire |
|---|---|---|
| `FALSE` (default) | PLC: `Table[iTableIndex]` | MDI setpoint: target, velocity, acc, dec, abs/rel |
| `TRUE` | V90: traversing block 0..15 (set up in V-ASSISTANT) | block number only |

Teach-in (`xTeach`) writes the PLC table only. The block does **not** write the drive's own
traversing-block parameters (that is an acyclic parameter-channel job, not part of this FB).

## Setting up

1. **V90 PN / V-ASSISTANT** — control mode *Basic positioner (EPOS)*, telegram **111**, LU per
   load revolution set to match the table units, EP/enable terminals and STO wired.
2. **TIA Portal hardware** — add the V90 PN (GSDML), assign device name and IP, select
   telegram 111. Note the module's **hardware identifier** (system constant) and use it for
   `hwIdIn` / `hwIdOut` (normally the same ID for both; use the separate IDs if your catalog
   shows them).
3. **Import** the three sources and create the instance DB.
4. **Call** the FB every scan from a cyclic OB:

```scl
"iDB_V90_Axis1"(
    hwIdIn := "V90_Axis1~Telegram111",
    hwIdOut := "V90_Axis1~Telegram111",
    xEnable := "Axis1".cmd.enable,
    xAck := "Axis1".cmd.ack,
    xExecute := "Axis1".cmd.go,
    xAbort := "Axis1".cmd.abort,
    xHome := "Axis1".cmd.home,
    iTableIndex := "Axis1".cmd.index,
    rOverridePct := "Axis1".cmd.override,
    Table := "DB_V90_Table".Axis1);

"Axis1".sts.ready := "iDB_V90_Axis1".xReady;
"Axis1".sts.done := "iDB_V90_Axis1".xDone;
"Axis1".sts.errorId := "iDB_V90_Axis1".wErrorId;
```

Typical sequence: `xEnable` → wait `xReady` → `xHome` (until `xHomed`) → set `iTableIndex`,
pulse `xExecute` → wait `xDone`. Errors and drive faults are cleared with a rising edge on `xAck`.

## Interface summary

**Inputs** — `hwIdIn`, `hwIdOut`, `xEnable`, `xAck`, `xExecute`, `xHalt`, `xAbort`, `xHome`,
`xSetRef`, `xRefCam`, `xHomeDirNeg`, `xJog1`, `xJog2`, `xTeach`, `xUseDriveTable`,
`iTableIndex`, `rOverridePct`, `wPosStw2Add`, `tAckTimeout`, `tMoveTimeout`
(`xAck`, `xExecute`, `xHome`, `xTeach` act on a rising edge; `xEnable`, `xHalt`, `xAbort`, `xSetRef`, `xJog*` are levels).

**Outputs** — `xReady`, `xBusy`, `xDone`, `xHomed`, `xFault`, `xWarning`, `xError`, `wErrorId`,
`wFaultCode`, `wWarnCode`, `diActPos`, `diActVeloRaw`, `iActiveIndex`, `iState`, raw `wZsw1`,
`wPosZsw1`, `wPosZsw2`, `iRetValRd`, `iRetValWr`.

**InOut** — `Table : Array[0..15] of "typeV90TableEntry"`.

Error IDs are listed in the block header (`16#8001`…`16#8021`).

`wPosStw2Add` is OR-ed into POS_STW2. Use it for bits the block does not manage — in
particular the software / hardware limit-switch enables of the V90 PN (the block does not set
them for you; check the manual for the bit numbers and enable them if you rely on drive-side limits).

## Telegram 111 map used by the block

PLC → drive (`aTx`): `STW1, POS_STW1, POS_STW2, STW2, OVERRIDE, MDI_TARPOS(32), MDI_VELOCITY(32), MDI_ACC, MDI_DEC, PZD12(user)`
Drive → PLC (`aRx`): `ZSW1, POS_ZSW1, POS_ZSW2, ZSW2, MELDW, XIST_A(32), NIST_B(32), FAULT_CODE, WARN_CODE, PZD12(user)`

32-bit values are sent high word first. OVERRIDE / MDI_ACC / MDI_DEC: `16#4000 = 100 %`.
All indices and bit masks are named constants at the end of the block's interface, so a
correction is a one-line edit.

## Verify before first move

I could not open the Siemens manuals while writing this (the environment's network policy blocks
`support.industry.siemens.com`), so the items below come from search-result summaries of Siemens
documents, general PROFIdrive/EPOS knowledge, or memory. Check each against the *SINAMICS V90 PN
operating instructions* (telegram 111, control/status word tables — POS_STW1, POS_STW2, STW1, ZSW1).

| Item | Used in the block | Basis |
|---|---|---|
| Receive word order (PLC → drive) | STW1, POS_STW1, POS_STW2, STW2, OVERRIDE, TARPOS, VELOCITY, ACC, DEC, user | Siemens CU250X-2 EPOS getting-started doc (summary) |
| `POS_STW1.15` activate MDI | `16#8000` | Same doc (summary) |
| `POS_STW2` bits 1 / 2 / 8 / 9 (set ref / ref cam / active-passive homing / start direction) | `16#0002` / `0004` / bit 8 = 0 / `0200` | Same doc (summary); **direction polarity not confirmed** |
| STW1 bits 0–7, 10 | standard PROFIdrive + EPOS | High confidence |
| STW1.8/.9 jog, STW1.11 start referencing | `16#0100` / `0200` / `0800` | Memory — verify |
| ZSW1.10 target reached, .11 reference set, .12 task acknowledged | `16#0400` / `0800` / `1000` | Memory — verify |
| **Send word order after ZSW1** (POS_ZSW1, POS_ZSW2, ZSW2, MELDW, XIST_A, NIST_B, FAULT, WARN) | `aRx` indices | Memory — verify; affects position and fault code readout |
| **MDI absolute / relative** | `POS_STW1.13` (`16#2000`) = absolute | **Guess — must verify; a wrong bit makes moves absolute/relative the wrong way** |
| **Traversing block select** (`xUseDriveTable`) | `POS_STW1.0–5` | Memory — verify (V90 has 16 blocks → 4 bits) |
| MDI velocity unit | 1000 LU/min | Memory — verify |
| `PZD12` | user word, unused | Siemens doc (summary): free for V90 via p29150/p29151 |

The block treats the task-acknowledge bit (ZSW1.12) as either a level or a toggle (it waits for
the bit to *change*), so it works with both interpretations.

## Commissioning checklist

1. Axis uncoupled / free to turn, `rOverridePct` = 10, hand on the stop.
2. `xEnable` → `iState` should reach 10 and `wZsw1` should show bit 2 (operation enabled) set.
   If it stalls in state 1 or 2 the block raises `16#8021` after 10 s: check STO, EP terminals,
   control mode and telegram selection.
3. Jog with `xJog1` / `xJog2` (speed and direction come from the drive's jog parameters).
4. Home (`xHome`) — verify `xHomed`; if homing never ends, check `xRefCam`, direction and the
   drive's homing mode.
5. Jog to a position, `xTeach` into an index, move away, `xExecute` that index and check you
   land on the taught position.
6. Do one **relative** and one **absolute** move and confirm the direction/distance is right —
   this is what proves the MDI abs/rel bit.
7. `xAbort`, `xHalt` and `xEnable` drop mid-move: axis must stop under the drive's decel ramps.
8. Pull the PROFINET cable: `wErrorId` = `16#8010`; the drive's own telegram-failure reaction applies.

## Assumptions about the S7-1200 G2

- The telegram is moved with `DPRD_DAT` / `DPWR_DAT` into word arrays (consistent 24-byte
  transfer). I could not confirm these instructions for the G2 from Siemens documentation; if
  your TIA version rejects them, replace regions *Read_telegram* / *Write_telegram* — nothing
  else in the block touches the I/O.
- Only plain SCL is used (no VARIANT logic, no S7-1500-only features), `HW_IO`, `TON_TIME`,
  optimized block access.
- Call it from a cyclic OB; scan time should be at or below the PROFINET update time of the drive.

## What was checked

- A structural lint of the SCL (balanced blocks, every `#name` declared, no writes to inputs or
  constants, statement terminators).
- A line-for-line Python port of the state machine run against a mock V90 (enable sequence,
  homing, MDI and drive-table moves, abort, timeouts, validation errors, fault acknowledge,
  comms loss, drive dropout mid-move, level and toggling task-acknowledge). All scenarios pass.

This validates the sequencing, not SCL syntax in TIA Portal and not the real V90 bit assignments.

## Sources

None of these could be opened directly; they were consulted through search-result summaries only.

- Siemens, *CU250X-2 EPOS function Getting Started part 8 — Telegram 111 for positioning*
  (support entry 109737985)
- Siemens, *SINA_POS / FB284 with V90 PN* application notes (entries 109766642, 109765618)
- Siemens, *PLC read/write V90 traversing block* (entry 109791870) — relevant if you also want to
  write the drive's own traversing-block table acyclically
