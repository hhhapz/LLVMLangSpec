Type System {#typesystem}
-----------

The LLVM type system is one of the most important features of the
intermediate representation. Being typed enables a number of
optimizations to be performed on the intermediate representation
directly, without having to do extra analyses on the side before the
transformation. A strong type system makes it easier to read the
generated code and enables novel analyses and transformations that are
not feasible to perform on normal three address code representations.

### Void Type {#t_void}

Overview

:   

The void type does not represent any value and has no size.

Syntax

:   

<!-- -->

    void

### Function Type {#t_function}

Overview

:   

The function type can be thought of as a function signature. It consists
of a return type and a list of formal parameter types. The return type
of a function type is a void type or first class type \-\-- except for
`label <t_label>`{.interpreted-text role="ref"} and
`metadata <t_metadata>`{.interpreted-text role="ref"} types.

Syntax

:   

<!-- -->

    <returntype> (<parameter list>)

\...where \'`<parameter list>`\' is a comma-separated list of type
specifiers. Optionally, the parameter list may include a type `...`,
which indicates that the function takes a variable number of arguments.
Variable argument functions can access their arguments with the
`variable argument
handling intrinsic <int_varargs>`{.interpreted-text role="ref"}
functions. \'`<returntype>`\' is any type except
`label <t_label>`{.interpreted-text role="ref"} and
`metadata <t_metadata>`{.interpreted-text role="ref"}.

Examples

:   

  ------------------------ ----------------------------------------------------------
  `i32 (i32)`              function taking an `i32`, returning an `i32`

  `float (i16, i32 *) *`   `Pointer <t_pointer>`{.interpreted-text role="ref"} to a
                           function that takes an `i16` and a
                           `pointer <t_pointer>`{.interpreted-text role="ref"} to
                           `i32`, returning `float`.

  `i32 (i8*, ...)`         A vararg function that takes at least one
                           `pointer <t_pointer>`{.interpreted-text role="ref"} to
                           `i8` (char in C), which returns an integer. This is the
                           signature for `printf` in LLVM.

  `{i32, i32} (i32)`       A function taking an `i32`, returning a
                           `structure <t_struct>`{.interpreted-text role="ref"}
                           containing two `i32` values
  ------------------------ ----------------------------------------------------------

### First Class Types {#t_firstclass}

The `first class <t_firstclass>`{.interpreted-text role="ref"} types are
perhaps the most important. Values of these types are the only ones
which can be produced by instructions.

#### Single Value Types {#t_single_value}

These are the types that are valid in registers from CodeGen\'s
perspective.

##### Integer Type {#t_integer}

Overview

:   

The integer type is a very simple type that simply specifies an
arbitrary bit width for the integer type desired. Any bit width from 1
bit to 2^23^-1 (about 8 million) can be specified.

Syntax

:   

<!-- -->

    iN

The number of bits the integer will occupy is specified by the `N`
value.

###### Examples:

  ---------------- ------------------------------------------------
  `i1`             a single-bit integer.

  `i32`            a 32-bit integer.

  `i1942652`       a really big integer of over 1 million bits.
  ---------------- ------------------------------------------------

##### Floating-Point Types {#t_floating}

  Type          Description
  ------------- -------------------------------------------------
  `half`        16-bit floating-point value
  `float`       32-bit floating-point value
  `double`      64-bit floating-point value
  `fp128`       128-bit floating-point value (112-bit mantissa)
  `x86_fp80`    80-bit floating-point value (X87)
  `ppc_fp128`   128-bit floating-point value (two 64-bits)

The binary format of half, float, double, and fp128 correspond to the
IEEE-754-2008 specifications for binary16, binary32, binary64, and
binary128 respectively.

##### X86\_mmx Type

Overview

:   

The x86\_mmx type represents a value held in an MMX register on an x86
machine. The operations allowed on it are quite limited: parameters and
return values, load and store, and bitcast. User-specified MMX
instructions are represented as intrinsic or asm calls with arguments
and/or results of this type. There are no arrays, vectors or constants
of this type.

Syntax

:   

<!-- -->

    x86_mmx

##### Pointer Type {#t_pointer}

Overview

:   

The pointer type is used to specify memory locations. Pointers are
commonly used to reference objects in memory.

Pointer types may have an optional address space attribute defining the
numbered address space where the pointed-to object resides. The default
address space is number zero. The semantics of non-zero address spaces
are target-specific.

Note that LLVM does not permit pointers to void (`void*`) nor does it
permit pointers to labels (`label*`). Use `i8*` instead.

Syntax

:   

<!-- -->

    <type> *

Examples

:   

  --------------------- ---------------------------------------------------------
  `[4 x i32]*`          A `pointer <t_pointer>`{.interpreted-text role="ref"} to
                        `array <t_array>`{.interpreted-text role="ref"} of four
                        `i32` values.

  `i32 (i32*) *`        A `pointer <t_pointer>`{.interpreted-text role="ref"} to
                        a `function <t_function>`{.interpreted-text role="ref"}
                        that takes an `i32*`, returning an `i32`.

  `i32 addrspace(5)*`   A `pointer <t_pointer>`{.interpreted-text role="ref"} to
                        an `i32` value that resides in address space \#5.
  --------------------- ---------------------------------------------------------

##### Vector Type {#t_vector}

Overview

:   

A vector type is a simple derived type that represents a vector of
elements. Vector types are used when multiple primitive data are
operated in parallel using a single instruction (SIMD). A vector type
requires a size (number of elements) and an underlying primitive data
type. Vector types are considered
`first class <t_firstclass>`{.interpreted-text role="ref"}.

Syntax

:   

<!-- -->

    < <# elements> x <elementtype> >

The number of elements is a constant integer value larger than 0;
elementtype may be any integer, floating-point or pointer type. Vectors
of size zero are not allowed.

Examples

:   

  ------------------- --------------------------------------------------
  `<4 x i32>`         Vector of 4 32-bit integer values.

  `<8 x float>`       Vector of 8 32-bit floating-point values.

  `<2 x i64>`         Vector of 2 64-bit integer values.

  `<4 x i64*>`        Vector of 4 pointers to 64-bit integer values.
  ------------------- --------------------------------------------------

#### Label Type {#t_label}

Overview

:   

The label type represents code labels.

Syntax

:   

<!-- -->

    label

#### Token Type {#t_token}

Overview

:   

The token type is used when a value is associated with an instruction
but all uses of the value must not attempt to introspect or obscure it.
As such, it is not appropriate to have a `phi <i_phi>`{.interpreted-text
role="ref"} or `select <i_select>`{.interpreted-text role="ref"} of type
token.

Syntax

:   

<!-- -->

    token

#### Metadata Type {#t_metadata}

Overview

:   

The metadata type represents embedded metadata. No derived types may be
created from metadata except for
`function <t_function>`{.interpreted-text role="ref"} arguments.

Syntax

:   

<!-- -->

    metadata

#### Aggregate Types {#t_aggregate}

Aggregate Types are a subset of derived types that can contain multiple
member types. `Arrays <t_array>`{.interpreted-text role="ref"} and
`structs <t_struct>`{.interpreted-text role="ref"} are aggregate types.
`Vectors <t_vector>`{.interpreted-text role="ref"} are not considered to
be aggregate types.

##### Array Type {#t_array}

Overview

:   

The array type is a very simple derived type that arranges elements
sequentially in memory. The array type requires a size (number of
elements) and an underlying data type.

Syntax

:   

<!-- -->

    [<# elements> x <elementtype>]

The number of elements is a constant integer value; `elementtype` may be
any type with a size.

Examples

:   

  ------------------ --------------------------------------
  `[40 x i32]`       Array of 40 32-bit integer values.

  `[41 x i32]`       Array of 41 32-bit integer values.

  `[4 x i8]`         Array of 4 8-bit integer values.
  ------------------ --------------------------------------

Here are some examples of multidimensional arrays:

  ------------------------- -----------------------------------------------
  `[3 x [4 x i32]]`         3x4 array of 32-bit integer values.

  `[12 x [10 x float]]`     12x10 array of single precision floating-point
                            values.

  `[2 x [3 x [4 x i16]]]`   2x3x4 array of 16-bit integer values.
  ------------------------- -----------------------------------------------

There is no restriction on indexing beyond the end of the array implied
by a static type (though there are restrictions on indexing beyond the
bounds of an allocated object in some cases). This means that
single-dimension \'variable sized array\' addressing can be implemented
in LLVM with a zero length array type. An implementation of \'pascal
style arrays\' in LLVM could use the type \"`{ i32, [0 x float]}`\", for
example.

##### Structure Type {#t_struct}

Overview

:   

The structure type is used to represent a collection of data members
together in memory. The elements of a structure may be any type that has
a size.

Structures in memory are accessed using \'`load`\' and \'`store`\' by
getting a pointer to a field with the \'`getelementptr`\' instruction.
Structures in registers are accessed using the \'`extractvalue`\' and
\'`insertvalue`\' instructions.

Structures may optionally be \"packed\" structures, which indicate that
the alignment of the struct is one byte, and that there is no padding
between the elements. In non-packed structs, padding between field types
is inserted as defined by the DataLayout string in the module, which is
required to match what the underlying code generator expects.

Structures can either be \"literal\" or \"identified\". A literal
structure is defined inline with other types (e.g. `{i32, i32}*`)
whereas identified types are always defined at the top level with a
name. Literal types are uniqued by their contents and can never be
recursive or opaque since there is no way to write one. Identified types
can be recursive, can be opaqued, and are never uniqued.

Syntax

:   

<!-- -->

    %T1 = type { <type list> }     ; Identified normal struct type
    %T2 = type <{ <type list> }>   ; Identified packed struct type

Examples

:   

  -------------------------- ------------------------------------------------------------
  `{ i32, i32, i32 }`        A triple of three `i32` values

  `{ float, i32 (i32) * }`   A pair, where the first element is a `float` and the second
                             element is a `pointer <t_pointer>`{.interpreted-text
                             role="ref"} to a `function <t_function>`{.interpreted-text
                             role="ref"} that takes an `i32`, returning an `i32`.

  `<{ i8, i32 }>`            A packed struct known to be 5 bytes in size.
  -------------------------- ------------------------------------------------------------

##### Opaque Structure Types {#t_opaque}

Overview

:   

Opaque structure types are used to represent named structure types that
do not have a body specified. This corresponds (for example) to the C
notion of a forward declared structure.

Syntax

:   

<!-- -->

    %X = type opaque
    %52 = type opaque

Examples

:   

  -------------- -------------------
  `opaque`       An opaque type.

  -------------- -------------------

