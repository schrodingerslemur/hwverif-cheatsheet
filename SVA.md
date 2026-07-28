# SVA and bind

## Assert vs cover
- `assert property`: must always hold. Fails loudly, takes an `else`
- `cover property`: just records that it happened. **No `else`**

Example (inside an interface):
```systemverilog
interface system_bus_v7 (input bit clk);
	...
	SVA_WERE: assert property (@(posedge clk) disable iff (rst_n==1'b0) !(re && we))
		else $error("we/re asserted at the same time");

	SVA_RERERE: cover property (@(posedge clk) disable iff (rst_n==1'b0) (re ##1 re ##1 re)); // no else!
	SVA_WEWEWE: cover property (@(posedge clk) disable iff (rst_n==1'b0) (we ##1 we ##1 we)); // no else!
endinterface
```
> Always label your assertions (`SVA_WERE:`). Otherwise the tool names them for you and the log is unreadable.

## Named properties and default clocking
Declare `default clocking` once instead of repeating `@(posedge clk)`:
```systemverilog
module assertions(input logic clk, input logic rst_n, input logic [2:0] slicer);

default clocking cb1 @(posedge clk); endclocking

property valid_slicer;
	(slicer == 3'b001) || (slicer == 3'b010) || (slicer == 3'b100);
endproperty

SVA_SLICER1: assert property ( disable iff (!rst_n) valid_slicer);

endmodule
```
> `disable iff (<expr>)` kills the check while `<expr>` is true. Use it so reset doesn't fire every assertion.

## Operators
| Operator | Meaning |
|---|---|
| `a ##1 b` | `b` one cycle after `a` |
| `a ##[1:3] b` | `b` 1 to 3 cycles after `a` |
| `a \|-> b` | if `a`, then `b` in the **same** cycle |
| `a \|=> b` | if `a`, then `b` in the **next** cycle |
| `$rose(x)` / `$fell(x)` | 0-to-1 / 1-to-0 this cycle |
| `$past(x)` | value of `x` last cycle |
| `$stable(x)` | `x` unchanged since last cycle |
| `not p` | `p` never holds |

## bind
Use `bind` to attach an assertion module to a DUT **without editing the DUT**. This is how you reach internal signals.

Example (in top_hdl.sv):
```systemverilog
module top_hdl();
	...
	streaming_engine_v7 dut(.bus(bus));

	// bind <target_module> : <instance> <checker_module> <checker_inst> (<ports>);
	bind streaming_engine_v7 : dut assertions assertions_inst (
		.clk(clk),
		.rst_n(bus.rst_n),
		.slicer(dut.slicer)   // internal DUT signal
	);
endmodule
```
> `: dut` binds to that one instance. Drop it to bind to **every** instance of the module.

Add the assertion file to the `xrun` file list:
```bash
FILES="... top_hdl.sv system_bus.sv streaming_engine.sv assertions.sv"
xrun $FILES -uvm +UVM_TESTNAME="$1"
```
