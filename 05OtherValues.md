Other Values
------------

### Inline Assembler Expressions {#inlineasmexprs}

LLVM supports inline assembler expressions (as opposed to `Module-Level
Inline Assembly <moduleasm>`{.interpreted-text role="ref"}) through the
use of a special value. This value represents the inline assembler as a
template string (containing the instructions to emit), a list of operand
constraints (stored as a string), a flag that indicates whether or not
the inline asm expression has side effects, and a flag indicating
whether the function containing the asm needs to align its stack
conservatively.

The template string supports argument substitution of the operands using
\"`$`\" followed by a number, to indicate substitution of the given
register/memory location, as specified by the constraint string.
\"`${NUM:MODIFIER}`\" may also be used, where `MODIFIER` is a
target-specific annotation for how to print the operand (See
`inline-asm-modifiers`{.interpreted-text role="ref"}).

A literal \"`$`\" may be included by using \"`$$`\" in the template. To
include other special characters into the output, the usual \"`\XX`\"
escapes may be used, just as in other strings. Note that after template
substitution, the resulting assembly string is parsed by LLVM\'s
integrated assembler unless it is disabled \-- even when emitting a `.s`
file \-- and thus must contain assembly syntax known to LLVM.

LLVM also supports a few more substitions useful for writing inline
assembly:

-   `${:uid}`: Expands to a decimal integer unique to this inline
    assembly blob. This substitution is useful when declaring a local
    label. Many standard compiler optimizations, such as inlining, may
    duplicate an inline asm blob. Adding a blob-unique identifier
    ensures that the two labels will not conflict during assembly. This
    is used to implement [GCC\'s %= special format
    string](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html).
-   `${:comment}`: Expands to the comment character of the current
    target\'s assembly dialect. This is usually `#`, but many targets
    use other strings, such as `;`, `//`, or `!`.
-   `${:private}`: Expands to the assembler private label prefix. Labels
    with this prefix will not appear in the symbol table of the
    assembled object. Typically the prefix is `L`, but targets may use
    other strings. `.L` is relatively popular.

LLVM\'s support for inline asm is modeled closely on the requirements of
Clang\'s GCC-compatible inline-asm support. Thus, the feature-set and
the constraint and modifier codes listed here are similar or identical
to those in GCC\'s inline asm support. However, to be clear, the syntax
of the template and constraint strings described here is *not* the same
as the syntax accepted by GCC and Clang, and, while most constraint
letters are passed through as-is by Clang, some get translated to other
codes when converting from the C source to the LLVM assembly.

An example inline assembler expression is:

``` {.llvm}
i32 (i32) asm "bswap $0", "=r,r"
```

Inline assembler expressions may **only** be used as the callee operand
of a `call <i_call>`{.interpreted-text role="ref"} or an
`invoke <i_invoke>`{.interpreted-text role="ref"} instruction. Thus,
typically we have:

``` {.llvm}
%X = call i32 asm "bswap $0", "=r,r"(i32 %Y)
```

Inline asms with side effects not visible in the constraint list must be
marked as having side effects. This is done through the use of the
\'`sideeffect`\' keyword, like so:

``` {.llvm}
call void asm sideeffect "eieio", ""()
```

In some cases inline asms will contain code that will not work unless
the stack is aligned in some way, such as calls or SSE instructions on
x86, yet will not contain code that does that alignment within the asm.
The compiler should make conservative assumptions about what the asm
might contain and should generate its usual stack alignment code in the
prologue if the \'`alignstack`\' keyword is present:

``` {.llvm}
call void asm alignstack "eieio", ""()
```

Inline asms also support using non-standard assembly dialects. The
assumed dialect is ATT. When the \'`inteldialect`\' keyword is present,
the inline asm is using the Intel dialect. Currently, ATT and Intel are
the only supported dialects. An example is:

``` {.llvm}
call void asm inteldialect "eieio", ""()
```

If multiple keywords appear the \'`sideeffect`\' keyword must come
first, the \'`alignstack`\' keyword second and the \'`inteldialect`\'
keyword last.

#### Inline Asm Constraint String

The constraint list is a comma-separated string, each element containing
one or more constraint codes.

For each element in the constraint list an appropriate register or
memory operand will be chosen, and it will be made available to assembly
template string expansion as `$0` for the first constraint in the list,
`$1` for the second, etc.

There are three different types of constraints, which are distinguished
by a prefix symbol in front of the constraint code: Output, Input, and
Clobber. The constraints must always be given in that order: outputs
first, then inputs, then clobbers. They cannot be intermingled.

There are also three different categories of constraint codes:

-   Register constraint. This is either a register class, or a fixed
    physical register. This kind of constraint will allocate a register,
    and if necessary, bitcast the argument or result to the appropriate
    type.
-   Memory constraint. This kind of constraint is for use with an
    instruction taking a memory operand. Different constraints allow for
    different addressing modes used by the target.
-   Immediate value constraint. This kind of constraint is for an
    integer or other immediate value which can be rendered directly into
    an instruction. The various target-specific constraints allow the
    selection of a value in the proper range for the instruction you
    wish to use it with.

##### Output constraints

Output constraints are specified by an \"`=`\" prefix (e.g. \"`=r`\").
This indicates that the assembly will write to this operand, and the
operand will then be made available as a return value of the `asm`
expression. Output constraints do not consume an argument from the call
instruction. (Except, see below about indirect outputs).

Normally, it is expected that no output locations are written to by the
assembly expression until *all* of the inputs have been read. As such,
LLVM may assign the same register to an output and an input. If this is
not safe (e.g. if the assembly contains two instructions, where the
first writes to one output, and the second reads an input and writes to
a second output), then the \"`&`\" modifier must be used (e.g.
\"`=&r`\") to specify that the output is an \"early-clobber\" output.
Marking an output as \"early-clobber\" ensures that LLVM will not use
the same register for any inputs (other than an input tied to this
output).

##### Input constraints

Input constraints do not have a prefix \-- just the constraint codes.
Each input constraint will consume one argument from the call
instruction. It is not permitted for the asm to write to any input
register or memory location (unless that input is tied to an output).
Note also that multiple inputs may all be assigned to the same register,
if LLVM can determine that they necessarily all contain the same value.

Instead of providing a Constraint Code, input constraints may also
\"tie\" themselves to an output constraint, by providing an integer as
the constraint string. Tied inputs still consume an argument from the
call instruction, and take up a position in the asm template numbering
as is usual \-- they will simply be constrained to always use the same
register as the output they\'ve been tied to. For example, a constraint
string of \"`=r,0`\" says to assign a register for output, and use that
register as an input as well (it being the 0\'th constraint).

It is permitted to tie an input to an \"early-clobber\" output. In that
case, no *other* input may share the same register as the input tied to
the early-clobber (even when the other input has the same value).

You may only tie an input to an output which has a register constraint,
not a memory constraint. Only a single input may be tied to an output.

There is also an \"interesting\" feature which deserves a bit of
explanation: if a register class constraint allocates a register which
is too small for the value type operand provided as input, the input
value will be split into multiple registers, and all of them passed to
the inline asm.

However, this feature is often not as useful as you might think.

Firstly, the registers are *not* guaranteed to be consecutive. So, on
those architectures that have instructions which operate on multiple
consecutive instructions, this is not an appropriate way to support
them. (e.g. the 32-bit SparcV8 has a 64-bit load, which instruction
takes a single 32-bit register. The hardware then loads into both the
named register, and the next register. This feature of inline asm would
not be useful to support that.)

A few of the targets provide a template string modifier allowing
explicit access to the second register of a two-register operand (e.g.
MIPS `L`, `M`, and `D`). On such an architecture, you can actually
access the second allocated register (yet, still, not any subsequent
ones). But, in that case, you\'re still probably better off simply
splitting the value into two separate operands, for clarity. (e.g. see
the description of the `A` constraint on X86, which, despite existing
only for use with this feature, is not really a good idea to use)

##### Indirect inputs and outputs

Indirect output or input constraints can be specified by the \"`*`\"
modifier (which goes after the \"`=`\" in case of an output). This
indicates that the asm will write to or read from the contents of an
*address* provided as an input argument. (Note that in this way,
indirect outputs act more like an *input* than an output: just like an
input, they consume an argument of the call expression, rather than
producing a return value. An indirect output constraint is an \"output\"
only in that the asm is expected to write to the contents of the input
memory location, instead of just read from it).

This is most typically used for memory constraint, e.g. \"`=*m`\", to
pass the address of a variable as a value.

It is also possible to use an indirect *register* constraint, but only
on output (e.g. \"`=*r`\"). This will cause LLVM to allocate a register
for an output value normally, and then, separately emit a store to the
address provided as input, after the provided inline asm. (It\'s not
clear what value this functionality provides, compared to writing the
store explicitly after the asm statement, and it can only produce worse
code, since it bypasses many optimization passes. I would recommend not
using it.)

##### Clobber constraints

A clobber constraint is indicated by a \"`~`\" prefix. A clobber does
not consume an input operand, nor generate an output. Clobbers cannot
use any of the general constraint code letters \-- they may use only
explicit register constraints, e.g. \"`~{eax}`\". The one exception is
that a clobber string of \"`~{memory}`\" indicates that the assembly
writes to arbitrary undeclared memory locations \-- not only the memory
pointed to by a declared indirect output.

Note that clobbering named registers that are also present in output
constraints is not legal.

##### Constraint Codes

After a potential prefix comes constraint code, or codes.

A Constraint Code is either a single letter (e.g. \"`r`\"), a \"`^`\"
character followed by two letters (e.g. \"`^wc`\"), or \"`{`\"
register-name \"`}`\" (e.g. \"`{eax}`\").

The one and two letter constraint codes are typically chosen to be the
same as GCC\'s constraint codes.

A single constraint may include one or more than constraint code in it,
leaving it up to LLVM to choose which one to use. This is included
mainly for compatibility with the translation of GCC inline asm coming
from clang.

There are two ways to specify alternatives, and either or both may be
used in an inline asm constraint list:

1)  Append the codes to each other, making a constraint code set. E.g.
    \"`im`\" or \"`{eax}m`\". This means \"choose any of the options in
    the set\". The choice of constraint is made independently for each
    constraint in the constraint list.
2)  Use \"`|`\" between constraint code sets, creating alternatives.
    Every constraint in the constraint list must have the same number of
    alternative sets. With this syntax, the same alternative in *all* of
    the items in the constraint list will be chosen together.

Putting those together, you might have a two operand constraint string
like `"rm|r,ri|rm"`. This indicates that if operand 0 is `r` or `m`,
then operand 1 may be one of `r` or `i`. If operand 0 is `r`, then
operand 1 may be one of `r` or `m`. But, operand 0 and 1 cannot both be
of type m.

However, the use of either of the alternatives features is *NOT*
recommended, as LLVM is not able to make an intelligent choice about
which one to use. (At the point it currently needs to choose, not enough
information is available to do so in a smart way.) Thus, it simply tries
to make a choice that\'s most likely to compile, not one that will be
optimal performance. (e.g., given \"`rm`\", it\'ll always choose to use
memory, not registers). And, if given multiple registers, or multiple
register classes, it will simply choose the first one. (In fact, it
doesn\'t currently even ensure explicitly specified physical registers
are unique, so specifying multiple physical registers as alternatives,
like `{r11}{r12},{r11}{r12}`, will assign r11 to both operands, not at
all what was intended.)

##### Supported Constraint Code List

The constraint codes are, in general, expected to behave the same way
they do in GCC. LLVM\'s support is often implemented on an \'as-needed\'
basis, to support C inline asm code which was supported by GCC. A
mismatch in behavior between LLVM and GCC likely indicates a bug in
LLVM.

Some constraint codes are typically supported by all targets:

-   `r`: A register in the target\'s general purpose register class.
-   `m`: A memory address operand. It is target-specific what addressing
    modes are supported, typical examples are register, or register +
    register offset, or register + immediate offset (of some
    target-specific size).
-   `i`: An integer constant (of target-specific width). Allows either a
    simple immediate, or a relocatable value.
-   `n`: An integer constant \-- *not* including relocatable values.
-   `s`: An integer constant, but allowing *only* relocatable values.
-   `X`: Allows an operand of any kind, no constraint whatsoever.
    Typically useful to pass a label for an asm branch or call.
-   `{register-name}`: Requires exactly the named physical register.

Other constraints are target-specific:

AArch64:

-   `z`: An immediate integer 0. Outputs `WZR` or `XZR`, as appropriate.
-   `I`: An immediate integer valid for an `ADD` or `SUB` instruction,
    i.e. 0 to 4095 with optional shift by 12.
-   `J`: An immediate integer that, when negated, is valid for an `ADD`
    or `SUB` instruction, i.e. -1 to -4095 with optional left shift
    by 12.
-   `K`: An immediate integer that is valid for the \'bitmask immediate
    32\' of a logical instruction like `AND`, `EOR`, or `ORR` with a
    32-bit register.
-   `L`: An immediate integer that is valid for the \'bitmask immediate
    64\' of a logical instruction like `AND`, `EOR`, or `ORR` with a
    64-bit register.
-   `M`: An immediate integer for use with the `MOV` assembly alias on a
    32-bit register. This is a superset of `K`: in addition to the
    bitmask immediate, also allows immediate integers which can be
    loaded with a single `MOVZ` or `MOVL` instruction.
-   `N`: An immediate integer for use with the `MOV` assembly alias on a
    64-bit register. This is a superset of `L`.
-   `Q`: Memory address operand must be in a single register (no
    offsets). (However, LLVM currently does this for the `m` constraint
    as well.)
-   `r`: A 32 or 64-bit integer register (W\* or X\*).
-   `w`: A 32, 64, or 128-bit floating-point/SIMD register.
-   `x`: A lower 128-bit floating-point/SIMD register (`V0` to `V15`).

AMDGPU:

-   `r`: A 32 or 64-bit integer register.
-   `[0-9]v`: The 32-bit VGPR register, number 0-9.
-   `[0-9]s`: The 32-bit SGPR register, number 0-9.

All ARM modes:

-   `Q`, `Um`, `Un`, `Uq`, `Us`, `Ut`, `Uv`, `Uy`: Memory address
    operand. Treated the same as operand `m`, at the moment.

ARM and ARM\'s Thumb2 mode:

-   `j`: An immediate integer between 0 and 65535 (valid for `MOVW`)
-   `I`: An immediate integer valid for a data-processing instruction.
-   `J`: An immediate integer between -4095 and 4095.
-   `K`: An immediate integer whose bitwise inverse is valid for a
    data-processing instruction. (Can be used with template modifier
    \"`B`\" to print the inverted value).
-   `L`: An immediate integer whose negation is valid for a
    data-processing instruction. (Can be used with template modifier
    \"`n`\" to print the negated value).
-   `M`: A power of two or a integer between 0 and 32.
-   `N`: Invalid immediate constraint.
-   `O`: Invalid immediate constraint.
-   `r`: A general-purpose 32-bit integer register (`r0-r15`).
-   `l`: In Thumb2 mode, low 32-bit GPR registers (`r0-r7`). In ARM
    mode, same as `r`.
-   `h`: In Thumb2 mode, a high 32-bit GPR register (`r8-r15`). In ARM
    mode, invalid.
-   `w`: A 32, 64, or 128-bit floating-point/SIMD register: `s0-s31`,
    `d0-d31`, or `q0-q15`.
-   `x`: A 32, 64, or 128-bit floating-point/SIMD register: `s0-s15`,
    `d0-d7`, or `q0-q3`.
-   `t`: A low floating-point/SIMD register: `s0-s31`, `d0-d16`, or
    `q0-q8`.

ARM\'s Thumb1 mode:

-   `I`: An immediate integer between 0 and 255.
-   `J`: An immediate integer between -255 and -1.
-   `K`: An immediate integer between 0 and 255, with optional
    left-shift by some amount.
-   `L`: An immediate integer between -7 and 7.
-   `M`: An immediate integer which is a multiple of 4 between 0
    and 1020.
-   `N`: An immediate integer between 0 and 31.
-   `O`: An immediate integer which is a multiple of 4 between -508
    and 508.
-   `r`: A low 32-bit GPR register (`r0-r7`).
-   `l`: A low 32-bit GPR register (`r0-r7`).
-   `h`: A high GPR register (`r0-r7`).
-   `w`: A 32, 64, or 128-bit floating-point/SIMD register: `s0-s31`,
    `d0-d31`, or `q0-q15`.
-   `x`: A 32, 64, or 128-bit floating-point/SIMD register: `s0-s15`,
    `d0-d7`, or `q0-q3`.
-   `t`: A low floating-point/SIMD register: `s0-s31`, `d0-d16`, or
    `q0-q8`.

Hexagon:

-   `o`, `v`: A memory address operand, treated the same as constraint
    `m`, at the moment.
-   `r`: A 32 or 64-bit register.

MSP430:

-   `r`: An 8 or 16-bit register.

MIPS:

-   `I`: An immediate signed 16-bit integer.
-   `J`: An immediate integer zero.
-   `K`: An immediate unsigned 16-bit integer.
-   `L`: An immediate 32-bit integer, where the lower 16 bits are 0.
-   `N`: An immediate integer between -65535 and -1.
-   `O`: An immediate signed 15-bit integer.
-   `P`: An immediate integer between 1 and 65535.
-   `m`: A memory address operand. In MIPS-SE mode, allows a base
    address register plus 16-bit immediate offset. In MIPS mode, just a
    base register.
-   `R`: A memory address operand. In MIPS-SE mode, allows a base
    address register plus a 9-bit signed offset. In MIPS mode, the same
    as constraint `m`.
-   `ZC`: A memory address operand, suitable for use in a `pref`, `ll`,
    or `sc` instruction on the given subtarget (details vary).
-   `r`, `d`, `y`: A 32 or 64-bit GPR register.
-   `f`: A 32 or 64-bit FPU register (`F0-F31`), or a 128-bit MSA
    register (`W0-W31`). In the case of MSA registers, it is recommended
    to use the `w` argument modifier for compatibility with GCC.
-   `c`: A 32-bit or 64-bit GPR register suitable for indirect jump
    (always `25`).
-   `l`: The `lo` register, 32 or 64-bit.
-   `x`: Invalid.

NVPTX:

-   `b`: A 1-bit integer register.
-   `c` or `h`: A 16-bit integer register.
-   `r`: A 32-bit integer register.
-   `l` or `N`: A 64-bit integer register.
-   `f`: A 32-bit float register.
-   `d`: A 64-bit float register.

PowerPC:

-   `I`: An immediate signed 16-bit integer.
-   `J`: An immediate unsigned 16-bit integer, shifted left 16 bits.
-   `K`: An immediate unsigned 16-bit integer.
-   `L`: An immediate signed 16-bit integer, shifted left 16 bits.
-   `M`: An immediate integer greater than 31.
-   `N`: An immediate integer that is an exact power of 2.
-   `O`: The immediate integer constant 0.
-   `P`: An immediate integer constant whose negation is a signed 16-bit
    constant.
-   `es`, `o`, `Q`, `Z`, `Zy`: A memory address operand, currently
    treated the same as `m`.
-   `r`: A 32 or 64-bit integer register.
-   `b`: A 32 or 64-bit integer register, excluding `R0` (that is:
    `R1-R31`).
-   `f`: A 32 or 64-bit float register (`F0-F31`), or when QPX is
    enabled, a 128 or 256-bit QPX register (`Q0-Q31`; aliases the `F`
    registers).
-   `v`: For `4 x f32` or `4 x f64` types, when QPX is enabled, a 128 or
    256-bit QPX register (`Q0-Q31`), otherwise a 128-bit altivec vector
    register (`V0-V31`).
-   `y`: Condition register (`CR0-CR7`).
-   `wc`: An individual CR bit in a CR register.
-   `wa`, `wd`, `wf`: Any 128-bit VSX vector register, from the full VSX
    register set (overlapping both the floating-point and vector
    register files).
-   `ws`: A 32 or 64-bit floating-point register, from the full VSX
    register set.

Sparc:

-   `I`: An immediate 13-bit signed integer.
-   `r`: A 32-bit integer register.
-   `f`: Any floating-point register on SparcV8, or a floating-point
    register in the \"low\" half of the registers on SparcV9.
-   `e`: Any floating-point register. (Same as `f` on SparcV8.)

SystemZ:

-   `I`: An immediate unsigned 8-bit integer.
-   `J`: An immediate unsigned 12-bit integer.
-   `K`: An immediate signed 16-bit integer.
-   `L`: An immediate signed 20-bit integer.
-   `M`: An immediate integer 0x7fffffff.
-   `Q`: A memory address operand with a base address and a 12-bit
    immediate unsigned displacement.
-   `R`: A memory address operand with a base address, a 12-bit
    immediate unsigned displacement, and an index register.
-   `S`: A memory address operand with a base address and a 20-bit
    immediate signed displacement.
-   `T`: A memory address operand with a base address, a 20-bit
    immediate signed displacement, and an index register.
-   `r` or `d`: A 32, 64, or 128-bit integer register.
-   `a`: A 32, 64, or 128-bit integer address register (excludes R0,
    which in an address context evaluates as zero).
-   `h`: A 32-bit value in the high part of a 64bit data register
    (LLVM-specific)
-   `f`: A 32, 64, or 128-bit floating-point register.

X86:

-   `I`: An immediate integer between 0 and 31.
-   `J`: An immediate integer between 0 and 64.
-   `K`: An immediate signed 8-bit integer.
-   `L`: An immediate integer, 0xff or 0xffff or (in 64-bit mode only)
    0xffffffff.
-   `M`: An immediate integer between 0 and 3.
-   `N`: An immediate unsigned 8-bit integer.
-   `O`: An immediate integer between 0 and 127.
-   `e`: An immediate 32-bit signed integer.
-   `Z`: An immediate 32-bit unsigned integer.
-   `o`, `v`: Treated the same as `m`, at the moment.
-   `q`: An 8, 16, 32, or 64-bit register which can be accessed as an
    8-bit `l` integer register. On X86-32, this is the `a`, `b`, `c`,
    and `d` registers, and on X86-64, it is all of the integer
    registers.
-   `Q`: An 8, 16, 32, or 64-bit register which can be accessed as an
    8-bit `h` integer register. This is the `a`, `b`, `c`, and `d`
    registers.
-   `r` or `l`: An 8, 16, 32, or 64-bit integer register.
-   `R`: An 8, 16, 32, or 64-bit \"legacy\" integer register \-- one
    which has existed since i386, and can be accessed without the REX
    prefix.
-   `f`: A 32, 64, or 80-bit \'387 FPU stack pseudo-register.
-   `y`: A 64-bit MMX register, if MMX is enabled.
-   `x`: If SSE is enabled: a 32 or 64-bit scalar operand, or 128-bit
    vector operand in a SSE register. If AVX is also enabled, can also
    be a 256-bit vector operand in an AVX register. If AVX-512 is also
    enabled, can also be a 512-bit vector operand in an AVX512 register,
    Otherwise, an error.
-   `Y`: The same as `x`, if *SSE2* is enabled, otherwise an error.
-   `A`: Special case: allocates EAX first, then EDX, for a single
    operand (in 32-bit mode, a 64-bit integer operand will get split
    into two registers). It is not recommended to use this constraint,
    as in 64-bit mode, the 64-bit operand will get allocated only to RAX
    \-- if two 32-bit operands are needed, you\'re better off splitting
    it yourself, before passing it to the asm statement.

XCore:

-   `r`: A 32-bit integer register.

#### Asm template argument modifiers {#inline-asm-modifiers}

In the asm template string, modifiers can be used on the operand
reference, like \"`${0:n}`\".

The modifiers are, in general, expected to behave the same way they do
in GCC. LLVM\'s support is often implemented on an \'as-needed\' basis,
to support C inline asm code which was supported by GCC. A mismatch in
behavior between LLVM and GCC likely indicates a bug in LLVM.

Target-independent:

-   `c`: Print an immediate integer constant unadorned, without the
    target-specific immediate punctuation (e.g. no `$` prefix).
-   `n`: Negate and print immediate integer constant unadorned, without
    the target-specific immediate punctuation (e.g. no `$` prefix).
-   `l`: Print as an unadorned label, without the target-specific label
    punctuation (e.g. no `$` prefix).

AArch64:

-   `w`: Print a GPR register with a `w*` name instead of `x*` name.
    E.g., instead of `x30`, print `w30`.
-   `x`: Print a GPR register with a `x*` name. (this is the default,
    anyhow).
-   `b`, `h`, `s`, `d`, `q`: Print a floating-point/SIMD register with a
    `b*`, `h*`, `s*`, `d*`, or `q*` name, rather than the default of
    `v*`.

AMDGPU:

-   `r`: No effect.

ARM:

-   `a`: Print an operand as an address (with `[` and `]` surrounding a
    register).
-   `P`: No effect.
-   `q`: No effect.
-   `y`: Print a VFP single-precision register as an indexed double
    (e.g. print as `d4[1]` instead of `s9`)
-   `B`: Bitwise invert and print an immediate integer constant without
    `#` prefix.
-   `L`: Print the low 16-bits of an immediate integer constant.
-   `M`: Print as a register set suitable for ldm/stm. Also prints *all*
    register operands subsequent to the specified one (!), so use
    carefully.
-   `Q`: Print the low-order register of a register-pair, or the
    low-order register of a two-register operand.
-   `R`: Print the high-order register of a register-pair, or the
    high-order register of a two-register operand.
-   `H`: Print the second register of a register-pair. (On a big-endian
    system, `H` is equivalent to `Q`, and on little-endian system, `H`
    is equivalent to `R`.)
-   `e`: Print the low doubleword register of a NEON quad register.
-   `f`: Print the high doubleword register of a NEON quad register.
-   `m`: Print the base register of a memory operand without the `[` and
    `]` adornment.

Hexagon:

-   `L`: Print the second register of a two-register operand. Requires
    that it has been allocated consecutively to the first.
-   `I`: Print the letter \'i\' if the operand is an integer constant,
    otherwise nothing. Used to print \'addi\' vs \'add\' instructions.

MSP430:

No additional modifiers.

MIPS:

-   `X`: Print an immediate integer as hexadecimal
-   `x`: Print the low 16 bits of an immediate integer as hexadecimal.
-   `d`: Print an immediate integer as decimal.
-   `m`: Subtract one and print an immediate integer as decimal.
-   `z`: Print \$0 if an immediate zero, otherwise print normally.
-   `L`: Print the low-order register of a two-register operand, or
    prints the address of the low-order word of a double-word memory
    operand.
-   `M`: Print the high-order register of a two-register operand, or
    prints the address of the high-order word of a double-word memory
    operand.
-   `D`: Print the second register of a two-register operand, or prints
    the second word of a double-word memory operand. (On a big-endian
    system, `D` is equivalent to `L`, and on little-endian system, `D`
    is equivalent to `M`.)
-   `w`: No effect. Provided for compatibility with GCC which requires
    this modifier in order to print MSA registers (`W0-W31`) with the
    `f` constraint.

NVPTX:

-   `r`: No effect.

PowerPC:

-   `L`: Print the second register of a two-register operand. Requires
    that it has been allocated consecutively to the first.
-   `I`: Print the letter \'i\' if the operand is an integer constant,
    otherwise nothing. Used to print \'addi\' vs \'add\' instructions.
-   `y`: For a memory operand, prints formatter for a two-register
    X-form instruction. (Currently always prints `r0,OPERAND`).
-   `U`: Prints \'u\' if the memory operand is an update form, and
    nothing otherwise. (NOTE: LLVM does not support update form, so this
    will currently always print nothing)
-   `X`: Prints \'x\' if the memory operand is an indexed form. (NOTE:
    LLVM does not support indexed form, so this will currently always
    print nothing)

Sparc:

-   `r`: No effect.

SystemZ:

SystemZ implements only `n`, and does *not* support any of the other
target-independent modifiers.

X86:

-   `c`: Print an unadorned integer or symbol name. (The latter is
    target-specific behavior for this typically target-independent
    modifier).
-   `A`: Print a register name with a \'`*`\' before it.
-   `b`: Print an 8-bit register name (e.g. `al`); do nothing on a
    memory operand.
-   `h`: Print the upper 8-bit register name (e.g. `ah`); do nothing on
    a memory operand.
-   `w`: Print the 16-bit register name (e.g. `ax`); do nothing on a
    memory operand.
-   `k`: Print the 32-bit register name (e.g. `eax`); do nothing on a
    memory operand.
-   `q`: Print the 64-bit register name (e.g. `rax`), if 64-bit
    registers are available, otherwise the 32-bit register name; do
    nothing on a memory operand.
-   `n`: Negate and print an unadorned integer, or, for operands other
    than an immediate integer (e.g. a relocatable symbol expression),
    print a \'-\' before the operand. (The behavior for relocatable
    symbol expressions is a target-specific behavior for this typically
    target-independent modifier)
-   `H`: Print a memory reference with additional offset +8.
-   `P`: Print a memory reference or operand for use as the argument of
    a call instruction. (E.g. omit `(rip)`, even though it\'s
    PC-relative.)

XCore:

No additional modifiers.

#### Inline Asm Metadata

The call instructions that wrap inline asm nodes may have a
\"`!srcloc`\" MDNode attached to it that contains a list of constant
integers. If present, the code generator will use the integer as the
location cookie value when report errors through the `LLVMContext` error
reporting mechanisms. This allows a front-end to correlate backend
errors that occur with inline asm back to the source code that produced
it. For example:

``` {.llvm}
call void asm sideeffect "something bad", ""(), !srcloc !42
...
!42 = !{ i32 1234567 }
```

It is up to the front-end to make sense of the magic numbers it places
in the IR. If the MDNode contains multiple constants, the code generator
will use the one that corresponds to the line of the asm that the error
occurs on.

