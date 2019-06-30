### Conversion Operations

The instructions in this category are the conversion instructions
(casting) which all take a single operand and a type. They perform
various bit conversions on the operand.

#### \'`trunc .. to`\' Instruction {#i_trunc}

##### Syntax:

    <result> = trunc <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`trunc`\' instruction truncates its operand to the type `ty2`.

##### Arguments:

The \'`trunc`\' instruction takes a value to trunc, and a type to trunc
it to. Both types must be of `integer <t_integer>`{.interpreted-text
role="ref"} types, or vectors of the same number of integers. The bit
size of the `value` must be larger than the bit size of the destination
type, `ty2`. Equal sized types are not allowed.

##### Semantics:

The \'`trunc`\' instruction truncates the high order bits in `value` and
converts the remaining bits to `ty2`. Since the source size must be
larger than the destination size, `trunc` cannot be a *no-op cast*. It
will always truncate bits.

##### Example:

``` {.llvm}
%X = trunc i32 257 to i8                        ; yields i8:1
%Y = trunc i32 123 to i1                        ; yields i1:true
%Z = trunc i32 122 to i1                        ; yields i1:false
%W = trunc <2 x i16> <i16 8, i16 7> to <2 x i8> ; yields <i8 8, i8 7>
```

#### \'`zext .. to`\' Instruction {#i_zext}

##### Syntax:

    <result> = zext <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`zext`\' instruction zero extends its operand to type `ty2`.

##### Arguments:

The \'`zext`\' instruction takes a value to cast, and a type to cast it
to. Both types must be of `integer <t_integer>`{.interpreted-text
role="ref"} types, or vectors of the same number of integers. The bit
size of the `value` must be smaller than the bit size of the destination
type, `ty2`.

##### Semantics:

The `zext` fills the high order bits of the `value` with zero bits until
it reaches the size of the destination type, `ty2`.

When zero extending from i1, the result will always be either 0 or 1.

##### Example:

``` {.llvm}
%X = zext i32 257 to i64              ; yields i64:257
%Y = zext i1 true to i32              ; yields i32:1
%Z = zext <2 x i16> <i16 8, i16 7> to <2 x i32> ; yields <i32 8, i32 7>
```

#### \'`sext .. to`\' Instruction {#i_sext}

##### Syntax:

    <result> = sext <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`sext`\' sign extends `value` to the type `ty2`.

##### Arguments:

The \'`sext`\' instruction takes a value to cast, and a type to cast it
to. Both types must be of `integer <t_integer>`{.interpreted-text
role="ref"} types, or vectors of the same number of integers. The bit
size of the `value` must be smaller than the bit size of the destination
type, `ty2`.

##### Semantics:

The \'`sext`\' instruction performs a sign extension by copying the sign
bit (highest order bit) of the `value` until it reaches the bit size of
the type `ty2`.

When sign extending from i1, the extension always results in -1 or 0.

##### Example:

``` {.llvm}
%X = sext i8  -1 to i16              ; yields i16   :65535
%Y = sext i1 true to i32             ; yields i32:-1
%Z = sext <2 x i16> <i16 8, i16 7> to <2 x i32> ; yields <i32 8, i32 7>
```

#### \'`fptrunc .. to`\' Instruction

##### Syntax:

    <result> = fptrunc <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`fptrunc`\' instruction truncates `value` to type `ty2`.

##### Arguments:

The \'`fptrunc`\' instruction takes a
`floating-point <t_floating>`{.interpreted-text role="ref"} value to
cast and a `floating-point <t_floating>`{.interpreted-text role="ref"}
type to cast it to. The size of `value` must be larger than the size of
`ty2`. This implies that `fptrunc` cannot be used to make a *no-op
cast*.

##### Semantics:

The \'`fptrunc`\' instruction casts a `value` from a larger
`floating-point <t_floating>`{.interpreted-text role="ref"} type to a
smaller `floating-point
<t_floating>`{.interpreted-text role="ref"} type. This instruction is
assumed to execute in the default `floating-point
environment <floatenv>`{.interpreted-text role="ref"}.

##### Example:

``` {.llvm}
%X = fptrunc double 16777217.0 to float    ; yields float:16777216.0
%Y = fptrunc double 1.0E+300 to half       ; yields half:+infinity
```

#### \'`fpext .. to`\' Instruction

##### Syntax:

    <result> = fpext <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`fpext`\' extends a floating-point `value` to a larger
floating-point value.

##### Arguments:

The \'`fpext`\' instruction takes a
`floating-point <t_floating>`{.interpreted-text role="ref"} `value` to
cast, and a `floating-point <t_floating>`{.interpreted-text role="ref"}
type to cast it to. The source type must be smaller than the destination
type.

##### Semantics:

The \'`fpext`\' instruction extends the `value` from a smaller
`floating-point <t_floating>`{.interpreted-text role="ref"} type to a
larger `floating-point
<t_floating>`{.interpreted-text role="ref"} type. The `fpext` cannot be
used to make a *no-op cast* because it always changes bits. Use
`bitcast` to make a *no-op cast* for a floating-point cast.

##### Example:

``` {.llvm}
%X = fpext float 3.125 to double         ; yields double:3.125000e+00
%Y = fpext double %X to fp128            ; yields fp128:0xL00000000000000004000900000000000
```

#### \'`fptoui .. to`\' Instruction

##### Syntax:

    <result> = fptoui <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`fptoui`\' converts a floating-point `value` to its unsigned
integer equivalent of type `ty2`.

##### Arguments:

The \'`fptoui`\' instruction takes a value to cast, which must be a
scalar or vector `floating-point <t_floating>`{.interpreted-text
role="ref"} value, and a type to cast it to `ty2`, which must be an
`integer <t_integer>`{.interpreted-text role="ref"} type. If `ty` is a
vector floating-point type, `ty2` must be a vector integer type with the
same number of elements as `ty`

##### Semantics:

The \'`fptoui`\' instruction converts its `floating-point
<t_floating>`{.interpreted-text role="ref"} operand into the nearest
(rounding towards zero) unsigned integer value. If the value cannot fit
in `ty2`, the result is a
`poison value <poisonvalues>`{.interpreted-text role="ref"}.

##### Example:

``` {.llvm}
%X = fptoui double 123.0 to i32      ; yields i32:123
%Y = fptoui float 1.0E+300 to i1     ; yields undefined:1
%Z = fptoui float 1.04E+17 to i8     ; yields undefined:1
```

#### \'`fptosi .. to`\' Instruction

##### Syntax:

    <result> = fptosi <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`fptosi`\' instruction converts
`floating-point <t_floating>`{.interpreted-text role="ref"} `value` to
type `ty2`.

##### Arguments:

The \'`fptosi`\' instruction takes a value to cast, which must be a
scalar or vector `floating-point <t_floating>`{.interpreted-text
role="ref"} value, and a type to cast it to `ty2`, which must be an
`integer <t_integer>`{.interpreted-text role="ref"} type. If `ty` is a
vector floating-point type, `ty2` must be a vector integer type with the
same number of elements as `ty`

##### Semantics:

The \'`fptosi`\' instruction converts its `floating-point
<t_floating>`{.interpreted-text role="ref"} operand into the nearest
(rounding towards zero) signed integer value. If the value cannot fit in
`ty2`, the result is a `poison value <poisonvalues>`{.interpreted-text
role="ref"}.

##### Example:

``` {.llvm}
%X = fptosi double -123.0 to i32      ; yields i32:-123
%Y = fptosi float 1.0E-247 to i1      ; yields undefined:1
%Z = fptosi float 1.04E+17 to i8      ; yields undefined:1
```

#### \'`uitofp .. to`\' Instruction

##### Syntax:

    <result> = uitofp <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`uitofp`\' instruction regards `value` as an unsigned integer and
converts that value to the `ty2` type.

##### Arguments:

The \'`uitofp`\' instruction takes a value to cast, which must be a
scalar or vector `integer <t_integer>`{.interpreted-text role="ref"}
value, and a type to cast it to `ty2`, which must be an
`floating-point <t_floating>`{.interpreted-text role="ref"} type. If
`ty` is a vector integer type, `ty2` must be a vector floating-point
type with the same number of elements as `ty`

##### Semantics:

The \'`uitofp`\' instruction interprets its operand as an unsigned
integer quantity and converts it to the corresponding floating-point
value. If the value cannot be exactly represented, it is rounded using
the default rounding mode.

##### Example:

``` {.llvm}
%X = uitofp i32 257 to float         ; yields float:257.0
%Y = uitofp i8 -1 to double          ; yields double:255.0
```

#### \'`sitofp .. to`\' Instruction

##### Syntax:

    <result> = sitofp <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`sitofp`\' instruction regards `value` as a signed integer and
converts that value to the `ty2` type.

##### Arguments:

The \'`sitofp`\' instruction takes a value to cast, which must be a
scalar or vector `integer <t_integer>`{.interpreted-text role="ref"}
value, and a type to cast it to `ty2`, which must be an
`floating-point <t_floating>`{.interpreted-text role="ref"} type. If
`ty` is a vector integer type, `ty2` must be a vector floating-point
type with the same number of elements as `ty`

##### Semantics:

The \'`sitofp`\' instruction interprets its operand as a signed integer
quantity and converts it to the corresponding floating-point value. If
the value cannot be exactly represented, it is rounded using the default
rounding mode.

##### Example:

``` {.llvm}
%X = sitofp i32 257 to float         ; yields float:257.0
%Y = sitofp i8 -1 to double          ; yields double:-1.0
```

#### \'`ptrtoint .. to`\' Instruction {#i_ptrtoint}

##### Syntax:

    <result> = ptrtoint <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`ptrtoint`\' instruction converts the pointer or a vector of
pointers `value` to the integer (or vector of integers) type `ty2`.

##### Arguments:

The \'`ptrtoint`\' instruction takes a `value` to cast, which must be a
value of type `pointer <t_pointer>`{.interpreted-text role="ref"} or a
vector of pointers, and a type to cast it to `ty2`, which must be an
`integer <t_integer>`{.interpreted-text role="ref"} or a vector of
integers type.

##### Semantics:

The \'`ptrtoint`\' instruction converts `value` to integer type `ty2` by
interpreting the pointer value as an integer and either truncating or
zero extending that value to the size of the integer type. If `value` is
smaller than `ty2` then a zero extension is done. If `value` is larger
than `ty2` then a truncation is done. If they are the same size, then
nothing is done (*no-op cast*) other than a type change.

##### Example:

``` {.llvm}
%X = ptrtoint i32* %P to i8                         ; yields truncation on 32-bit architecture
%Y = ptrtoint i32* %P to i64                        ; yields zero extension on 32-bit architecture
%Z = ptrtoint <4 x i32*> %P to <4 x i64>; yields vector zero extension for a vector of addresses on 32-bit architecture
```

#### \'`inttoptr .. to`\' Instruction {#i_inttoptr}

##### Syntax:

    <result> = inttoptr <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`inttoptr`\' instruction converts an integer `value` to a pointer
type, `ty2`.

##### Arguments:

The \'`inttoptr`\' instruction takes an
`integer <t_integer>`{.interpreted-text role="ref"} value to cast, and a
type to cast it to, which must be a
`pointer <t_pointer>`{.interpreted-text role="ref"} type.

##### Semantics:

The \'`inttoptr`\' instruction converts `value` to type `ty2` by
applying either a zero extension or a truncation depending on the size
of the integer `value`. If `value` is larger than the size of a pointer
then a truncation is done. If `value` is smaller than the size of a
pointer then a zero extension is done. If they are the same size,
nothing is done (*no-op cast*).

##### Example:

``` {.llvm}
%X = inttoptr i32 255 to i32*          ; yields zero extension on 64-bit architecture
%Y = inttoptr i32 255 to i32*          ; yields no-op on 32-bit architecture
%Z = inttoptr i64 0 to i32*            ; yields truncation on 32-bit architecture
%Z = inttoptr <4 x i32> %G to <4 x i8*>; yields truncation of vector G to four pointers
```

#### \'`bitcast .. to`\' Instruction {#i_bitcast}

##### Syntax:

    <result> = bitcast <ty> <value> to <ty2>             ; yields ty2

##### Overview:

The \'`bitcast`\' instruction converts `value` to type `ty2` without
changing any bits.

##### Arguments:

The \'`bitcast`\' instruction takes a value to cast, which must be a
non-aggregate first class value, and a type to cast it to, which must
also be a non-aggregate `first class <t_firstclass>`{.interpreted-text
role="ref"} type. The bit sizes of `value` and the destination type,
`ty2`, must be identical. If the source type is a pointer, the
destination type must also be a pointer of the same size. This
instruction supports bitwise conversion of vectors to integers and to
vectors of other types (as long as they have the same size).

##### Semantics:

The \'`bitcast`\' instruction converts `value` to type `ty2`. It is
always a *no-op cast* because no bits change with this conversion. The
conversion is done as if the `value` had been stored to memory and read
back as type `ty2`. Pointer (or vector of pointers) types may only be
converted to other pointer (or vector of pointers) types with the same
address space through this instruction. To convert pointers to other
types, use the `inttoptr <i_inttoptr>`{.interpreted-text role="ref"} or
`ptrtoint <i_ptrtoint>`{.interpreted-text role="ref"} instructions
first.

##### Example:

``` {.text}
%X = bitcast i8 255 to i8              ; yields i8 :-1
%Y = bitcast i32* %x to sint*          ; yields sint*:%x
%Z = bitcast <2 x int> %V to i64;        ; yields i64: %V
%Z = bitcast <2 x i32*> %V to <2 x i64*> ; yields <2 x i64*>
```

#### \'`addrspacecast .. to`\' Instruction {#i_addrspacecast}

##### Syntax:

    <result> = addrspacecast <pty> <ptrval> to <pty2>       ; yields pty2

##### Overview:

The \'`addrspacecast`\' instruction converts `ptrval` from `pty` in
address space `n` to type `pty2` in address space `m`.

##### Arguments:

The \'`addrspacecast`\' instruction takes a pointer or vector of pointer
value to cast and a pointer type to cast it to, which must have a
different address space.

##### Semantics:

The \'`addrspacecast`\' instruction converts the pointer value `ptrval`
to type `pty2`. It can be a *no-op cast* or a complex value
modification, depending on the target and the address space pair.
Pointer conversions within the same address space must be performed with
the `bitcast` instruction. Note that if the address space conversion is
legal then both result and operand refer to the same memory location.

##### Example:

``` {.llvm}
%X = addrspacecast i32* %x to i32 addrspace(1)*    ; yields i32 addrspace(1)*:%x
%Y = addrspacecast i32 addrspace(1)* %y to i64 addrspace(2)*    ; yields i64 addrspace(2)*:%y
%Z = addrspacecast <4 x i32*> %z to <4 x float addrspace(3)*>   ; yields <4 x float addrspace(3)*>:%z
```

