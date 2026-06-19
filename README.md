# hwverif-cheatsheet

## Coverage
1) Create `covfile.cf` with this content:
```systemverilog
select_coverage -block -expr -toggle -fsm 
set_statement_scoring
set_branch_scoring
//set_fsm_scoring -hold_transition on
//set_fsm_arc_scoring
//set_fsm_reset_scoring
//set_explicit_block_scoring -off
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
