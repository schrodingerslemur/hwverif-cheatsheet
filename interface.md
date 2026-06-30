Example:
```systemverilog
interface system_bus_v5 (input bit clk);
	parameter ADDR_WIDTH = 16;
	parameter DATA_WIDTH = 16;
	bit rst_n; // active 0 synchronous reset
	logic re; // active 1 read enable
	logic we; // active 1 write enable
	logic [ADDR_WIDTH-1:0] addr; // address from outside, from system
	logic [DATA_WIDTH-1:0] data_from_system; // data coming in
	logic [DATA_WIDTH-1:0] data_to_system; // data going out
	logic [DATA_WIDTH-1:0] data_op; // operation generated data

	//use input to only allow sampling the signal
	//use output to only allow driving the signal 
	//you can also use inout, which allows both, but it is a bit less elegant
	clocking drv_cb @(posedge clk);
		default input #1step output #1;
		output rst_n, re, we, addr, data_from_system; 
		input data_to_system, data_op;
	endclocking

	clocking mon_cb @(negedge clk);
		default input #1step output #1;
		input rst_n, re, we, addr, data_from_system; 
		// the signals above were listed as output in a previous
		input data_to_system, data_op;
	endclocking
endinterface
```

Refer to clocking blocks:
- `#1step` means sample input (DUT output) right before the clock signal (i.e. posedge for drv_cb)
-  #1 means drive outputs 1 time unit after clock signal
