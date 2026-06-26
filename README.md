# hwverif-cheatsheet
## Transaction-Level Modelling (TLM)
Write stimuli as a form of transactions (functions)

Example:
```systemverilog
function automatic write(addr, data);
  bus.we <= 1'b1;
  bus.re <= 1'b0;
  bus.addr <= addr;
  bus.data <= data;
  @(posedge bus.clk);
endfunction
```

## Coverage
### Types of coverage
1) Line coverage
    - Checks whether all lines were executed
2) Block/branch coverage 
    - Checks whether all blocks of code were executed
    - i.e. every always block, if, case statement
3) Expression coverage
    - Checks whether all parts of an expression was executed
    - Creates a truth table without useless cases
    - i.e. for a && b: `2'b11`, `2'b0x`, and `2'bx0` are checked
4) Toggle coverage
    - Checks whether every bit of every signal is toggled from 0-to-1 and 1-to-0
5) FSM coverage
    - Checks whether every state and state transition is exercised

### Coverage terminology
**bins**: enumerated buckets of all scenarios for given coverage metric <br>
**hole**: all unvisited bin

### Coverage analysis
1) Create `covfile.cf` with this content:
```systemverilog
select_coverage -block -expr -toggle -fsm 
set_statement_scoring
set_branch_scoring
set_fsm_scoring -hold_transition on
set_fsm_arc_scoring
set_fsm_reset_scoring
//set_explicit_block_scoring -off // uncomment to get rid of redundant cases
//set_toggle_scoring -sv_enum
```

2) To simulate, run
```bash
xrun <sv_files> -covfile covfile.cf [-covoverwrite] [-covdut <dut_module_name>] [-covtest <test_name>]
```

3) To open the GUI, run
```bash
imc
```

4) Press ***Load Data***, navigate to `cov_work/scope/test/` and click on ***load***.
5) Click on ***DUT*** and click on bottom right of screen (block, expression, toggle, etc.) to specify the type of coverage you want to navigate through.

### Merging coverage data
1) Create two different coverage database using `-covtest <test_name>` flag on xrun.
2) Run
```bash
imc -batch
merge <test_name_1> <test_name_2> -out <merged_test_name>
```
3) Launch imc in GUI mode again and load in new merged test data.
