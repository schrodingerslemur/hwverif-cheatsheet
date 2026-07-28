# Formal (JasperGold)

Formal proves a property over **all** inputs — no stimulus, no testbench. Good at control logic, bad at wide datapaths.

## The three statement types
| Statement | In formal | Use |
|---|---|---|
| `assert property` | proved or falsified | what the **design** must do |
| `assume property` | constrains the input space | what the **environment** promises |
| `cover property` | reachability check | is this scenario even possible |

> **If a property constrains a design *input*, it is an `assume`, not an `assert`.** Assertions constrain what the design *produces*.

Same text, completely different meaning:
```systemverilog
// "I claim this holds -- tell me if it doesn't"
SVA_WERE: assert property (@(posedge clk) disable iff (!rst_n) !(re && we));

// "only consider inputs where this holds"
SVA_NO_RW: assume property (@(posedge clk) disable iff (!rst_n) !(re && we));

// "is a back-to-back read even reachable?"
SVA_RERERE: cover property (@(posedge clk) disable iff (!rst_n) (re ##1 re ##1 re));
```

## Where to write them
Same place as simulation — inside the interface, or in an assertion module `bind`ed to the DUT (see **SVA.md**). Formal elaborates on the **DUT**, not on `top_hdl`, so no UVM files are analyzed.

## Running it
run.tcl:
```tcl
clear -all                                                       ;# reset the session

analyze -sv12 streaming_engine.sv system_bus.sv assertions.sv    ;# parse
elaborate -top streaming_engine_v7                               ;# build the model

clock clk                                                        ;# declare the clock
reset -expression {!rst_n}                                       ;# declare the reset

prove -all                                                       ;# attempt every property
report
```
```bash
jg                       # interactive GUI
jg -batch -tcl run.tcl   # batch
```
> `clock` and `reset` are mandatory. Without them every property fails immediately.

Useful additions:
```tcl
set_prove_time_limit 3600s
assume -from_assert SVA_FULL   ;# reclassify an assert as an assume
cover -all                     ;# reachability of cover properties
get_reset_info                 ;# confirm reset was understood
```

## Reading the results
| Status | Meaning |
|---|---|
| proven | holds for all inputs, forever |
| falsified | counterexample found — click it in the GUI for the waveform |
| undetermined | hit the time/depth limit, result unknown |
| vacuous | passed only because the antecedent never happened — **treat as a failure** |

> Over-constraining is how formal lies to you. An `assume` that is too strong proves everything over an input space that excludes the bug. Review every assumption, and `assert` in simulation whatever you `assume` in formal.
