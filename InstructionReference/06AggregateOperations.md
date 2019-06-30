### Aggregate Operations

LLVM supports several instructions for working with
`aggregate <t_aggregate>`{.interpreted-text role="ref"} values.

#### \'`extractvalue`\' Instruction

##### Syntax:

    <result> = extractvalue <aggregate type> <val>, <idx>{, <idx>}*

##### Overview:

The \'`extractvalue`\' instruction extracts the value of a member field
from an `aggregate <t_aggregate>`{.interpreted-text role="ref"} value.

##### Arguments:

The first operand of an \'`extractvalue`\' instruction is a value of
`struct <t_struct>`{.interpreted-text role="ref"} or
`array <t_array>`{.interpreted-text role="ref"} type. The other operands
are constant indices to specify which value to extract in a similar
manner as indices in a \'`getelementptr`\' instruction.

The major differences to `getelementptr` indexing are:

-   Since the value being indexed is not a pointer, the first index is
    omitted and assumed to be zero.
-   At least one index must be specified.
-   Not only struct indices but also array indices must be in bounds.

##### Semantics:

The result is the value at the position in the aggregate specified by
the index operands.

##### Example:

``` {.text}
<result> = extractvalue {i32, float} %agg, 0    ; yields i32
```

#### \'`insertvalue`\' Instruction

##### Syntax:

    <result> = insertvalue <aggregate type> <val>, <ty> <elt>, <idx>{, <idx>}*    ; yields <aggregate type>

##### Overview:

The \'`insertvalue`\' instruction inserts a value into a member field in
an `aggregate <t_aggregate>`{.interpreted-text role="ref"} value.

##### Arguments:

The first operand of an \'`insertvalue`\' instruction is a value of
`struct <t_struct>`{.interpreted-text role="ref"} or
`array <t_array>`{.interpreted-text role="ref"} type. The second operand
is a first-class value to insert. The following operands are constant
indices indicating the position at which to insert the value in a
similar manner as indices in a \'`extractvalue`\' instruction. The value
to insert must have the same type as the value identified by the
indices.

##### Semantics:

The result is an aggregate of the same type as `val`. Its value is that
of `val` except that the value at the position specified by the indices
is that of `elt`.

##### Example:

``` {.llvm}
%agg1 = insertvalue {i32, float} undef, i32 1, 0              ; yields {i32 1, float undef}
%agg2 = insertvalue {i32, float} %agg1, float %val, 1         ; yields {i32 1, float %val}
%agg3 = insertvalue {i32, {float}} undef, float %val, 1, 0    ; yields {i32 undef, {float %val}}
```

