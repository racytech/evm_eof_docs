## EVM Object Foramt (EOF) 

### Included Proposals.
- [EIP-3540: Structured Bytecode and Versioning (EVM Object Format v1)](#eip-3540-structured-bytecode-and-versioning-evm-object-format-v1)
- [EIP-3670: Code Validation](#eip-3670-code-validation)
- [EIP-4200: Static relative jumps](#eip-4200-static-relative-jumps)
- [EIP-4750: Functions](#eip-4750-functions)
- [EIP-5450: Stack Validation](#eip-5450-stack-validation)
- EIP-6206: JUMPF instruction
- EIP-7480: Data section access instructions
- EIP-663: Unlimited SWAP and DUP instructions
- EIP-7069: Revamped CALL instructions (does not require EOF)
- TBA: Contract Creation
- TBA: Restrict code and gas introspection

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
    - validate max stack height
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