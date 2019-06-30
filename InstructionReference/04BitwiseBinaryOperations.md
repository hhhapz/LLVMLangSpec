### Bitwise Binary Operations

Bitwise binary operators are used to do various forms of bit-twiddling
in a program. They are generally very efficient instructions and can
commonly be strength reduced from other instructions. They require two
operands of the same type, execute an operation on them, and produce a
single value. The resulting value is the same type as its operands.

#### \'`shl`\' Instruction

##### Syntax:

    <result> = shl <ty> <op1>, <op2>           ; yields ty:result
    <result> = shl nuw <ty> <op1>, <op2>       ; yields ty:result
    <result> = shl nsw <ty> <op1>, <op2>       ; yields ty:result
    <result> = shl nuw nsw <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`shl`\' instruction returns the first operand shifted to the left
a specified number of bits.

##### Arguments:

Both arguments to the \'`shl`\' instruction must be the same
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer type.
\'`op2`\' is treated as an unsigned value.

##### Semantics:

The value produced is `op1` \* 2^op2^ mod 2^n^, where `n` is the width
of the result. If `op2` is (statically or dynamically) equal to or
larger than the number of bits in `op1`, this instruction returns a
`poison value <poisonvalues>`{.interpreted-text role="ref"}. If the
arguments are vectors, each vector element of `op1` is shifted by the
corresponding shift amount in `op2`.

If the `nuw` keyword is present, then the shift produces a poison value
if it shifts out any non-zero bits. If the `nsw` keyword is present,
then the shift produces a poison value if it shifts out any bits that
disagree with the resultant sign bit.

##### Example:

``` {.text}
<result> = shl i32 4, %var   ; yields i32: 4 << %var
<result> = shl i32 4, 2      ; yields i32: 16
<result> = shl i32 1, 10     ; yields i32: 1024
<result> = shl i32 1, 32     ; undefined
<result> = shl <2 x i32> < i32 1, i32 1>, < i32 1, i32 2>   ; yields: result=<2 x i32> < i32 2, i32 4>
```

#### \'`lshr`\' Instruction

##### Syntax:

    <result> = lshr <ty> <op1>, <op2>         ; yields ty:result
    <result> = lshr exact <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`lshr`\' instruction (logical shift right) returns the first
operand shifted to the right a specified number of bits with zero fill.

##### Arguments:

Both arguments to the \'`lshr`\' instruction must be the same
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer type.
\'`op2`\' is treated as an unsigned value.

##### Semantics:

This instruction always performs a logical shift right operation. The
most significant bits of the result will be filled with zero bits after
the shift. If `op2` is (statically or dynamically) equal to or larger
than the number of bits in `op1`, this instruction returns a `poison
value <poisonvalues>`{.interpreted-text role="ref"}. If the arguments
are vectors, each vector element of `op1` is shifted by the
corresponding shift amount in `op2`.

If the `exact` keyword is present, the result value of the `lshr` is a
poison value if any of the bits shifted out are non-zero.

##### Example:

``` {.text}
<result> = lshr i32 4, 1   ; yields i32:result = 2
<result> = lshr i32 4, 2   ; yields i32:result = 1
<result> = lshr i8  4, 3   ; yields i8:result = 0
<result> = lshr i8 -2, 1   ; yields i8:result = 0x7F
<result> = lshr i32 1, 32  ; undefined
<result> = lshr <2 x i32> < i32 -2, i32 4>, < i32 1, i32 2>   ; yields: result=<2 x i32> < i32 0x7FFFFFFF, i32 1>
```

#### \'`ashr`\' Instruction

##### Syntax:

    <result> = ashr <ty> <op1>, <op2>         ; yields ty:result
    <result> = ashr exact <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`ashr`\' instruction (arithmetic shift right) returns the first
operand shifted to the right a specified number of bits with sign
extension.

##### Arguments:

Both arguments to the \'`ashr`\' instruction must be the same
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer type.
\'`op2`\' is treated as an unsigned value.

##### Semantics:

This instruction always performs an arithmetic shift right operation,
The most significant bits of the result will be filled with the sign bit
of `op1`. If `op2` is (statically or dynamically) equal to or larger
than the number of bits in `op1`, this instruction returns a `poison
value <poisonvalues>`{.interpreted-text role="ref"}. If the arguments
are vectors, each vector element of `op1` is shifted by the
corresponding shift amount in `op2`.

If the `exact` keyword is present, the result value of the `ashr` is a
poison value if any of the bits shifted out are non-zero.

##### Example:

``` {.text}
<result> = ashr i32 4, 1   ; yields i32:result = 2
<result> = ashr i32 4, 2   ; yields i32:result = 1
<result> = ashr i8  4, 3   ; yields i8:result = 0
<result> = ashr i8 -2, 1   ; yields i8:result = -1
<result> = ashr i32 1, 32  ; undefined
<result> = ashr <2 x i32> < i32 -2, i32 4>, < i32 1, i32 3>   ; yields: result=<2 x i32> < i32 -1, i32 0>
```

#### \'`and`\' Instruction

##### Syntax:

    <result> = and <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`and`\' instruction returns the bitwise logical and of its two
operands.

##### Arguments:

The two arguments to the \'`and`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The truth table used for the \'`and`\' instruction is:

+-----+-----+-----+
| In0 | In1 | Out |
+-----+-----+-----+
| > 0 | > 0 | > 0 |
+-----+-----+-----+
| > 0 | > 1 | > 0 |
+-----+-----+-----+
| > 1 | > 0 | > 0 |
+-----+-----+-----+
| > 1 | > 1 | > 1 |
+-----+-----+-----+

##### Example:

``` {.text}
<result> = and i32 4, %var         ; yields i32:result = 4 & %var
<result> = and i32 15, 40          ; yields i32:result = 8
<result> = and i32 4, 8            ; yields i32:result = 0
```

#### \'`or`\' Instruction

##### Syntax:

    <result> = or <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`or`\' instruction returns the bitwise logical inclusive or of its
two operands.

##### Arguments:

The two arguments to the \'`or`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The truth table used for the \'`or`\' instruction is:

+-----+-----+-----+
| In0 | In1 | Out |
+-----+-----+-----+
| > 0 | > 0 | > 0 |
+-----+-----+-----+
| > 0 | > 1 | > 1 |
+-----+-----+-----+
| > 1 | > 0 | > 1 |
+-----+-----+-----+
| > 1 | > 1 | > 1 |
+-----+-----+-----+

##### Example:

    <result> = or i32 4, %var         ; yields i32:result = 4 | %var
    <result> = or i32 15, 40          ; yields i32:result = 47
    <result> = or i32 4, 8            ; yields i32:result = 12

#### \'`xor`\' Instruction

##### Syntax:

    <result> = xor <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`xor`\' instruction returns the bitwise logical exclusive or of
its two operands. The `xor` is used to implement the \"one\'s
complement\" operation, which is the \"\~\" operator in C.

##### Arguments:

The two arguments to the \'`xor`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The truth table used for the \'`xor`\' instruction is:

+-----+-----+-----+
| In0 | In1 | Out |
+-----+-----+-----+
| > 0 | > 0 | > 0 |
+-----+-----+-----+
| > 0 | > 1 | > 1 |
+-----+-----+-----+
| > 1 | > 0 | > 1 |
+-----+-----+-----+
| > 1 | > 1 | > 0 |
+-----+-----+-----+

##### Example:

``` {.text}
<result> = xor i32 4, %var         ; yields i32:result = 4 ^ %var
<result> = xor i32 15, 40          ; yields i32:result = 39
<result> = xor i32 4, 8            ; yields i32:result = 12
<result> = xor i32 %V, -1          ; yields i32:result = ~%V
```

