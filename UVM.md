# UVM
Two-top system:
- HDL top
  - Instantiates DUT
  - Instantiates interface
  - Creates clock
  - Registers virtual interface into UVM config db
Example:
```
```

- HVL top
-
## UVM Config DB
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
