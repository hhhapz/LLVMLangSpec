### Unary Operations

Unary operators require a single operand, execute an operation on it,
and produce a single value. The operand might represent multiple data,
as is the case with the `vector <t_vector>`{.interpreted-text
role="ref"} data type. The result value has the same type as its
operand.

#### \'`fneg`\' Instruction

##### Syntax:

    <result> = fneg [fast-math flags]* <ty> <op1>   ; yields ty:result

##### Overview:

The \'`fneg`\' instruction returns the negation of its operand.

##### Arguments:

The argument to the \'`fneg`\' instruction must be a
`floating-point <t_floating>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
values.

##### Semantics:

The value produced is a copy of the operand with its sign bit flipped.
This instruction can also take any number of `fast-math
flags <fastmath>`{.interpreted-text role="ref"}, which are optimization
hints to enable otherwise unsafe floating-point optimizations:

##### Example:

``` {.text}
<result> = fneg float %val          ; yields float:result = -%var
```

