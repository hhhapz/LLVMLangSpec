### Vector Operations

LLVM supports several instructions to represent vector operations in a
target-independent manner. These instructions cover the element-access
and vector-specific operations needed to process vectors effectively.
While LLVM does directly support these vector operations, many
sophisticated algorithms will want to use target-specific intrinsics to
take full advantage of a specific target.

#### \'`extractelement`\' Instruction {#i_extractelement}

##### Syntax:

    <result> = extractelement <n x <ty>> <val>, <ty2> <idx>  ; yields <ty>

##### Overview:

The \'`extractelement`\' instruction extracts a single scalar element
from a vector at a specified index.

##### Arguments:

The first operand of an \'`extractelement`\' instruction is a value of
`vector <t_vector>`{.interpreted-text role="ref"} type. The second
operand is an index indicating the position from which to extract the
element. The index may be a variable of any integer type.

##### Semantics:

The result is a scalar of the same type as the element type of `val`.
Its value is the value at position `idx` of `val`. If `idx` exceeds the
length of `val`, the result is a
`poison value <poisonvalues>`{.interpreted-text role="ref"}.

##### Example:

``` {.text}
<result> = extractelement <4 x i32> %vec, i32 0    ; yields i32
```

#### \'`insertelement`\' Instruction {#i_insertelement}

##### Syntax:

    <result> = insertelement <n x <ty>> <val>, <ty> <elt>, <ty2> <idx>    ; yields <n x <ty>>

##### Overview:

The \'`insertelement`\' instruction inserts a scalar element into a
vector at a specified index.

##### Arguments:

The first operand of an \'`insertelement`\' instruction is a value of
`vector <t_vector>`{.interpreted-text role="ref"} type. The second
operand is a scalar value whose type must equal the element type of the
first operand. The third operand is an index indicating the position at
which to insert the value. The index may be a variable of any integer
type.

##### Semantics:

The result is a vector of the same type as `val`. Its element values are
those of `val` except at position `idx`, where it gets the value `elt`.
If `idx` exceeds the length of `val`, the result is a
`poison value <poisonvalues>`{.interpreted-text role="ref"}.

##### Example:

``` {.text}
<result> = insertelement <4 x i32> %vec, i32 1, i32 0    ; yields <4 x i32>
```

#### \'`shufflevector`\' Instruction {#i_shufflevector}

##### Syntax:

    <result> = shufflevector <n x <ty>> <v1>, <n x <ty>> <v2>, <m x i32> <mask>    ; yields <m x <ty>>

##### Overview:

The \'`shufflevector`\' instruction constructs a permutation of elements
from two input vectors, returning a vector with the same element type as
the input and length that is the same as the shuffle mask.

##### Arguments:

The first two operands of a \'`shufflevector`\' instruction are vectors
with the same type. The third argument is a shuffle mask whose element
type is always \'i32\'. The result of the instruction is a vector whose
length is the same as the shuffle mask and whose element type is the
same as the element type of the first two operands.

The shuffle mask operand is required to be a constant vector with either
constant integer or undef values.

##### Semantics:

The elements of the two input vectors are numbered from left to right
across both of the vectors. The shuffle mask operand specifies, for each
element of the result vector, which element of the two input vectors the
result element gets. If the shuffle mask is undef, the result vector is
undef. If any element of the mask operand is undef, that element of the
result is undef. If the shuffle mask selects an undef element from one
of the input vectors, the resulting element is undef.

##### Example:

``` {.text}
<result> = shufflevector <4 x i32> %v1, <4 x i32> %v2,
                        <4 x i32> <i32 0, i32 4, i32 1, i32 5>  ; yields <4 x i32>
<result> = shufflevector <4 x i32> %v1, <4 x i32> undef,
                        <4 x i32> <i32 0, i32 1, i32 2, i32 3>  ; yields <4 x i32> - Identity shuffle.
<result> = shufflevector <8 x i32> %v1, <8 x i32> undef,
                        <4 x i32> <i32 0, i32 1, i32 2, i32 3>  ; yields <4 x i32>
<result> = shufflevector <4 x i32> %v1, <4 x i32> %v2,
                        <8 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7 >  ; yields <8 x i32>
```

