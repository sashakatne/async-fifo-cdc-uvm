# Asynchronous FIFO CDC Design and Verification Study

## For reviewers (30-second read)

- **What:** a 64-entry, 8-bit asynchronous FIFO crossing an 80 MHz write clock and a 50 MHz read clock, verified with a UVM environment. Canonical code is in [`Post_M5/UVM/`](Post_M5/UVM/).
- **Team and my part:** a 4-person team project from Spring 2024. I made 66 of the 96 commits and wrote the RTL, UVM scoreboard, driver, and monitors. I re-verified and cleaned up the evidence in May 2026.
- **Bugs fixed:** both CDC pointer synchronizers were reset from the source clock domain instead of the destination ([design.md D-1](Post_M5/UVM/design.md)). Simulation could not catch it, because the testbench toggles both resets together; it was a design-review fix. The half flags came from one counter written by both clock domains, which I replaced with per-domain pointer arithmetic (D-2).
- **Results:** a clean UVM run covers 659 writes and 596 reads with 0 mismatches. An injected write-data corruption (`WDATA_CORRUPTION_BUG`) produced 250 scoreboard mismatches ([transcript](Post_M5/docs/transcript_datacorruptionbug.txt)). Directed fill/drain tests confirm the half flags at 32 entries, full at 64, and empty at 0.
- **Limits:** simulation only. There are no SVA assertions or formal proofs here, and simulation does not prove metastability safety.

This repository documents the design, verification, debugging, and validation of
an asynchronous FIFO for safe data transfer across independent clock domains.
The work is organized as a milestone progression from a conventional
SystemVerilog testbench to a final UVM-based verification environment with
scoreboarding, functional coverage, bug injection, reset-aware checking, and
farm-generated waveform evidence.

The canonical final implementation is located in
[`Post_M5/UVM/`](Post_M5/UVM/).  Earlier milestone directories are preserved as
historical design and verification snapshots so the development path remains
auditable.

## Abstract

Asynchronous FIFOs are commonly used in digital systems where producer and
consumer logic operate under unrelated clocks.  Correct implementation requires
careful handling of clock-domain crossing (CDC), pointer synchronization,
full/empty flag generation, reset behavior, and verification race avoidance.

This project implements and verifies a 64-entry, 8-bit asynchronous FIFO with an
80 MHz write domain and a 50 MHz read domain.  The final RTL uses local binary
pointers, Gray-coded cross-domain pointers, two-flop destination-domain
synchronizers, wrap-aware full/empty detection, and next-pointer occupancy for
half-full and half-empty flags.

The final verification environment uses UVM agents, independent read/write
stimulus, accepted-transfer monitors, a reset-aware queue scoreboard, directed
flag-threshold tests, intentional bug-injection hooks, and DUT-scoped coverage.
The checked-in evidence includes Siemens Questa farm transcripts, coverage
summaries, parsed waveform samples, and rendered waveform images.

## Research Objectives

The project investigates four practical questions:

1. **CDC correctness:** How should FIFO pointers and status flags be structured
   so data can cross between unrelated clocks without unsafe shared state?
2. **Verification architecture:** How can a testbench independently validate
   ordering, overflow protection, underflow protection, and flag timing?
3. **Race avoidance:** How should drivers and monitors be scheduled so
   simulator event ordering does not create false pass/fail behavior?
4. **Evidence quality:** How can simulator results be captured in a form that is
   reproducible, reviewable, and useful for design debugging?

## Summary Of Final Results

| Category | Result |
|---|---|
| Canonical target | [`Post_M5/UVM`](Post_M5/UVM/) |
| Simulator | Siemens Questa 2021.3_1 |
| FIFO depth | 64 entries |
| Data width | 8 bits |
| Write clock | 12.5 ns period, 80 MHz |
| Read clock | 20 ns period, 50 MHz |
| CDC method | Gray pointers plus two-flop destination-domain synchronizers |
| Reference model | Reset-aware queue scoreboard |
| UVM result | `UVM_WARNING : 0`, `UVM_ERROR : 0`, `UVM_FATAL : 0` |
| Coverage result | 100% DUT-filtered code coverage and 100% covergroup coverage |
| Directed flag result | Immediate half-full, full, half-empty, and empty threshold behavior |
| Evidence manifest | [`Post_M5/UVM/MANIFEST.txt`](Post_M5/UVM/MANIFEST.txt) |

![Post_M5 directed flag debug waveform](Post_M5/UVM/flag_debug_waveforms.png)

The waveform above was rendered from a farm-generated VCD.  It shows both
clocks with red rising-edge markers, decimal pointer locations, write/read
enables, and all four status flags.  The visualization demonstrates that the
status flags are asserted at the intended threshold edges rather than merely
settling later in the simulation.

## Repository Organization

The repository is intentionally structured as a series of snapshots.  Each
directory represents a distinct verification maturity level rather than a layer
that must be imported by later directories.

| Directory | Role |
|---|---|
| [`M1/ConventionalTB`](M1/ConventionalTB/) | Procedural SystemVerilog testbench with queue-based checking |
| [`M2/CLASS`](M2/CLASS/) | Class-based pre-UVM environment with generator, driver, monitor, scoreboard, and environment |
| [`M3/CLASS`](M3/CLASS/) | Enhanced class-based environment with coverage and directed half-flag timing evidence |
| [`M4/UVM`](M4/UVM/) | Initial UVM conversion with agents, sequencers, sequences, monitors, and scoreboard |
| [`M5/UVM`](M5/UVM/) | UVM coverage, assertions, bug-injection hooks, and refined run flow |
| [`Post_M5/UVM`](Post_M5/UVM/) | Final canonical UVM implementation and verification evidence |
| [`Miscellaneous`](Miscellaneous/) | Standalone FIFO variants and focused directed debug experiments |

## Asynchronous FIFO Design Problem

An asynchronous FIFO buffers data between a write domain and a read domain that
do not share a clock.  In this project, write operations can occur every
12.5 ns while read operations can occur every 20 ns.  The FIFO must preserve
data ordering while preventing overflow and underflow.

Two implementation hazards are central:

- **Unsafe shared occupancy state:** A conventional `fifo_count` written by both
  clock domains is not a valid CDC structure.
- **Multi-bit pointer sampling:** A raw binary pointer can change multiple bits
  at once, making it unsafe to sample directly in another clock domain.

The final design resolves these hazards with local binary arithmetic,
cross-domain Gray-coded pointers, and destination-domain synchronization.

```text
Write domain                                  Read domain
------------                                  -----------
binary_wptr                                   binary_rptr
    |                                             |
    v                                             v
Gray-coded wptr  ---> 2-FF sync into rclk ---> rq2_wptr
wFull / wHalfFull                         rEmpty / rHalfEmpty

Read domain                                   Write domain
-----------                                   ------------
Gray-coded rptr  ---> 2-FF sync into wclk ---> wq2_rptr
```

## Final RTL Architecture

The canonical DUT is [`Post_M5/UVM/async_fifo.sv`](Post_M5/UVM/async_fifo.sv).
It is decomposed into five functional blocks:

```text
asynchronous_fifo
|-- fifo_memory      storage array with gated write/read access
|-- write_pointer    binary/Gray write pointer, wFull, wHalfFull
|-- read_pointer     binary/Gray read pointer, rEmpty, rHalfEmpty
|-- sync_w2r         write pointer synchronized into the read clock domain
`-- sync_r2w         read pointer synchronized into the write clock domain
```

### Design Decision 1: Binary Locally, Gray Across Domains

Binary pointers are used inside each local clock domain because pointer
increment and occupancy arithmetic are straightforward in binary.  Before a
pointer crosses to the opposite domain, it is converted to Gray code.  Gray code
limits each increment to a one-bit transition, reducing the CDC sampling hazard.

### Design Decision 2: One Extra Pointer Bit

The FIFO stores 64 entries, so the RAM address is 6 bits.  The read and write
pointers are 7 bits wide.  The additional MSB distinguishes equal-address cases:

- equal address and same wrap state: FIFO empty;
- equal address and different wrap state: FIFO full.

This is the reason full detection cannot be implemented as a naive equality
comparison.

### Design Decision 3: Destination-Domain Synchronizers

Each two-flop synchronizer is clocked and reset in the domain that consumes the
synchronized pointer:

```systemverilog
sync_w2r (.clk(rclk), .rst_n(rrst), ...);  // write pointer into read domain
sync_r2w (.clk(wclk), .rst_n(wrst), ...);  // read pointer into write domain
```

This detail is essential.  A synchronizer clocked in the source domain does not
protect destination-domain logic from metastability.

### Design Decision 4: Next-Pointer Half-Flag Timing

The final design computes half flags from next local pointer occupancy and the
synchronized remote pointer:

```systemverilog
assign wptr_diff = binary_wptr_next - binary_rptr_sync;
assign rptr_diff = binary_wptr_sync - binary_rptr_next;
```

This avoids unsafe dual-clock `fifo_count` logic and ensures immediate threshold
behavior:

- `wHalfFull` asserts on the accepted write that reaches 32 entries;
- `rHalfEmpty` asserts on the accepted read that drops occupancy to 32 entries.

Earlier milestone snapshots exposed a one-clock-late half-flag behavior.  The
directed tests and waveforms below document the corrected timing.

## Directed Flag Timing Evidence

Write-side threshold zoom:

![Write-side half-full and full threshold](Post_M5/UVM/flag_debug_write_thresholds.png)

Read-side threshold zoom:

![Read-side half-empty and empty threshold](Post_M5/UVM/flag_debug_read_thresholds.png)

The directed flag-threshold run proves the following events:

- at write pointer 32, `wHalfFull` is asserted;
- at write pointer 64, `wFull` is asserted;
- a write attempted while full is blocked;
- at read pointer 32 with write pointer 64, `rHalfEmpty` is asserted;
- at read pointer 64, `rEmpty` is asserted.

The corresponding farm transcript is
[`Post_M5/UVM/flag_threshold_transcript_farm.txt`](Post_M5/UVM/flag_threshold_transcript_farm.txt).

## UVM Verification Methodology

The final verification environment uses separate write and read agents.  Each
domain has its own sequencer, driver, and monitor.  Both monitors publish
observed transactions to a scoreboard through UVM analysis ports.

```text
tb_top
|-- DUT: asynchronous_fifo
|-- intf: shared SystemVerilog interface
`-- uvm_test_top: fifo_random_test / fifo_base_test
    `-- fifo_env
        |-- write_agent
        |   |-- write_sequencer
        |   |-- write_driver
        |   `-- write_monitor
        |-- read_agent
        |   |-- read_sequencer
        |   |-- read_driver
        |   `-- read_monitor
        `-- fifo_scoreboard
```

### Race-Free Driver And Monitor Timing

The DUT samples control and data on rising clock edges.  The final drivers
therefore drive stimulus on negative edges so enables and data are stable before
the DUT sampling edge.  This avoids simulator race artifacts.

The monitors publish only accepted transfers:

- write transaction: `winc && !wFull`;
- read transaction: `rinc && !rEmpty`.

This distinction is important because overflow and underflow attempts are valid
negative stimulus.  They should test the DUT's protection logic without
polluting the scoreboard reference queue.

### Reset-Aware Queue Scoreboard

The final [`fifo_scoreboard`](Post_M5/UVM/async_fifo_scoreboard.sv) maintains an
independent SystemVerilog queue reference model:

- accepted writes push expected `wData`;
- accepted reads pop and compare against DUT `rData`;
- unknown data, empty-queue reads, and data mismatches raise `uvm_error`;
- reset assertions flush pending expected data so the scoreboard remains aligned
  during reset-sequence testing.

Final farm scoreboard summary from
[`Post_M5/UVM/transcript_farm.txt`](Post_M5/UVM/transcript_farm.txt):

```text
Writes observed       : 659
Reads  observed       : 596
Reset flushes         : 2
Residual expected_q   : 0
Mismatches / errors   : 0
Verdict: *** PASSED ***
UVM_WARNING : 0
UVM_ERROR   : 0
UVM_FATAL   : 0
```

## Coverage Strategy

The coverage flow intentionally instruments the DUT rather than inflating
coverage by measuring testbench internals.  The final run script uses
DUT-scoped coverage:

```tcl
vopt tb_top -o top_optimized +acc +cover=sbfec+asynchronous_fifo(rtl).
```

The coverage model includes:

- statement, branch, expression, and condition coverage on the DUT;
- covergroups for flag behavior and flag transitions;
- data integrity and data pattern coverage;
- burst, idle, high-frequency, abrupt-change, reset, and throughput behavior.

![Post_M5 coverage summary](Post_M5/docs/coverage_summary.png)

The final farm evidence reports:

```text
DUT filtered instance coverage: 100.00%
TOTAL COVERGROUP COVERAGE:     100.00%
Covergroup types:              9
```

## Intentional Defect Injection

The final RTL includes guarded defect macros that can be enabled one at a time
in [`Post_M5/UVM/run.do`](Post_M5/UVM/run.do).  They remain disabled in the
default passing build.

| Macro | Injected defect | Verification purpose |
|---|---|---|
| `WDATA_CORRUPTION_BUG` | Corrupts write data before storage | Validates scoreboard data-integrity checking |
| `SYNC_BUG` | Removes the second synchronizer flop | Exercises CDC robustness assumptions |
| `RPTR_BUG` | Uses binary read pointer instead of Gray encoding | Tests pointer encoding assumptions |
| `WPTR_FULLFLAG_BUG` | Uses naive full equality | Validates wrap-aware full detection |

This defect-injection strategy demonstrates that the verification environment is
capable of failing for meaningful RTL bugs, not only passing for the nominal
configuration.

## Technical Contributions

The following project elements are particularly relevant from a digital design
and verification perspective:

**Independent reference model**

The scoreboard does not inspect DUT memory or pointer internals.  It derives
expected behavior independently from accepted write transactions.  This reduces
the risk of reproducing the DUT's assumptions inside the checker.

**Accepted-transfer modeling**

The verification environment separates attempted operations from accepted
operations.  This enables overflow and underflow pressure tests without creating
false scoreboard entries.

**CDC-correct half-flag implementation**

Half-full and half-empty flags are generated from local next pointers and
synchronized remote pointers.  This removes unsafe shared occupancy state and
fixes one-clock-late threshold behavior.

**Reset-aware self-checking**

The final default run includes reset stimulus.  The scoreboard handles reset
flushes explicitly instead of disabling resets to simplify the test.

**Reproducible waveform artifacts**

Farm-generated VCD files are parsed into CSV, SVG, and PNG artifacts.  The
committed images provide design-debug evidence without requiring a waveform GUI.

**DUT-focused coverage reporting**

Coverage is scoped to the design under test.  This prevents testbench code from
artificially inflating the reported verification result.

## Evidence And Artifacts

Important final evidence files:

| File | Purpose |
|---|---|
| [`Post_M5/UVM/MANIFEST.txt`](Post_M5/UVM/MANIFEST.txt) | Concise run manifest with commands, results, and verdict |
| [`Post_M5/UVM/transcript_farm.txt`](Post_M5/UVM/transcript_farm.txt) | Full final UVM farm transcript |
| [`Post_M5/UVM/flag_threshold_transcript_farm.txt`](Post_M5/UVM/flag_threshold_transcript_farm.txt) | Directed flag-threshold farm transcript |
| [`Post_M5/UVM/design.md`](Post_M5/UVM/design.md) | Detailed post-fix design and verification notes |
| [`Post_M5/UVM/waveform_samples.csv`](Post_M5/UVM/waveform_samples.csv) | Parsed waveform samples from the final farm VCD |
| [`Post_M5/UVM/flag_debug_samples.csv`](Post_M5/UVM/flag_debug_samples.csv) | Parsed directed flag waveform samples |

Raw simulator artifacts such as `.vcd`, `.ucdb`, `.wlf`, `work/`, and
`modelsim.ini` are intentionally gitignored.  The repository commits the
reviewable derived evidence: transcripts, manifests, CSV samples, SVGs, and
PNGs.

## Reproducibility

This project targets Siemens/Mentor Questa.  The checked-in farm transcripts
were generated with Questa 2021.3_1.

Run the final UVM simulation:

```sh
cd Post_M5/UVM
vsim -c -do "do run.do; quit -f"
```

Run the focused directed flag-threshold simulation:

```sh
cd Post_M5/UVM
vsim -c -do "do run_flags.do; quit -f"
```

Regenerate waveform images after VCD-producing runs:

```sh
cd Post_M5/UVM
python3 make_artifacts.py
python3 make_flag_debug_waveforms.py
```

The explicit `quit -f` is required for noninteractive runs because the run
scripts set `NoQuitOnFinish 1`.

## Recommended Review Path

For a technical review, inspect the following files in order:

1. [`Post_M5/UVM/async_fifo.sv`](Post_M5/UVM/async_fifo.sv)
   Final RTL: memory, pointer handlers, Gray conversion, synchronizers, and
   status flags.

2. [`Post_M5/UVM/async_fifo_scoreboard.sv`](Post_M5/UVM/async_fifo_scoreboard.sv)
   Reset-aware queue scoreboard and final PASS/FAIL summary.

3. [`Post_M5/UVM/async_fifo_coverage.sv`](Post_M5/UVM/async_fifo_coverage.sv)
   Functional coverage model.

4. [`Post_M5/UVM/run.do`](Post_M5/UVM/run.do)
   Questa compile, optimization, simulation, and coverage flow.

5. [`Post_M5/UVM/MANIFEST.txt`](Post_M5/UVM/MANIFEST.txt)
   Farm run manifest and concise verification verdict.

6. [`Post_M5/UVM/design.md`](Post_M5/UVM/design.md)
   Detailed design-review notes and post-fix rationale.

## Concise Project Summary

This repository presents a complete CDC-oriented asynchronous FIFO verification
case study.  The final design uses Gray-coded pointers and destination-domain
two-flop synchronizers to safely transfer data between independent write and
read clocks.  The final verification environment uses UVM agents, race-free
drivers, accepted-transfer monitors, a reset-aware queue scoreboard, functional
coverage, and intentional defect injection.  Directed farm simulations prove
immediate half-full, full, half-empty, and empty flag behavior.  The final
Questa run is warning-clean, reports zero UVM errors and fatals, and reaches
100% DUT-focused code coverage and 100% covergroup coverage.

## Project Notes

- [`Post_M5/UVM`](Post_M5/UVM/) is the canonical final implementation.
- Earlier milestone directories are preserved to show the evolution from
  procedural verification to class-based verification and then to UVM.
- [`CLAUDE.md`](CLAUDE.md) documents the simulator workflow and farm conventions
  used during repository repair and verification.
