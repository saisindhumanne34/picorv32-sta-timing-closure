# picorv32-sta-timing-closure
STA and timing closure study on PicoRV32 (Sky130, OpenLane) — constraint setup, WNS/TNS across clock periods, and closing a setup violation via synthesis strategy and placement density changes
# PicoRV32 — Timing Constraints, STA & Timing Closure

This repository documents a focused Static Timing Analysis (STA) and timing closure study on the PicoRV32 RISC-V core, built directly on top of the [picorv32a / OpenLane / Sky130 SoC design project](../../soc-design-and-planning-vsd). Where that project took the design from RTL to a routed GDSII, this one asks a narrower question: **at what clock speed does this design actually work, why does it fail where it fails, and can we fix it without touching the RTL?**

**Tools:** OpenLane | OpenSTA | Sky130 PDK
**Design:** PicoRV32 RISC-V Core
**Flow stage covered:** Synthesis → Floorplan → Placement → CTS, with OpenSTA sign-off after each

---

## 1. Where This Starts From

This isn't a from-scratch project. The mechanical foundation — a working `pre_sta.conf`, a hand-built `my_base.sdc`, and a known-good OpenSTA invocation — already existed from earlier Day 4 work. This project extends that SDC with the constraint types it was still missing, then uses it to systematically characterize and close a real timing violation.

| Already had | What it is |
|---|---|
| `pre_sta.conf` | Config file invoking OpenSTA, in the OpenLane root directory |
| `my_base.sdc` | `designs/picorv32a/src/my_base.sdc` — hand-created SDC file |
| Working `sta` invocation | Confirmed via `sta -exit pre_sta.conf` inside the OpenLane Docker container |

**Environment:**
- OpenLane container: `zealous_kapitsa`, started via `cd /home/vscode/Desktop/OpenLane && make mount`
- Re-entered per session with `docker exec -it zealous_kapitsa bash`
- SDC file: `/openlane/designs/picorv32a/src/my_base.sdc`
- STA config: `/openlane/pre_sta.conf`
- Reports: `/openlane/reports/`

`[SCREENSHOT: terminal showing docker exec -it zealous_kapitsa bash and the resulting OpenLane container prompt]`

---

## 2. Auditing and Extending the SDC

Before writing anything new, the existing `my_base.sdc` was dumped and checked against five required constraint types: clock definition, I/O delay, clock uncertainty, false path, and multicycle path.

```bash
cat /openlane/designs/picorv32a/src/my_base.sdc
```

All five were already present:

| Constraint type | Status | Line |
|---|---|---|
| Clock definition | ✅ | `create_clock ... -period $::env(CLOCK_PERIOD)` |
| Input/output delay | ✅ | `set_input_delay` / `set_output_delay` at 20% of period |
| Clock uncertainty | ✅ | `set_clock_uncertainty 0.2 [get_clocks $::env(CLOCK_PORT)]` |
| False path | ✅ | `set_false_path -from [get_ports resetn]` — resetn is asynchronous |
| Multicycle path | ⚠️ placeholder | referenced `mem_wstrb_reg/Q` → `mem_addr_reg/D`, pins that don't exist in the actual netlist — fixed in Section 6 once real register names were available |

---

## 3. Running STA Across Five Clock Periods

**A note on the `sta` invocation:** the first attempts to run `sta pre_sta.conf > report.rpt` appeared to hang indefinitely. The cause: `pre_sta.conf` has no `exit` command at the end, so after running its checks, OpenSTA drops into an interactive Tcl prompt and waits for more input instead of returning control to the shell. The fix was to invoke it with `-exit`:

```bash
sta -exit pre_sta.conf > /openlane/reports/timing_20ns.rpt
```

This was then repeated for five clock periods — 20ns, 10ns, 8ns, 5ns, and 3ns — by editing `CLOCK_PERIOD` in the SDC before each run:

```bash
sed -i "s/set ::env(CLOCK_PERIOD).*/set ::env(CLOCK_PERIOD) 20.000/" /openlane/designs/picorv32a/src/my_base.sdc
sta -exit pre_sta.conf > /openlane/reports/timing_20ns.rpt
# repeated for 10.000, 8.000, 5.000, 3.000
```

`[SCREENSHOT: terminal showing the sed + sta command pair for at least one clock period]`

### Result — the 20ns "safe" clock still violates

The most important finding from this step: **even the loosest constraint tested (20ns) fails timing.** The critical path runs through PicoRV32's multiplier logic (`pcpi_mul`), a chain of 33+ combinational gate stages:

```
Startpoint: _28783_ (rising edge-triggered flip-flop clocked by clk)
Endpoint:   _28171_ (rising edge-triggered flip-flop clocked by clk)
Path Type: max
...
                                 22.45   data arrival time
                                 19.51   data required time
-----------------------------------------------------------------------------
                                 -2.95   slack (VIOLATED)
```

`[SCREENSHOT: full timing_20ns.rpt critical path, from Startpoint to slack VIOLATED]`

---

## 4. Extracting WNS / TNS for All Five Runs

```bash
grep -H "wns\|tns" /openlane/reports/timing_20ns.rpt /openlane/reports/timing_10ns.rpt \
  /openlane/reports/timing_8ns.rpt /openlane/reports/timing_5ns.rpt /openlane/reports/timing_3ns.rpt
```

| Clock Period | WNS (ns) | TNS (ns) |
|---|---|---|
| 20ns | -2.95 | -28.52 |
| 10ns | -12.95 | -1039.49 |
| 8ns | -14.95 | -1824.40 |
| 5ns | -17.95 | -4582.96 |
| 3ns | -19.95 | -7523.19 |

`[SCREENSHOT: terminal output of the grep command above, showing all five WNS/TNS pairs]`

All five results are internally consistent with a single fixed 22.45ns critical path — WNS drops by almost exactly the same amount the clock period does at each step, confirming the same bottleneck is dominating at every speed.

---

## 5. Documenting the "Before" State — 3ns Forced Violation

The 3ns run doubles as the deliberately broken baseline for the closure case study:

```
tns -7523.19
wns -19.95
```

This is the documented **"before"** state going into the closure attempt below.

---

## 6. Fixing the Multicycle Path Placeholder

With real timing data in hand, the placeholder multicycle path was replaced with an actual register pair pulled from a report:

```bash
grep -E "dfxtp" /openlane/reports/timing_20ns.rpt | head -20
```

This surfaced a real, existing flip-flop-to-flip-flop connection: `_29443_/Q` → `_29266_/D`.

```bash
sed -i 's|set_multicycle_path 2 -setup -from \[get_pins mem_wstrb_reg/Q\] -to \[get_pins mem_addr_reg/D\]|set_multicycle_path 2 -setup -from [get_pins _29443_/Q] -to [get_pins _29266_/D]|' \
  /openlane/designs/picorv32a/src/my_base.sdc
```

Rerunning STA confirmed the fix: the "pin not found" warnings disappeared, and WNS/TNS were unchanged from baseline — as expected, since this register pair sits on an unrelated path from the critical multiplier path.

```
tns -28.52
wns -2.95
```

`[SCREENSHOT: final my_base.sdc contents + the rerun confirming no pin-not-found warnings]`

---

## 7. Timing Closure — Two Techniques, Tested Independently

Both techniques were applied to **separate, isolated OpenLane runs** (`prep -design picorv32a -tag <name> -overwrite`), each re-run from synthesis through CTS, so their effects could be measured independently against the same 20ns baseline rather than compounding on top of each other.

### Technique 1 — Synthesis strategy (logic-level fix)

```
prep -design picorv32a -tag synth_strategy_test -overwrite
set ::env(SYNTH_STRATEGY) "DELAY 3"
run_synthesis
run_floorplan
run_placement
run_cts
```

`[SCREENSHOT: terminal showing the full run_synthesis → run_cts sequence completing cleanly]`

`pre_sta.conf` was repointed at the new run's netlist and STA rerun:

```bash
sta -exit pre_sta.conf > /openlane/reports/after_synth_strategy.rpt
```

**Result: fully closed.**

```
Startpoint: ... Endpoint: _39272_
...
                                 14.51   data arrival time
                                 19.52   data required time
-----------------------------------------------------------------------------
                                  5.01   slack (MET)
tns 0.00
wns 0.00
```

The critical path's data arrival time dropped from **22.45ns → 14.51ns**. `SYNTH_STRATEGY="DELAY 3"` tells Yosys to prioritize speed over area during technology mapping — it picks faster gate variants and restructures logic to shorten the critical path, fixing the violation at the logic-structure level, before placement even happens.

`[SCREENSHOT: full after_synth_strategy.rpt critical path showing slack MET, plus the wns/tns = 0.00 result]`

### Technique 2 — Placement density (physical-level fix)

```
prep -design picorv32a -tag density_test -overwrite
run_synthesis
run_floorplan
set ::env(PL_TARGET_DENSITY) 0.45
run_placement
run_cts
```

`[SCREENSHOT: terminal showing the set PL_TARGET_DENSITY command and run_placement/run_cts completing cleanly]`

```bash
sta -exit pre_sta.conf > /openlane/reports/after_density_change.rpt
```

**Result: no improvement.**

```
tns -28.52
wns -2.95
```

Checking the actual critical path confirmed this wasn't a fluke — it's the **identical path**, same instance names (`_29443_`, `_28171_`, etc.), same delay values down to the hundredth of a nanosecond, as the untouched baseline. Lowering placement density gives the placer more room and can shorten wire lengths, but it does nothing to a violation whose bottleneck is *logic depth*, not *interconnect delay*. Spreading cells out doesn't shorten a 33-stage combinational chain.

`[SCREENSHOT: after_density_change.rpt critical path, showing it's identical to the baseline]`

---

## 8. Comparison Table

| Clock Period / Run | WNS (ns) | TNS (ns) | Notes |
|---|---|---|---|
| 20ns | -2.95 | -28.52 | Baseline — violates even at a "safe" clock, due to the pcpi_mul combinational path |
| 10ns | -12.95 | -1039.49 | |
| 8ns | -14.95 | -1824.40 | |
| 5ns | -17.95 | -4582.96 | |
| 3ns (violation) | -19.95 | -7523.19 | Documented "before" state for the case study |
| After Technique 1 — SYNTH_STRATEGY="DELAY 3" | **0.00** | **0.00** | Fixed — critical path arrival time: 22.45ns → 14.51ns |
| After Technique 2 — PL_TARGET_DENSITY=0.45 | -2.95 | -28.52 | No improvement — identical critical path |

---

## 9. Critical Path Analysis

The dominant setup violation runs from `_28783_` to `_28171_`, both flip-flops clocked by `clk`. Between them sits a chain of over 30 combinational cells — buffers, OR/AND gates, and wide fan-in gates — associated with `genblk1.pcpi_mul`, PicoRV32's multiplier block. This is a **logic-depth problem**: the multiplier's combinational implementation is simply too deep to settle within one clock period at any of the tested speeds, including the nominally safe 20ns.

`report_checks -path_delay max` was used throughout to pull this path — the single worst timing path start-to-end, annotated with per-cell delay, slew, and fanout at each stage.

---

## 10. Timing Closure Case Study — Before / After

**Problem:** PicoRV32 fails setup timing even at a relaxed 20ns clock period (WNS -2.95ns, TNS -28.52ns), and the violation worsens sharply as the clock is tightened, reaching WNS -19.95ns at 3ns.

**Root cause:** A single dominant combinational path through the multiplier logic (`pcpi_mul`), roughly 22.45ns long across 33+ gate stages — well beyond what any of the tested clock periods can accommodate without a logic-level or structural change.

**Fix attempted:**
1. Changed `SYNTH_STRATEGY` to `"DELAY 3"` and re-ran synthesis through CTS.
2. Changed `PL_TARGET_DENSITY` to `0.45` and re-ran placement through CTS.

**Result:** Technique 1 fully closed the violation (WNS/TNS → 0.00), by shortening the critical path itself through faster gate selection and logic restructuring during synthesis. Technique 2 had zero effect — placement density changes only affect wire length and routing congestion, and this violation's bottleneck was never wire delay, it was logic depth.

**What we'd try next if Technique 1 hadn't worked:** multi-cycle path constraints on the multiplier's output register (if architecturally valid, i.e. if the design can tolerate the multiplier taking more than one cycle), pipelining the multiplier internally at the RTL level, or a targeted `set_max_delay` on that specific path paired with manual cell upsizing.

---

## 11. Key Takeaways

- A clock period that looks "safe" on paper isn't automatically timing-clean — always verify with STA rather than assuming.
- Not all timing-closure levers work for all violation types. A logic-depth-driven violation needs a logic-level fix; placement-level changes (density, in this case) can be completely inert against it.
- `sta` needs an explicit `-exit` when driven by a non-interactive script, or it will hang waiting at a Tcl prompt instead of returning control to the shell.
- Isolating each closure technique in its own OpenLane run (rather than stacking them) makes it possible to attribute improvement to a specific cause.

---

## 12. Deliverables in This Repo

- `sdc/my_base.sdc` — final extended SDC (clock, I/O delay, uncertainty, false path, multicycle path with real pin names)
- `reports/timing_20ns.rpt`, `timing_10ns.rpt`, `timing_8ns.rpt`, `timing_5ns.rpt`, `timing_3ns.rpt` — raw STA reports for all five clock periods
- `reports/after_synth_strategy.rpt` — STA after Technique 1
- `reports/after_density_change.rpt` — STA after Technique 2
- `screenshots/` — terminal evidence for each step above
- This README

---

## References

- [OpenLane — Achieving Timing Closure](https://openlane2.readthedocs.io/en/latest/usage/timing_closure.html)
- [OpenSTA GitHub + command reference](https://github.com/The-OpenROAD-Project/OpenSTA)
- [VLSIDA chip-tutorials — STA on a real OpenLane design](https://github.com/VLSIDA/chip-tutorials)
- Parent project: [soc-design-and-planning-vsd](../../soc-design-and-planning-vsd) — the picorv32a RTL-to-GDSII build this project extends
