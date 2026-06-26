# UVM
## Two-top system:
HDL top and HVL top
### HDL top
  - Instantiates interface
  - Instantiates DUT
  - Creates clock
  - Registers virtual interface into UVM config db
    
Example (hdl_top.sv):
```systemverilog
module hdl_top();
  // Instantiate interface
  system_bus #(...) bus(...);

  // Instantiate DUT
  streaming_engine #(...) dut(...);

  // Create clock
  bit clk;
  initial clk = 0;
  always begin
    #10; clk = ~clk;
  end
  assign bus.clk = clk;

  // Register virtual interface into UVM config db
  initial begin
    import uvm_pkg::uvm_config_db;
    uvm_config_db#(virtual system_bus)::set(null, "uvm_test_top", "system_bus", bus);
  end
endmodule: hdl_top
```

### HVL top
- Imports UVM packages
- Controls which test to run

Example (hvl_top.sv):
```systemverilog
module hvl_top();

  // Import uvm packages
  import uvm_pkg::*;
  import lab_pkg::*;

  // Control which test to run
  initial begin
    run_test("test_name");
  end

endmodule: hvl_top
```
Example (lab_pkg.sv):
```systemverilog
package lab_pkg;
    // standard UVM imports
    import uvm_pkg::*;
    `include "uvm_macros.svh"
    // shared compile-time parameters
    import params_pkg::*;
    // user-defined imports. order matters!
    `include "operations.svh"
    `include "seq.svh"
    `include "drv.svh"
    `include "mon.svh"
    `include "agt.svh"
    `include "env.svh"
    `include "tst.svh"
endpackage: lab_pkg
```

### UVM Config DB
Allows to get/set `value` for `field_name` in `inst_name` using UVM component `cntxt` as starting search point <br>
Example:
```systemverilog
uvm_config_db#(virtual system_bus)::set(null, "uvm_test_top", "system_bus", bus);
```
Parameters:
```
static function void <set>/<get> (
  uvm_component cntxt,
  string        inst_name,
  string        field_name,
  T             value
);
```
> `cntxt` is mostly *this* or *null* <br>
> `"uvm_test_top"` is convention for `inst_name`

## UVM Components
