- [Basic](#basic)
  - [OutputOperands and InputOperands](#outputoperands-and-inputoperands)
    - [asmSymbolicName and position based varaible](#asmsymbolicname-and-position-based-varaible)
    - [constraint](#constraint)
      - [OutputOperands](#outputoperands)
      - [InputOperands](#inputoperands)
    - [cvariablename](#cvariablename)
  - [Clobbers](#clobbers)
- [Reference](#reference)

# Basic
- Basic form #1
  ``` C
  __asm__ __volatile__(Instructions
                     : OutputOperands
                   [ : InputOperands
                   [ : Clobbers ] ])
  ```
- Basic form #2
  ``` C
  __asm__ __volatile(Instructions
                   : OutputOperands
                   : InputOperands
                   : Clobbers
                   : GotoLabels)
  ```

Form #1 is appeared more often.

## OutputOperands and InputOperands
The output operands should be described as the following template
``` C
[ [asmSymbolicName] ] constraint (cvariablename)
```

### asmSymbolicName and position based varaible
asmSymbolicName is rarely used. If it is used, we should see something like 
`%[name]` in the instructions part.

Usually, programmer uses the default **zero-based** position operands. It means that 
in the instruction part, %0 corresponding to the first output operands and %1
corresponding to the second output operands and so on.

### constraint
#### OutputOperands
Output constraints must **begin with a preifx**
- either = (a variable overwriting an existing value) 
- or + (when reading and writing).

After a prefix, it must be **followed by one or more condition constraints**. Common constraints include 
- r for register
- m for memory. 

When you list more than one possible location (for example, "=rm"), the compiler chooses the most efficient one based on the current context.

#### InputOperands
Input constraint strings may not begin with either = or +.

When you list more than one possible location (for example, "irm"), the compiler chooses the most efficient one based on the current context.

Input constraints can also be digits (for example, "0"). This indicates that the specified input must be in the same place as the output constraint at the (zero-based) index in the output constraint list. When using asmSymbolicName syntax for the output operands, you may use these names (enclosed in brackets []) instead of digits.

### cvariablename
Specifies a **C lvalue expression** to hold the output, typically a variable name. The enclosing parentheses are a required part of the syntax.

## Clobbers
When the compiler selects which registers to use to represent input and output operands, it does not use any of the clobbered registers. As a result, clobbered registers are available for any use in the assembler code.

There are 2 special clobbers
- cc
        
  The "cc" clobber indicates that the assembler code modifies the flags register. On some machines, GCC represents the condition codes as a specific hardware register; "cc" serves to name this register. On other machines, condition code handling is different, and specifying "cc" has no effect. But it is valid no matter what the target.

  -> Must add this line if the assembly instruction modify flag registers
- memory
  
  The "memory" clobber tells the compiler that the assembly code performs memory reads or writes to items other than those listed in the input and output operands (for example, accessing the memory pointed to by one of the input parameters). To ensure memory contains correct values, GCC may need to flush specific register values to memory before executing the asm. Further, the compiler does not assume that any values read from memory before an asm remain unchanged after that asm ; it reloads them as needed. Using the "memory" clobber effectively forms a read/write memory barrier for the compiler.
  
  Note that this clobber does not prevent the processor from doing speculative reads past the asm statement. To prevent that, you need processor-specific fence instructions.

  -> compilation time memory barrier

# Reference
[GNU how to use inline assembly language in C code](https://gcc.gnu.org/onlinedocs/gcc/extensions-to-the-c-language-family/how-to-use-inline-assembly-language-in-c-code.html#outputoperands)