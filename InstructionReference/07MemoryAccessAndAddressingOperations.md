### Memory Access and Addressing Operations

A key design point of an SSA-based representation is how it represents
memory. In LLVM, no memory locations are in SSA form, which makes things
very simple. This section describes how to read, write, and allocate
memory in LLVM.

#### \'`alloca`\' Instruction

##### Syntax:

    <result> = alloca [inalloca] <type> [, <ty> <NumElements>] [, align <alignment>] [, addrspace(<num>)]     ; yields type addrspace(num)*:result

##### Overview:

The \'`alloca`\' instruction allocates memory on the stack frame of the
currently executing function, to be automatically released when this
function returns to its caller. The object is always allocated in the
address space for allocas indicated in the datalayout.

##### Arguments:

The \'`alloca`\' instruction allocates `sizeof(<type>)*NumElements`
bytes of memory on the runtime stack, returning a pointer of the
appropriate type to the program. If \"NumElements\" is specified, it is
the number of elements allocated, otherwise \"NumElements\" is defaulted
to be one. If a constant alignment is specified, the value result of the
allocation is guaranteed to be aligned to at least that boundary. The
alignment may not be greater than `1 << 29`. If not specified, or if
zero, the target can choose to align the allocation on any convenient
boundary compatible with the type.

\'`type`\' may be any sized type.

##### Semantics:

Memory is allocated; a pointer is returned. The operation is undefined
if there is insufficient stack space for the allocation. \'`alloca`\'d
memory is automatically released when the function returns. The
\'`alloca`\' instruction is commonly used to represent automatic
variables that must have an address available. When the function returns
(either with the `ret` or `resume` instructions), the memory is
reclaimed. Allocating zero bytes is legal, but the returned pointer may
not be unique. The order in which memory is allocated (ie., which way
the stack grows) is not specified.

##### Example:

``` {.llvm}
%ptr = alloca i32                             ; yields i32*:ptr
%ptr = alloca i32, i32 4                      ; yields i32*:ptr
%ptr = alloca i32, i32 4, align 1024          ; yields i32*:ptr
%ptr = alloca i32, align 1024                 ; yields i32*:ptr
```

#### \'`load`\' Instruction

##### Syntax:

    <result> = load [volatile] <ty>, <ty>* <pointer>[, align <alignment>][, !nontemporal !<index>][, !invariant.load !<index>][, !invariant.group !<index>][, !nonnull !<index>][, !dereferenceable !<deref_bytes_node>][, !dereferenceable_or_null !<deref_bytes_node>][, !align !<align_node>]
    <result> = load atomic [volatile] <ty>, <ty>* <pointer> [syncscope("<target-scope>")] <ordering>, align <alignment> [, !invariant.group !<index>]
    !<index> = !{ i32 1 }
    !<deref_bytes_node> = !{i64 <dereferenceable_bytes>}
    !<align_node> = !{ i64 <value_alignment> }

##### Overview:

The \'`load`\' instruction is used to read from memory.

##### Arguments:

The argument to the `load` instruction specifies the memory address from
which to load. The type specified must be a
`first class <t_firstclass>`{.interpreted-text role="ref"} type of known
size (i.e. not containing an
`opaque structural type <t_opaque>`{.interpreted-text role="ref"}). If
the `load` is marked as `volatile`, then the optimizer is not allowed to
modify the number or order of execution of this `load` with other
`volatile operations <volatile>`{.interpreted-text role="ref"}.

If the `load` is marked as `atomic`, it takes an extra `ordering
<ordering>`{.interpreted-text role="ref"} and optional
`syncscope("<target-scope>")` argument. The `release` and `acq_rel`
orderings are not valid on `load` instructions. Atomic loads produce
`defined <memmodel>`{.interpreted-text role="ref"} results when they may
see multiple atomic stores. The type of the pointee must be an integer,
pointer, or floating-point type whose bit width is a power of two
greater than or equal to eight and less than or equal to a
target-specific size limit. `align` must be explicitly specified on
atomic loads, and the load has undefined behavior if the alignment is
not set to a value which is at least the size in bytes of the pointee.
`!nontemporal` does not have any defined semantics for atomic loads.

The optional constant `align` argument specifies the alignment of the
operation (that is, the alignment of the memory address). A value of 0
or an omitted `align` argument means that the operation has the ABI
alignment for the target. It is the responsibility of the code emitter
to ensure that the alignment information is correct. Overestimating the
alignment results in undefined behavior. Underestimating the alignment
may produce less efficient code. An alignment of 1 is always safe. The
maximum possible alignment is `1 << 29`. An alignment value higher than
the size of the loaded type implies memory up to the alignment value
bytes can be safely loaded without trapping in the default address
space. Access of the high bytes can interfere with debugging tools, so
should not be accessed if the function has the `sanitize_thread` or
`sanitize_address` attributes.

The optional `!nontemporal` metadata must reference a single metadata
name `<index>` corresponding to a metadata node with one `i32` entry of
value 1. The existence of the `!nontemporal` metadata on the instruction
tells the optimizer and code generator that this load is not expected to
be reused in the cache. The code generator may select special
instructions to save cache bandwidth, such as the `MOVNT` instruction on
x86.

The optional `!invariant.load` metadata must reference a single metadata
name `<index>` corresponding to a metadata node with no entries. If a
load instruction tagged with the `!invariant.load` metadata is executed,
the optimizer may assume the memory location referenced by the load
contains the same value at all points in the program where the memory
location is known to be dereferenceable; otherwise, the behavior is
undefined.

The optional `!invariant.group` metadata must reference a single metadata name

`<index>` corresponding to a metadata node with no entries. See
    `invariant.group` metadata.

The optional `!nonnull` metadata must reference a single metadata name
`<index>` corresponding to a metadata node with no entries. The
existence of the `!nonnull` metadata on the instruction tells the
optimizer that the value loaded is known to never be null. If the value
is null at runtime, the behavior is undefined. This is analogous to the
`nonnull` attribute on parameters and return values. This metadata can
only be applied to loads of a pointer type.

The optional `!dereferenceable` metadata must reference a single
metadata name `<deref_bytes_node>` corresponding to a metadata node with
one `i64` entry. The existence of the `!dereferenceable` metadata on the
instruction tells the optimizer that the value loaded is known to be
dereferenceable. The number of bytes known to be dereferenceable is
specified by the integer value in the metadata node. This is analogous
to the \'\'dereferenceable\'\' attribute on parameters and return
values. This metadata can only be applied to loads of a pointer type.

The optional `!dereferenceable_or_null` metadata must reference a single
metadata name `<deref_bytes_node>` corresponding to a metadata node with
one `i64` entry. The existence of the `!dereferenceable_or_null`
metadata on the instruction tells the optimizer that the value loaded is
known to be either dereferenceable or null. The number of bytes known to
be dereferenceable is specified by the integer value in the metadata
node. This is analogous to the \'\'dereferenceable\_or\_null\'\'
attribute on parameters and return values. This metadata can only be
applied to loads of a pointer type.

The optional `!align` metadata must reference a single metadata name
`<align_node>` corresponding to a metadata node with one `i64` entry.
The existence of the `!align` metadata on the instruction tells the
optimizer that the value loaded is known to be aligned to a boundary
specified by the integer value in the metadata node. The alignment must
be a power of 2. This is analogous to the \'\'align\'\' attribute on
parameters and return values. This metadata can only be applied to loads
of a pointer type. If the returned value is not appropriately aligned at
runtime, the behavior is undefined.

##### Semantics:

The location of memory pointed to is loaded. If the value being loaded
is of scalar type then the number of bytes read does not exceed the
minimum number of bytes needed to hold all bits of the type. For
example, loading an `i24` reads at most three bytes. When loading a
value of a type like `i20` with a size that is not an integral number of
bytes, the result is undefined if the value was not originally written
using a store of the same type.

##### Examples:

``` {.llvm}
%ptr = alloca i32                               ; yields i32*:ptr
store i32 3, i32* %ptr                          ; yields void
%val = load i32, i32* %ptr                      ; yields i32:val = i32 3
```

#### \'`store`\' Instruction

##### Syntax:

    store [volatile] <ty> <value>, <ty>* <pointer>[, align <alignment>][, !nontemporal !<index>][, !invariant.group !<index>]        ; yields void
    store atomic [volatile] <ty> <value>, <ty>* <pointer> [syncscope("<target-scope>")] <ordering>, align <alignment> [, !invariant.group !<index>] ; yields void

##### Overview:

The \'`store`\' instruction is used to write to memory.

##### Arguments:

There are two arguments to the `store` instruction: a value to store and
an address at which to store it. The type of the `<pointer>` operand
must be a pointer to the `first class <t_firstclass>`{.interpreted-text
role="ref"} type of the `<value>` operand. If the `store` is marked as
`volatile`, then the optimizer is not allowed to modify the number or
order of execution of this `store` with other
`volatile operations <volatile>`{.interpreted-text role="ref"}. Only
values of `first class
<t_firstclass>`{.interpreted-text role="ref"} types of known size (i.e.
not containing an `opaque
structural type <t_opaque>`{.interpreted-text role="ref"}) can be
stored.

If the `store` is marked as `atomic`, it takes an extra `ordering
<ordering>`{.interpreted-text role="ref"} and optional
`syncscope("<target-scope>")` argument. The `acquire` and `acq_rel`
orderings aren\'t valid on `store` instructions. Atomic loads produce
`defined <memmodel>`{.interpreted-text role="ref"} results when they may
see multiple atomic stores. The type of the pointee must be an integer,
pointer, or floating-point type whose bit width is a power of two
greater than or equal to eight and less than or equal to a
target-specific size limit. `align` must be explicitly specified on
atomic stores, and the store has undefined behavior if the alignment is
not set to a value which is at least the size in bytes of the pointee.
`!nontemporal` does not have any defined semantics for atomic stores.

The optional constant `align` argument specifies the alignment of the
operation (that is, the alignment of the memory address). A value of 0
or an omitted `align` argument means that the operation has the ABI
alignment for the target. It is the responsibility of the code emitter
to ensure that the alignment information is correct. Overestimating the
alignment results in undefined behavior. Underestimating the alignment
may produce less efficient code. An alignment of 1 is always safe. The
maximum possible alignment is `1 << 29`. An alignment value higher than
the size of the stored type implies memory up to the alignment value
bytes can be stored to without trapping in the default address space.
Storing to the higher bytes however may result in data races if another
thread can access the same address. Introducing a data race is not
allowed. Storing to the extra bytes is not allowed even in situations
where a data race is known to not exist if the function has the
`sanitize_address` attribute.

The optional `!nontemporal` metadata must reference a single metadata
name `<index>` corresponding to a metadata node with one `i32` entry of
value 1. The existence of the `!nontemporal` metadata on the instruction
tells the optimizer and code generator that this load is not expected to
be reused in the cache. The code generator may select special
instructions to save cache bandwidth, such as the `MOVNT` instruction on
x86.

The optional `!invariant.group` metadata must reference a single
metadata name `<index>`. See `invariant.group` metadata.

##### Semantics:

The contents of memory are updated to contain `<value>` at the location
specified by the `<pointer>` operand. If `<value>` is of scalar type
then the number of bytes written does not exceed the minimum number of
bytes needed to hold all bits of the type. For example, storing an `i24`
writes at most three bytes. When writing a value of a type like `i20`
with a size that is not an integral number of bytes, it is unspecified
what happens to the extra bits that do not belong to the type, but they
will typically be overwritten.

##### Example:

``` {.llvm}
%ptr = alloca i32                               ; yields i32*:ptr
store i32 3, i32* %ptr                          ; yields void
%val = load i32, i32* %ptr                      ; yields i32:val = i32 3
```

#### \'`fence`\' Instruction

##### Syntax:

    fence [syncscope("<target-scope>")] <ordering>  ; yields void

##### Overview:

The \'`fence`\' instruction is used to introduce happens-before edges
between operations.

##### Arguments:

\'`fence`\' instructions take an `ordering <ordering>`{.interpreted-text
role="ref"} argument which defines what *synchronizes-with* edges they
add. They can only be given `acquire`, `release`, `acq_rel`, and
`seq_cst` orderings.

##### Semantics:

A fence A which has (at least) `release` ordering semantics
*synchronizes with* a fence B with (at least) `acquire` ordering
semantics if and only if there exist atomic operations X and Y, both
operating on some atomic object M, such that A is sequenced before X, X
modifies M (either directly or through some side effect of a sequence
headed by X), Y is sequenced before B, and Y observes M. This provides a
*happens-before* dependency between A and B. Rather than an explicit
`fence`, one (but not both) of the atomic operations X or Y might
provide a `release` or `acquire` (resp.) ordering constraint and still
*synchronize-with* the explicit `fence` and establish the
*happens-before* edge.

A `fence` which has `seq_cst` ordering, in addition to having both
`acquire` and `release` semantics specified above, participates in the
global program order of other `seq_cst` operations and/or fences.

A `fence` instruction can also take an optional
\"`syncscope <syncscope>`{.interpreted-text role="ref"}\" argument.

##### Example:

``` {.text}
fence acquire                                        ; yields void
fence syncscope("singlethread") seq_cst              ; yields void
fence syncscope("agent") seq_cst                     ; yields void
```

#### \'`cmpxchg`\' Instruction

##### Syntax:

    cmpxchg [weak] [volatile] <ty>* <pointer>, <ty> <cmp>, <ty> <new> [syncscope("<target-scope>")] <success ordering> <failure ordering> ; yields  { ty, i1 }

##### Overview:

The \'`cmpxchg`\' instruction is used to atomically modify memory. It
loads a value in memory and compares it to a given value. If they are
equal, it tries to store a new value into the memory.

##### Arguments:

There are three arguments to the \'`cmpxchg`\' instruction: an address
to operate on, a value to compare to the value currently be at that
address, and a new value to place at that address if the compared values
are equal. The type of \'\<cmp\>\' must be an integer or pointer type
whose bit width is a power of two greater than or equal to eight and
less than or equal to a target-specific size limit. \'\<cmp\>\' and
\'\<new\>\' must have the same type, and the type of \'\<pointer\>\'
must be a pointer to that type. If the `cmpxchg` is marked as
`volatile`, then the optimizer is not allowed to modify the number or
order of execution of this `cmpxchg` with other
`volatile operations <volatile>`{.interpreted-text role="ref"}.

The success and failure `ordering <ordering>`{.interpreted-text
role="ref"} arguments specify how this `cmpxchg` synchronizes with other
atomic operations. Both ordering parameters must be at least
`monotonic`, the ordering constraint on failure must be no stronger than
that on success, and the failure ordering cannot be either `release` or
`acq_rel`.

A `cmpxchg` instruction can also take an optional
\"`syncscope <syncscope>`{.interpreted-text role="ref"}\" argument.

The pointer passed into cmpxchg must have alignment greater than or
equal to the size in memory of the operand.

##### Semantics:

The contents of memory at the location specified by the \'`<pointer>`\'
operand is read and compared to \'`<cmp>`\'; if the values are equal,
\'`<new>`\' is written to the location. The original value at the
location is returned, together with a flag indicating success (true) or
failure (false).

If the cmpxchg operation is marked as `weak` then a spurious failure is
permitted: the operation may not write `<new>` even if the comparison
matched.

If the cmpxchg operation is strong (the default), the i1 value is 1 if
and only if the value loaded equals `cmp`.

A successful `cmpxchg` is a read-modify-write instruction for the
purpose of identifying release sequences. A failed `cmpxchg` is
equivalent to an atomic load with an ordering parameter determined the
second ordering parameter.

##### Example:

``` {.llvm}
entry:
  %orig = load atomic i32, i32* %ptr unordered, align 4                      ; yields i32
  br label %loop

loop:
  %cmp = phi i32 [ %orig, %entry ], [%value_loaded, %loop]
  %squared = mul i32 %cmp, %cmp
  %val_success = cmpxchg i32* %ptr, i32 %cmp, i32 %squared acq_rel monotonic ; yields  { i32, i1 }
  %value_loaded = extractvalue { i32, i1 } %val_success, 0
  %success = extractvalue { i32, i1 } %val_success, 1
  br i1 %success, label %done, label %loop

done:
  ...
```

#### \'`atomicrmw`\' Instruction

##### Syntax:

    atomicrmw [volatile] <operation> <ty>* <pointer>, <ty> <value> [syncscope("<target-scope>")] <ordering>                   ; yields ty

##### Overview:

The \'`atomicrmw`\' instruction is used to atomically modify memory.

##### Arguments:

There are three arguments to the \'`atomicrmw`\' instruction: an
operation to apply, an address whose value to modify, an argument to the
operation. The operation must be one of the following keywords:

-   xchg
-   add
-   sub
-   and
-   nand
-   or
-   xor
-   max
-   min
-   umax
-   umin

The type of \'\<value\>\' must be an integer type whose bit width is a
power of two greater than or equal to eight and less than or equal to a
target-specific size limit. The type of the \'`<pointer>`\' operand must
be a pointer to that type. If the `atomicrmw` is marked as `volatile`,
then the optimizer is not allowed to modify the number or order of
execution of this `atomicrmw` with other `volatile
operations <volatile>`{.interpreted-text role="ref"}.

A `atomicrmw` instruction can also take an optional
\"`syncscope <syncscope>`{.interpreted-text role="ref"}\" argument.

##### Semantics:

The contents of memory at the location specified by the \'`<pointer>`\'
operand are atomically read, modified, and written back. The original
value at the location is returned. The modification is specified by the
operation argument:

-   xchg: `*ptr = val`
-   add: `*ptr = *ptr + val`
-   sub: `*ptr = *ptr - val`
-   and: `*ptr = *ptr & val`
-   nand: `*ptr = ~(*ptr & val)`
-   or: `*ptr = *ptr | val`
-   xor: `*ptr = *ptr ^ val`
-   max: `*ptr = *ptr > val ? *ptr : val` (using a signed comparison)
-   min: `*ptr = *ptr < val ? *ptr : val` (using a signed comparison)
-   umax: `*ptr = *ptr > val ? *ptr : val` (using an unsigned
    comparison)
-   umin: `*ptr = *ptr < val ? *ptr : val` (using an unsigned
    comparison)

##### Example:

``` {.llvm}
%old = atomicrmw add i32* %ptr, i32 1 acquire                        ; yields i32
```

#### \'`getelementptr`\' Instruction

##### Syntax:

    <result> = getelementptr <ty>, <ty>* <ptrval>{, [inrange] <ty> <idx>}*
    <result> = getelementptr inbounds <ty>, <ty>* <ptrval>{, [inrange] <ty> <idx>}*
    <result> = getelementptr <ty>, <ptr vector> <ptrval>, [inrange] <vector index type> <idx>

##### Overview:

The \'`getelementptr`\' instruction is used to get the address of a
subelement of an `aggregate <t_aggregate>`{.interpreted-text role="ref"}
data structure. It performs address calculation only and does not access
memory. The instruction can also be used to calculate a vector of such
addresses.

##### Arguments:

The first argument is always a type used as the basis for the
calculations. The second argument is always a pointer or a vector of
pointers, and is the base address to start from. The remaining arguments
are indices that indicate which of the elements of the aggregate object
are indexed. The interpretation of each index is dependent on the type
being indexed into. The first index always indexes the pointer value
given as the second argument, the second index indexes a value of the
type pointed to (not necessarily the value directly pointed to, since
the first index can be non-zero), etc. The first type indexed into must
be a pointer value, subsequent types can be arrays, vectors, and
structs. Note that subsequent types being indexed into can never be
pointers, since that would require loading the pointer before continuing
calculation.

The type of each index argument depends on the type it is indexing into.
When indexing into a (optionally packed) structure, only `i32` integer
**constants** are allowed (when using a vector of indices they must all
be the **same** `i32` integer constant). When indexing into an array,
pointer or vector, integers of any width are allowed, and they are not
required to be constant. These integers are treated as signed values
where relevant.

For example, let\'s consider a C code fragment and how it gets compiled
to LLVM:

``` {.c}
struct RT {
  char A;
  int B[10][20];
  char C;
};
struct ST {
  int X;
  double Y;
  struct RT Z;
};

int *foo(struct ST *s) {
  return &s[1].Z.B[5][13];
}
```

The LLVM code generated by Clang is:

``` {.llvm}
%struct.RT = type { i8, [10 x [20 x i32]], i8 }
%struct.ST = type { i32, double, %struct.RT }

define i32* @foo(%struct.ST* %s) nounwind uwtable readnone optsize ssp {
entry:
  %arrayidx = getelementptr inbounds %struct.ST, %struct.ST* %s, i64 1, i32 2, i32 1, i64 5, i64 13
  ret i32* %arrayidx
}
```

##### Semantics:

In the example above, the first index is indexing into the
\'`%struct.ST*`\' type, which is a pointer, yielding a \'`%struct.ST`\'
= \'`{ i32, double, %struct.RT }`\' type, a structure. The second index
indexes into the third element of the structure, yielding a
\'`%struct.RT`\' = \'`{ i8 , [10 x [20 x i32]], i8 }`\' type, another
structure. The third index indexes into the second element of the
structure, yielding a \'`[10 x [20 x i32]]`\' type, an array. The two
dimensions of the array are subscripted into, yielding an \'`i32`\'
type. The \'`getelementptr`\' instruction returns a pointer to this
element, thus computing a value of \'`i32*`\' type.

Note that it is perfectly legal to index partially through a structure,
returning a pointer to an inner element. Because of this, the LLVM code
for the given testcase is equivalent to:

``` {.llvm}
define i32* @foo(%struct.ST* %s) {
  %t1 = getelementptr %struct.ST, %struct.ST* %s, i32 1                        ; yields %struct.ST*:%t1
  %t2 = getelementptr %struct.ST, %struct.ST* %t1, i32 0, i32 2                ; yields %struct.RT*:%t2
  %t3 = getelementptr %struct.RT, %struct.RT* %t2, i32 0, i32 1                ; yields [10 x [20 x i32]]*:%t3
  %t4 = getelementptr [10 x [20 x i32]], [10 x [20 x i32]]* %t3, i32 0, i32 5  ; yields [20 x i32]*:%t4
  %t5 = getelementptr [20 x i32], [20 x i32]* %t4, i32 0, i32 13               ; yields i32*:%t5
  ret i32* %t5
}
```

If the `inbounds` keyword is present, the result value of the
`getelementptr` is a `poison value <poisonvalues>`{.interpreted-text
role="ref"} if the base pointer is not an *in bounds* address of an
allocated object, or if any of the addresses that would be formed by
successive addition of the offsets implied by the indices to the base
address with infinitely precise signed arithmetic are not an *in bounds*
address of that allocated object. The *in bounds* addresses for an
allocated object are all the addresses that point into the object, plus
the address one byte past the end. The only *in bounds* address for a
null pointer in the default address-space is the null pointer itself. In
cases where the base is a vector of pointers the `inbounds` keyword
applies to each of the computations element-wise.

If the `inbounds` keyword is not present, the offsets are added to the
base address with silently-wrapping two\'s complement arithmetic. If the
offsets have a different width from the pointer, they are sign-extended
or truncated to the width of the pointer. The result value of the
`getelementptr` may be outside the object pointed to by the base
pointer. The result value may not necessarily be used to access memory
though, even if it happens to point into allocated storage. See the
`Pointer Aliasing Rules <pointeraliasing>`{.interpreted-text role="ref"}
section for more information.

If the `inrange` keyword is present before any index, loading from or
storing to any pointer derived from the `getelementptr` has undefined
behavior if the load or store would access memory outside of the bounds
of the element selected by the index marked as `inrange`. The result of
a pointer comparison or `ptrtoint` (including `ptrtoint`-like operations
involving memory) involving a pointer derived from a `getelementptr`
with the `inrange` keyword is undefined, with the exception of
comparisons in the case where both operands are in the range of the
element selected by the `inrange` keyword, inclusive of the address one
past the end of that element. Note that the `inrange` keyword is
currently only allowed in constant `getelementptr` expressions.

The getelementptr instruction is often confusing. For some more insight
into how it works, see
`the getelementptr FAQ <GetElementPtr>`{.interpreted-text role="doc"}.

##### Example:

``` {.llvm}
; yields [12 x i8]*:aptr
%aptr = getelementptr {i32, [12 x i8]}, {i32, [12 x i8]}* %saptr, i64 0, i32 1
; yields i8*:vptr
%vptr = getelementptr {i32, <2 x i8>}, {i32, <2 x i8>}* %svptr, i64 0, i32 1, i32 1
; yields i8*:eptr
%eptr = getelementptr [12 x i8], [12 x i8]* %aptr, i64 0, i32 1
; yields i32*:iptr
%iptr = getelementptr [10 x i32], [10 x i32]* @arr, i16 0, i16 0
```

##### Vector of pointers:

The `getelementptr` returns a vector of pointers, instead of a single
address, when one or more of its arguments is a vector. In such cases,
all vector arguments should have the same number of elements, and every
scalar argument will be effectively broadcast into a vector during
address calculation.

``` {.llvm}
; All arguments are vectors:
;   A[i] = ptrs[i] + offsets[i]*sizeof(i8)
%A = getelementptr i8, <4 x i8*> %ptrs, <4 x i64> %offsets

; Add the same scalar offset to each pointer of a vector:
;   A[i] = ptrs[i] + offset*sizeof(i8)
%A = getelementptr i8, <4 x i8*> %ptrs, i64 %offset

; Add distinct offsets to the same pointer:
;   A[i] = ptr + offsets[i]*sizeof(i8)
%A = getelementptr i8, i8* %ptr, <4 x i64> %offsets

; In all cases described above the type of the result is <4 x i8*>
```

The two following instructions are equivalent:

``` {.llvm}
getelementptr  %struct.ST, <4 x %struct.ST*> %s, <4 x i64> %ind1,
  <4 x i32> <i32 2, i32 2, i32 2, i32 2>,
  <4 x i32> <i32 1, i32 1, i32 1, i32 1>,
  <4 x i32> %ind4,
  <4 x i64> <i64 13, i64 13, i64 13, i64 13>

getelementptr  %struct.ST, <4 x %struct.ST*> %s, <4 x i64> %ind1,
  i32 2, i32 1, <4 x i32> %ind4, i64 13
```

Let\'s look at the C code, where the vector version of `getelementptr`
makes sense:

``` {.c}
// Let's assume that we vectorize the following loop:
double *A, *B; int *C;
for (int i = 0; i < size; ++i) {
  A[i] = B[C[i]];
}
```

``` {.llvm}
; get pointers for 8 elements from array B
%ptrs = getelementptr double, double* %B, <8 x i32> %C
; load 8 elements from array B into A
%A = call <8 x double> @llvm.masked.gather.v8f64.v8p0f64(<8 x double*> %ptrs,
     i32 8, <8 x i1> %mask, <8 x double> %passthru)
```

