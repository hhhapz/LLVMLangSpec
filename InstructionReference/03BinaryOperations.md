### Binary Operations {#binaryops}

Binary operators are used to do most of the computation in a program.
They require two operands of the same type, execute an operation on
them, and produce a single value. The operands might represent multiple
data, as is the case with the `vector <t_vector>`{.interpreted-text
role="ref"} data type. The result value has the same type as its
operands.

There are several different binary operators:

#### \'`add`\' Instruction {#i_add}

##### Syntax:

    <result> = add <ty> <op1>, <op2>          ; yields ty:result
    <result> = add nuw <ty> <op1>, <op2>      ; yields ty:result
    <result> = add nsw <ty> <op1>, <op2>      ; yields ty:result
    <result> = add nuw nsw <ty> <op1>, <op2>  ; yields ty:result

##### Overview:

The \'`add`\' instruction returns the sum of its two operands.

##### Arguments:

The two arguments to the \'`add`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The value produced is the integer sum of the two operands.

If the sum has unsigned overflow, the result returned is the
mathematical result modulo 2^n^, where n is the bit width of the result.

Because LLVM integers use a two\'s complement representation, this
instruction is appropriate for both signed and unsigned integers.

`nuw` and `nsw` stand for \"No Unsigned Wrap\" and \"No Signed Wrap\",
respectively. If the `nuw` and/or `nsw` keywords are present, the result
value of the `add` is a `poison value <poisonvalues>`{.interpreted-text
role="ref"} if unsigned and/or signed overflow, respectively, occurs.

##### Example:

``` {.text}
<result> = add i32 4, %var          ; yields i32:result = 4 + %var
```

#### \'`fadd`\' Instruction {#i_fadd}

##### Syntax:

    <result> = fadd [fast-math flags]* <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`fadd`\' instruction returns the sum of its two operands.

##### Arguments:

The two arguments to the \'`fadd`\' instruction must be
`floating-point <t_floating>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
values. Both arguments must have identical types.

##### Semantics:

The value produced is the floating-point sum of the two operands. This
instruction is assumed to execute in the default `floating-point
environment <floatenv>`{.interpreted-text role="ref"}. This instruction
can also take any number of `fast-math
flags <fastmath>`{.interpreted-text role="ref"}, which are optimization
hints to enable otherwise unsafe floating-point optimizations:

##### Example:

``` {.text}
<result> = fadd float 4.0, %var          ; yields float:result = 4.0 + %var
```

#### \'`sub`\' Instruction

##### Syntax:

    <result> = sub <ty> <op1>, <op2>          ; yields ty:result
    <result> = sub nuw <ty> <op1>, <op2>      ; yields ty:result
    <result> = sub nsw <ty> <op1>, <op2>      ; yields ty:result
    <result> = sub nuw nsw <ty> <op1>, <op2>  ; yields ty:result

##### Overview:

The \'`sub`\' instruction returns the difference of its two operands.

Note that the \'`sub`\' instruction is used to represent the \'`neg`\'
instruction present in most other intermediate representations.

##### Arguments:

The two arguments to the \'`sub`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The value produced is the integer difference of the two operands.

If the difference has unsigned overflow, the result returned is the
mathematical result modulo 2^n^, where n is the bit width of the result.

Because LLVM integers use a two\'s complement representation, this
instruction is appropriate for both signed and unsigned integers.

`nuw` and `nsw` stand for \"No Unsigned Wrap\" and \"No Signed Wrap\",
respectively. If the `nuw` and/or `nsw` keywords are present, the result
value of the `sub` is a `poison value <poisonvalues>`{.interpreted-text
role="ref"} if unsigned and/or signed overflow, respectively, occurs.

##### Example:

``` {.text}
<result> = sub i32 4, %var          ; yields i32:result = 4 - %var
<result> = sub i32 0, %val          ; yields i32:result = -%var
```

#### \'`fsub`\' Instruction {#i_fsub}

##### Syntax:

    <result> = fsub [fast-math flags]* <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`fsub`\' instruction returns the difference of its two operands.

##### Arguments:

The two arguments to the \'`fsub`\' instruction must be
`floating-point <t_floating>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
values. Both arguments must have identical types.

##### Semantics:

The value produced is the floating-point difference of the two operands.
This instruction is assumed to execute in the default `floating-point
environment <floatenv>`{.interpreted-text role="ref"}. This instruction
can also take any number of `fast-math
flags <fastmath>`{.interpreted-text role="ref"}, which are optimization
hints to enable otherwise unsafe floating-point optimizations:

##### Example:

``` {.text}
<result> = fsub float 4.0, %var           ; yields float:result = 4.0 - %var
<result> = fsub float -0.0, %val          ; yields float:result = -%var
```

#### \'`mul`\' Instruction

##### Syntax:

    <result> = mul <ty> <op1>, <op2>          ; yields ty:result
    <result> = mul nuw <ty> <op1>, <op2>      ; yields ty:result
    <result> = mul nsw <ty> <op1>, <op2>      ; yields ty:result
    <result> = mul nuw nsw <ty> <op1>, <op2>  ; yields ty:result

##### Overview:

The \'`mul`\' instruction returns the product of its two operands.

##### Arguments:

The two arguments to the \'`mul`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The value produced is the integer product of the two operands.

If the result of the multiplication has unsigned overflow, the result
returned is the mathematical result modulo 2^n^, where n is the bit
width of the result.

Because LLVM integers use a two\'s complement representation, and the
result is the same width as the operands, this instruction returns the
correct result for both signed and unsigned integers. If a full product
(e.g. `i32` \* `i32` -\> `i64`) is needed, the operands should be
sign-extended or zero-extended as appropriate to the width of the full
product.

`nuw` and `nsw` stand for \"No Unsigned Wrap\" and \"No Signed Wrap\",
respectively. If the `nuw` and/or `nsw` keywords are present, the result
value of the `mul` is a `poison value <poisonvalues>`{.interpreted-text
role="ref"} if unsigned and/or signed overflow, respectively, occurs.

##### Example:

``` {.text}
<result> = mul i32 4, %var          ; yields i32:result = 4 * %var
```

#### \'`fmul`\' Instruction {#i_fmul}

##### Syntax:

    <result> = fmul [fast-math flags]* <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`fmul`\' instruction returns the product of its two operands.

##### Arguments:

The two arguments to the \'`fmul`\' instruction must be
`floating-point <t_floating>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
values. Both arguments must have identical types.

##### Semantics:

The value produced is the floating-point product of the two operands.
This instruction is assumed to execute in the default `floating-point
environment <floatenv>`{.interpreted-text role="ref"}. This instruction
can also take any number of `fast-math
flags <fastmath>`{.interpreted-text role="ref"}, which are optimization
hints to enable otherwise unsafe floating-point optimizations:

##### Example:

``` {.text}
<result> = fmul float 4.0, %var          ; yields float:result = 4.0 * %var
```

#### \'`udiv`\' Instruction

##### Syntax:

    <result> = udiv <ty> <op1>, <op2>         ; yields ty:result
    <result> = udiv exact <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`udiv`\' instruction returns the quotient of its two operands.

##### Arguments:

The two arguments to the \'`udiv`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The value produced is the unsigned integer quotient of the two operands.

Note that unsigned integer division and signed integer division are
distinct operations; for signed integer division, use \'`sdiv`\'.

Division by zero is undefined behavior. For vectors, if any element of
the divisor is zero, the operation has undefined behavior.

If the `exact` keyword is present, the result value of the `udiv` is a
`poison value <poisonvalues>`{.interpreted-text role="ref"} if %op1 is
not a multiple of %op2 (as such, \"((a udiv exact b) mul b) == a\").

##### Example:

``` {.text}
<result> = udiv i32 4, %var          ; yields i32:result = 4 / %var
```

#### \'`sdiv`\' Instruction

##### Syntax:

    <result> = sdiv <ty> <op1>, <op2>         ; yields ty:result
    <result> = sdiv exact <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`sdiv`\' instruction returns the quotient of its two operands.

##### Arguments:

The two arguments to the \'`sdiv`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

The value produced is the signed integer quotient of the two operands
rounded towards zero.

Note that signed integer division and unsigned integer division are
distinct operations; for unsigned integer division, use \'`udiv`\'.

Division by zero is undefined behavior. For vectors, if any element of
the divisor is zero, the operation has undefined behavior. Overflow also
leads to undefined behavior; this is a rare case, but can occur, for
example, by doing a 32-bit division of -2147483648 by -1.

If the `exact` keyword is present, the result value of the `sdiv` is a
`poison value <poisonvalues>`{.interpreted-text role="ref"} if the
result would be rounded.

##### Example:

``` {.text}
<result> = sdiv i32 4, %var          ; yields i32:result = 4 / %var
```

#### \'`fdiv`\' Instruction {#i_fdiv}

##### Syntax:

    <result> = fdiv [fast-math flags]* <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`fdiv`\' instruction returns the quotient of its two operands.

##### Arguments:

The two arguments to the \'`fdiv`\' instruction must be
`floating-point <t_floating>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
values. Both arguments must have identical types.

##### Semantics:

The value produced is the floating-point quotient of the two operands.
This instruction is assumed to execute in the default `floating-point
environment <floatenv>`{.interpreted-text role="ref"}. This instruction
can also take any number of `fast-math
flags <fastmath>`{.interpreted-text role="ref"}, which are optimization
hints to enable otherwise unsafe floating-point optimizations:

##### Example:

``` {.text}
<result> = fdiv float 4.0, %var          ; yields float:result = 4.0 / %var
```

#### \'`urem`\' Instruction

##### Syntax:

    <result> = urem <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`urem`\' instruction returns the remainder from the unsigned
division of its two arguments.

##### Arguments:

The two arguments to the \'`urem`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

This instruction returns the unsigned integer *remainder* of a division.
This instruction always performs an unsigned division to get the
remainder.

Note that unsigned integer remainder and signed integer remainder are
distinct operations; for signed integer remainder, use \'`srem`\'.

Taking the remainder of a division by zero is undefined behavior. For
vectors, if any element of the divisor is zero, the operation has
undefined behavior.

##### Example:

``` {.text}
<result> = urem i32 4, %var          ; yields i32:result = 4 % %var
```

#### \'`srem`\' Instruction

##### Syntax:

    <result> = srem <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`srem`\' instruction returns the remainder from the signed
division of its two operands. This instruction can also take
`vector <t_vector>`{.interpreted-text role="ref"} versions of the values
in which case the elements must be integers.

##### Arguments:

The two arguments to the \'`srem`\' instruction must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of integer values.
Both arguments must have identical types.

##### Semantics:

This instruction returns the *remainder* of a division (where the result
is either zero or has the same sign as the dividend, `op1`), not the
*modulo* operator (where the result is either zero or has the same sign
as the divisor, `op2`) of a value. For more information about the
difference, see [The Math
Forum](http://mathforum.org/dr.math/problems/anne.4.28.99.html). For a
table of how this is implemented in various languages, please see
[Wikipedia: modulo
operation](http://en.wikipedia.org/wiki/Modulo_operation).

Note that signed integer remainder and unsigned integer remainder are
distinct operations; for unsigned integer remainder, use \'`urem`\'.

Taking the remainder of a division by zero is undefined behavior. For
vectors, if any element of the divisor is zero, the operation has
undefined behavior. Overflow also leads to undefined behavior; this is a
rare case, but can occur, for example, by taking the remainder of a
32-bit division of -2147483648 by -1. (The remainder doesn\'t actually
overflow, but this rule lets srem be implemented using instructions that
return both the result of the division and the remainder.)

##### Example:

``` {.text}
<result> = srem i32 4, %var          ; yields i32:result = 4 % %var
```

#### \'`frem`\' Instruction {#i_frem}

##### Syntax:

    <result> = frem [fast-math flags]* <ty> <op1>, <op2>   ; yields ty:result

##### Overview:

The \'`frem`\' instruction returns the remainder from the division of
its two operands.

##### Arguments:

The two arguments to the \'`frem`\' instruction must be
`floating-point <t_floating>`{.interpreted-text role="ref"} or
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
values. Both arguments must have identical types.

##### Semantics:

The value produced is the floating-point remainder of the two operands.
This is the same output as a libm \'`fmod`\' function, but without any
possibility of setting `errno`. The remainder has the same sign as the
dividend. This instruction is assumed to execute in the default
`floating-point
environment <floatenv>`{.interpreted-text role="ref"}. This instruction
can also take any number of `fast-math
flags <fastmath>`{.interpreted-text role="ref"}, which are optimization
hints to enable otherwise unsafe floating-point optimizations:

##### Example:

``` {.text}
<result> = frem float 4.0, %var          ; yields float:result = 4.0 % %var
```

