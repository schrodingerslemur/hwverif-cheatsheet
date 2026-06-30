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

## UVM Phases
Only the important ones:
1) Build phase: top to bottom
  - Memory allocation
2) Connect phase: bottom to top
  - Handler/pointer sharing, connection between components
3) Run phase
  - Consumes time

## UVM components
The following follows RSP-REQ model with focus on drv-seq-mon interaction and agent connection. Will show driver, agent and monitor code

### Driver
1) Implement all UVM import + class stuff
2) Declare virtual interace
3) Get virtual interface from UVM config db in build_phase
4) Implement RSP-REQ protocol in run_phase
   
drv.svh:
```systemverilog
class drv extends uvm_driver#(req_item, rsp_item);
  `uvm_component_utils(drv)

  // Declare virtual interface
  virtual <interface_type>#(<interface_params>) bus;

  function new(string name="drv", uvm_component parent);
    super.new(name, parent);
  endfunction: new

  function void build_phase(uvm_phase phase);
    super.build_phase(phase);

    // Get virtual interface
    if (!uvm_config_db #(virtual <interface_type>#(<interface_params>))::get(null,
        "uvm_test_top", "vif", bus))
            `uvm_fatal("LOG", "VIF not found");
        `uvm_info(get_type_name(), $sformatf("end of build phase"), UVM_NONE)
  endfunction: build_phase

  task run_phase(uvm_phase phase);
    req_item m_req;
    rsp_item m_rsp;

    // Implement RSP-REQ protocol
    forever begin
      seq_item_port.get_next_item(m_req); // get next item from sequencer
      m_rsp = rsp_item::type_id::create("m_rsp"); create rsp object (req was created by sequencer)
      m_rsp.set_id_info(m_req); // associate req with rsp

      @(bus.drv_cb); // clocking block
      // drive all output signals with NB assignments
      bus.drv_cb.rst_n <= m_req.rst_n;
      ...

      // drive all input signals with B assignments
      m_rsp.data = bus.drv_cb.data;

      // Call done on item
      seq_item_port.item_done(m_rsp);
    end
  endtask: run_phase
endclass: drv
```

### Agent
1) Implement all UVM class + import stuff
2) Create driver, monitor, and sequence objects in build_phase
3) Connect sequencer to driver in connect_phase

agt.svh:
```systemverilog
class agt extends uvm_agent;
    `uvm_component_utils(agt)

    drv m_drv;
    mon m_mon;
    uvm_sequencer #(req_item, rsp_item) m_sqr;

    function new(string name="agt", uvm_component
    parent);
        super.new(name, parent);
    endfunction: new

    // Create driver, monitor and sequencer
    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        m_drv = drv::type_id::create("m_drv", this);
        m_mon = mon::type_id::create("m_mon", this);
        m_sqr = uvm_sequencer #(req_item, rsp_item)::type_id::create("m_sqr", this);
        `uvm_info(get_type_name(), $sformatf("end of build phase"), UVM_NONE)
    endfunction: build_phase

    // Connect driver to sequencer
    function void connect_phase(uvm_phase phase);
        m_drv.seq_item_port.connect(m_sqr.seq_item_export);
    endfunction: connect_phase
endclass: agt
```

### Monitor
1) Implement all UVM class + import stuff
2) Declare virtual interface
3) Get virtual interface from UVM Config DB in build_phase
4) Implement model

mon.svh:
```systemverilog
class mon extends uvm_monitor;
  `uvm_component_utils(mon)

  // Declare virtual interface
  virtual <interface_type>#(<interface_params>) bus;

  function new(string name="mon", uvm_component parent);
        super.new(name, parent);
  endfunction: new

  // Get virtual interface
  virtual function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        if (!uvm_config_db#(virtual <interface_type>#(<interface_params>)::get(null,
        "uvm_test_top", "vif", bus))
            `uvm_fatal("mon", "Could not get vif")
        `uvm_info(get_type_name(), $sformatf("end of build phase"), UVM_NONE)
  endfunction

  // Implement model
  bit a, b, c;
  bit q [$];
  function reset_model();
    q.delete();
    {a, b, c} = '0;
  endfunction: reset_model();

  task run_phase(uvm_phase phase);
    super.run_phase(phase);

    forever begin
      @(bus.mon_cb);

      if (bus.mon_cb.rst_n == 1'b0) begin
        reset_model();
        continue;
      end

      // Decode signals
      // Based on decoded signals, simulate different changes
    end
  endtask: run_phase
endclass: mon
```

## UVM sequences
Sequences encapsulate and execute sequence items

First, create the basic `req_item` and `rsp_item`. Request items need constraints and should instantiate inputs into the DUT, response items do not need constraints and should instantiate outputs from the DUT.

1) Implement UVM stuff
2) Instantiate `rand` or normal signals
3) Implement constraints, only for `req_item`

seq.svh:
```systemverilog
class req_item extends uvm_sequence_item;
  `uvm_object_utils(req_item)

  // Rand and non-rand type variables
  rand bit re;
  rand bit we;
  bit rst_n = 1'b1; // you can set variable values
  rand bit [DATA_WIDTH-1:0] data;
  rand bit [ADDR_WIDTH-1:0] addr;

  // Implement constraint
  constraint valid {addr dist {BASE_CFG:=20, [BASE_CMD: BASE_CMD + OP_LAST]:=20,
  BASE_STATUS:=20, [BASE_DATA:BASE_DATA + FIFO_LENGTH]:=20}; }

  function new(string name = "req_item");
    super.new(name);
  endfunction: new

endclass: req_item

class rsp_item extends uvm_sequence_item;
  `uvm_object_utils(rsp_item)

  // Normal variables without constraints
  logic [DATA_WIDTH-1:0] data_to_system;
  logic [DATA_WIDTH-1:0] data_op;

  function new(string name = "rsp_item");
    super.new(name);
  endfunction: new

endclass: rsp_item
```

Secondly, you can keep extending classes to build on top of request items (usually not response items).
```systemverilog
class write_req_item extends req_item;
  `uvm_object_utils(write_req_item)

  constraint force_write { we == 1'b1; re == 1'b0; }

  function new(string name = "write_req_item");
    super.new(name);
  endfunction: new

endclass: write_req_item
```

Thirdly, create sequences which execute the sequence items.

1) Implement UVM macros
2) Create handlers for the sequence items
3) Create request item (not response item as driver creates them)
4) Create a virtual task body
5) Either call `uvm_do`/`uvm_do_with` or:
     - Start item -> Randomize item -> Finish item -> Get response
   
```systemverilog
class exec_seq extends uvm_sequence#(req_item, rsp_item);
  `uvm_object_utils(exec_seq)

  function new(string name = "");
    super.new(name);
  endfunction: new

  // Create handlers
  req_item req;
  rsp_item rsp;

  // Create other constrained random variables if required
  rand bit [ADDR_WIDTH-1:0] op_addr;
  constraint valid_op { op_addr inside {[BASE_CMD: BASE_CMD + OP_LAST]}; }

  // Create virtual task body
  virtual task body();
    // Create request item
    req = req_item::type_id::create("req");

    // Start item -> Randomize item -> Finish item -> Get response
    start_item(req);
    if (!req.randomize() with { addr == op_addr; }) begin
      `uvm_error(get_type_name, "Failed to randomize sequence item")
    end
    finish_item(req);
    get_response(rsp);

    // To deal with responses, do something like:
    // done = !rsp.data_to_system[2];
    // if (done) `uvm_do(req);

    `uvm_info(get_type_name, $sformatf("RSP item: data_to_system=%h, data_op=%h",
          rsp.data_to_system, rsp.data_op), UVM_MEDIUM)
  endtask
endclass: exec_seq
```

Fourthly, create parent sequences which call other sequences if required

```systemverilog
class fill_queue_seq extends uvm_sequence #(req_item, rsp_item);
  `uvm_object_utils(fill_queue_seq)

  function new(string name = "fill_queue_seq");
    super.new(name);
  endfunction

  virtual task body();
    exec_seq exec1;
    repeat (32) begin
      `uvm_do_with(
        exec1,
        { exec1.op_addr == BASE_CMD + OP_PUSH; }
      )
    end
  endtask

endclass: fill_queue_seq
```

Refer to **tst.svh** on how to choose which sequence to execute.

## Encapsulating UVM components
### Environment
1) Instantiate agent handler
2) Create agent in build phase

env.svh:
```systemverilog
class env extends uvm_env;
    `uvm_component_utils (env)
    agt m_agt;

    function new(string name="env", uvm_component
    parent);
        super.new(name, parent);
    endfunction: new

    virtual function void build_phase (uvm_phase phase);
        super.build_phase (phase);
        m_agt = agt::type_id::create ("m_agt", this);
        `uvm_info(get_type_name(), $sformatf("end of build phase"), UVM_NONE)
    endfunction
endclass
```

### Test
1) Instantiate environment handler
2) Create environment in build phase
3) Run sequences in run phase using `m_seq.start(m_env.m_agt.m_sqr)`

tst.svh:
```systemverilog
class lab5 extends uvm_test;
    `uvm_component_utils(lab5)

    env m_env;

    function new(string name="lab5", uvm_component
    parent);
        super.new(name, parent);
    endfunction: new

    // Create environment in build phase
    virtual function void build_phase(uvm_phase phase);
        super.build_phase (phase);
        m_env = env::type_id::create ("m_env", this);
        `uvm_info (get_type_name(), $sformatf ("end of build phase"), UVM_NONE)
    endfunction

    virtual task run_phase(uvm_phase phase);
        seq1 m_seq = seq1::type_id::create("m_seq");

        // This is not required unless you have rand variables in sequence.
        if (!m_seq.randomize()) `uvm_error(get_type_name, "randomize failed")

        super.run_phase(phase);
        phase.raise_objection (this); // let me run...
        m_seq.start(m_env.m_agt.m_sqr);
        phase.drop_objection (this); // ...done running
    endtask
endclass: lab5
```

