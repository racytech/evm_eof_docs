## EVM Object Foramt (EOF) 

### Included Proposals.
- [EIP-3540: Structured Bytecode and Versioning (EVM Object Format v1)](#eip-3540-structured-bytecode-and-versioning-evm-object-format-v1)
- [EIP-3670: Code Validation](#eip-3670-code-validation)
- [EIP-4200: Static relative jumps](#eip-4200-static-relative-jumps)
- [EIP-4750: Functions](#eip-4750-functions)
- [EIP-5450: Stack Validation](#eip-5450-stack-validation)
- [EIP-6206: JUMPF instruction](#eip-6206-jumpf-instruction)
- [EIP-7480: Data section access instructions](#eip-7480-data-section-access-instructions)
- EIP-663: Unlimited SWAP and DUP instructions (TODO)
- EIP-7069: Revamped CALL instructions (does not require EOF) (TODO)
- [TBA: Contract Creation](#tba-contract-creation)
- TBA: Restrict code and gas introspection (TODO)

*see [Opcode Table](https://docs.google.com/spreadsheets/d/e/2PACX-1vS0ungUTs_SkqaSrp7oghcOEZr3oCJJMcIR9rk42s4tAzggDLE4jAQDifXXZu9rNqq2BK-HnDP7bzB9/pubhtml) for the opcode description and pseudocode*

### EIP-3540. Structured Bytecode and Versioning (EVM Object Format v1)

Legacy bytecode does not have any structure or any bytecode organization rules. Meaning that instractions in it can have any order. This makes it hard to analize and validate it. EIP-3540 organizes bytecode into clear structure. Following picture illustrates EOF version 1 format.

![Container Structure](/assets/eof_container.png)


### EIP-3670: Code Validation

Right now EVM can execute a faulty bytecode without knowing upfront that it was faulty. Execution of invalid bytecode still consumes gas and time. With this EIP it is possible to avoid such scenario. Since EOF has a pre-defined structure it is possible to validate a code at the deploy time.

Validation steps (order may not be right):
- check if bytecode starts with *magic (OxEF00)*, read version
- validate EOF headers, check if each section header meets its attributes 
- validate types section body, check if each code section metadata meets its attributes
- validate each code in code section body
    - validate instructions:
        - check if deprecated opcodes exist
        - check if there is no truncated instructions (when immediate_size of instruction out of bound)
        - check if instructions that refer to code section metadata (`CALLF`), container section body (`CREATE3, RETURNCONTRACT`) or data section body (`DATALOADN`) do have correct immediate value 
    - validate relative jumps. check `RJUMP`, `RJUMPI` and `RJUMPV` for valid jump destinations
    - validate max stack height (EIP-5450: Stack Validation)
- validate each sub-container in container section body by doing all the previous steps

### EIP-4200: Static relative jumps

Legacy bytecode moves program counter by `PUSHn ... JUMP/JUMPI` which takes about 11 and 13 gas if no other instructions present in between them. With `RJUMP`, `RJUMPI` and `RJUMPV` it takes 2, 4 and 4 gas respectevly. So it is cheaper to move instruction pointers. 

`RJUMP`, `RJUMPI` and `RJUMPV` can not point to `PUSHn/RJUMP/RJUMPI/RJUMPV` and they can not point outside of code bounds (code section body's code[i] on image above). Allowed to point to a `JUMPDEST`, but is not required to.

Opcdes deprecated: `PC` 


### EIP-4750: Functions

This proposal introduces functions instead of frequent jumps and removes the need for `JUMP/JUMPI`. To perform repeated tasks legacy bytecode performs `PUSHn ... JUMP` which is costly and this sequence does not meet added **code validation** requirements.

`CALLF` - switch to the code section in the code section body (image above), start executing that code 
```
code_section_index = read_uint16_be(current_code[pc+1])
perform stack overflow check
return_stack.push({current_code_idx, PC_post_intruction, operand_stack.height - types[code_section_index].inputs}) 
current_code_index = code_section_index
set PC to 0
```
`return_stack` is a stack of items representing execution state to return to after function execution is finished. Limited to 1024 items. (Save current execution state)

```
return_stack:
    code_section_index
    offset
    stack_height
```
`code_section_index` code's index in the code section body in the image above \
`offset` program counter to start from in the code  \
`stack_height` calling function stack height

`RETF` - return from the code section, get back to the code section where `CALLF` was executed
```
val = return_stack.pop()
current_code_idx, pc = val.code_section_index, val.offset
```

Opcdes deprecated: `JUMP`, `JUMPI` \
Possible opcode deprecation: `JUMPDEST` 

### EIP-5450: Stack Validation

Perform stack validation on the bytecode (code section) at the deploy time, not at the run time as it is in legacy bytecode. This is done by checking number of stack items each instruction requires (underflow) and by checking maximum allowed stack height for the instruction to execute (overflow).

Additional checks on `CALLF`, `RETF` \
`CALLF` check if `type_section[immediate_arg].inputs` are less then `acc_stack_height` (for underflow) and check if `type_section[immediate_arg].outputs` + `acc_stack_height` is less then `STACK_SIZE_LIMIT = 1024` (for overflow) \
`RETF` check if `type_section[current_section].outputs` is equal to `acc_stack_height` \



### EIP-6206: JUMPF instruction

`JUMPF` Jump to a code section without adding a new return stack frame.

It is common for functions to make a call at the end of the routine only to then return. `JUMPF` optimizes this behavior by changing code sections without needing to update the return stack

Warks the same as `CALLF` except that `JUMPF` does not push to `return_stack`.

The code section must be `non-returning`. This can be checked by reading `type_section[i].outputs == 0x80` \
(The first code section MUST have 0 inputs and be non-returning)

### EIP-7480: Data section access instructions

Four new instrutions are introduced, that allow to read EOF container’s data section: 

`DATALOAD` Pushes 32-byte word to stack from EOF container's data section \
`DATALOADN` Pushes 32-byte immediate argument word to stack
 from EOF container's data section \
`DATASIZE` Pushes data section size to the stack \
`DATACOPY` Copies a segment of data section to memory 

### TBA: Contract Creation

`CREATE3` Create a new account with associated code (EOF container/sub-container)

```
initcontainer_index = read_uint8_be(code[pc+1])
endowment, salt, input_offset, input_size = stack.pop(4)
initcontainer = get_sub_container(initcontainer_index)
if initcontainer_size > MAX_INITCODE_SIZE:
  abort
return_val, addr, return_gas, success = vm.create3(
                                        initcontainer, gas, salt, endowment)
success ? stack.push(addr) : stack.push(0)
contract.gas += return_gas
```

`CREATE4` Create a new account with associated code (EOF container).
Expects initcontainer to be in transaction context. \
Introduces new transaction type which has a new filed `initcodes` of type `[][]byte`.
Does the same thing as `CREATE3` except it loads initcontainer from transaction context.

successful `vm.create3/vm.create4` execution ends with initcode executing `RETURNCONTRACT`

`RETURNCONTRACT` Fetch sub-container and append data to it

```
deploy_container_index = read_uint8_be(code[pc+1])
aux_data_offset = stack.pop()
deploy_container = get_sub_container(deploy_container_index)
size = (data_section_offset + data_section_size) - deploy_container.size()
aux_data = memory[aux_data_offset:aux_data_offset+size]
deploy_container.append(aux_data)
exec_scope.deploy_container = deploy_container # will set this as code for newly create account
```