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

Metadata
--------

LLVM IR allows metadata to be attached to instructions in the program
that can convey extra information about the code to the optimizers and
code generator. One example application of metadata is source-level
debug information. There are two metadata primitives: strings and nodes.

Metadata does not have a type, and is not a value. If referenced from a
`call` instruction, it uses the `metadata` type.

All metadata are identified in syntax by a exclamation point (\'`!`\').

### Metadata Nodes and Metadata Strings {#metadata-string}

A metadata string is a string surrounded by double quotes. It can
contain any character by escaping non-printable characters with
\"`\xx`\" where \"`xx`\" is the two digit hex code. For example:
\"`!"test\00"`\".

Metadata nodes are represented with notation similar to structure
constants (a comma separated list of elements, surrounded by braces and
preceded by an exclamation point). Metadata nodes can have any values as
their operand. For example:

``` {.llvm}
!{ !"test\00", i32 10}
```

Metadata nodes that aren\'t uniqued use the `distinct` keyword. For
example:

``` {.text}
!0 = distinct !{!"test\00", i32 10}
```

`distinct` nodes are useful when nodes shouldn\'t be merged based on
their content. They can also occur when transformations cause uniquing
collisions when metadata operands change.

A `named metadata <namedmetadatastructure>`{.interpreted-text
role="ref"} is a collection of metadata nodes, which can be looked up in
the module symbol table. For example:

``` {.llvm}
!foo = !{!4, !3}
```

Metadata can be used as function arguments. Here the `llvm.dbg.value`
intrinsic is using three metadata arguments:

``` {.llvm}
call void @llvm.dbg.value(metadata !24, metadata !25, metadata !26)
```

Metadata can be attached to an instruction. Here metadata `!21` is
attached to the `add` instruction using the `!dbg` identifier:

``` {.llvm}
%indvar.next = add i64 %indvar, 1, !dbg !21
```

Metadata can also be attached to a function or a global variable. Here
metadata `!22` is attached to the `f1` and `f2` functions, and the
globals `g1` and `g2` using the `!dbg` identifier:

``` {.llvm}
declare !dbg !22 void @f1()
define void @f2() !dbg !22 {
  ret void
}

@g1 = global i32 0, !dbg !22
@g2 = external global i32, !dbg !22
```

A transformation is required to drop any metadata attachment that it
does not know or know it can\'t preserve. Currently there is an
exception for metadata attachment to globals for `!type` and
`!absolute_symbol` which can\'t be unconditionally dropped unless the
global is itself deleted.

Metadata attached to a module using named metadata may not be dropped,
with the exception of debug metadata (named metadata with the name
`!llvm.dbg.*`).

More information about specific metadata nodes recognized by the
optimizers and code generator is found below.

#### Specialized Metadata Nodes {#specialized-metadata}

Specialized metadata nodes are custom data structures in metadata (as
opposed to generic tuples). Their fields are labelled, and can be
specified in any order.

These aren\'t inherently debug info centric, but currently all the
specialized metadata nodes are related to debug info.

##### DICompileUnit {#DICompileUnit}

`DICompileUnit` nodes represent a compile unit. The `enums:`,
`retainedTypes:`, `globals:`, `imports:` and `macros:` fields are tuples
containing the debug info to be emitted along with the compile unit,
regardless of code optimizations (some nodes are only emitted if there
are references to them from instructions). The `debugInfoForProfiling:`
field is a boolean indicating whether or not line-table discriminators
are updated to provide more-accurate debug info for profiling results.

``` {.text}
!0 = !DICompileUnit(language: DW_LANG_C99, file: !1, producer: "clang",
                    isOptimized: true, flags: "-O2", runtimeVersion: 2,
                    splitDebugFilename: "abc.debug", emissionKind: FullDebug,
                    enums: !2, retainedTypes: !3, globals: !4, imports: !5,
                    macros: !6, dwoId: 0x0abcd)
```

Compile unit descriptors provide the root scope for objects declared in
a specific compilation unit. File descriptors are defined using this
scope. These descriptors are collected by a named metadata node
`!llvm.dbg.cu`. They keep track of global variables, type information,
and imported entities (declarations and namespaces).

##### DIFile {#DIFile}

`DIFile` nodes represent files. The `filename:` can include slashes.

``` {.none}
!0 = !DIFile(filename: "path/to/file", directory: "/path/to/dir",
             checksumkind: CSK_MD5,
             checksum: "000102030405060708090a0b0c0d0e0f")
```

Files are sometimes used in `scope:` fields, and are the only valid
target for `file:` fields. Valid values for `checksumkind:` field are:
{CSK\_None, CSK\_MD5, CSK\_SHA1}

##### DIBasicType {#DIBasicType}

`DIBasicType` nodes represent primitive types, such as `int`, `bool` and
`float`. `tag:` defaults to `DW_TAG_base_type`.

``` {.text}
!0 = !DIBasicType(name: "unsigned char", size: 8, align: 8,
                  encoding: DW_ATE_unsigned_char)
!1 = !DIBasicType(tag: DW_TAG_unspecified_type, name: "decltype(nullptr)")
```

The `encoding:` describes the details of the type. Usually it\'s one of
the following:

``` {.text}
DW_ATE_address       = 1
DW_ATE_boolean       = 2
DW_ATE_float         = 4
DW_ATE_signed        = 5
DW_ATE_signed_char   = 6
DW_ATE_unsigned      = 7
DW_ATE_unsigned_char = 8
```

##### DISubroutineType {#DISubroutineType}

`DISubroutineType` nodes represent subroutine types. Their `types:`
field refers to a tuple; the first operand is the return type, while the
rest are the types of the formal arguments in order. If the first
operand is `null`, that represents a function with no return value (such
as `void foo() {}` in C++).

``` {.text}
!0 = !BasicType(name: "int", size: 32, align: 32, DW_ATE_signed)
!1 = !BasicType(name: "char", size: 8, align: 8, DW_ATE_signed_char)
!2 = !DISubroutineType(types: !{null, !0, !1}) ; void (int, char)
```

##### DIDerivedType {#DIDerivedType}

`DIDerivedType` nodes represent types derived from other types, such as
qualified types.

``` {.text}
!0 = !DIBasicType(name: "unsigned char", size: 8, align: 8,
                  encoding: DW_ATE_unsigned_char)
!1 = !DIDerivedType(tag: DW_TAG_pointer_type, baseType: !0, size: 32,
                    align: 32)
```

The following `tag:` values are valid:

``` {.text}
DW_TAG_member             = 13
DW_TAG_pointer_type       = 15
DW_TAG_reference_type     = 16
DW_TAG_typedef            = 22
DW_TAG_inheritance        = 28
DW_TAG_ptr_to_member_type = 31
DW_TAG_const_type         = 38
DW_TAG_friend             = 42
DW_TAG_volatile_type      = 53
DW_TAG_restrict_type      = 55
DW_TAG_atomic_type        = 71
```

::: {#DIDerivedTypeMember}
`DW_TAG_member` is used to define a member of a `composite type
<DICompositeType>`{.interpreted-text role="ref"}. The type of the member
is the `baseType:`. The `offset:` is the member\'s bit offset. If the
composite type has an ODR `identifier:` and does not set
`flags: DIFwdDecl`, then the member is uniqued based only on its `name:`
and `scope:`.
:::

`DW_TAG_inheritance` and `DW_TAG_friend` are used in the `elements:`
field of `composite types <DICompositeType>`{.interpreted-text
role="ref"} to describe parents and friends.

`DW_TAG_typedef` is used to provide a name for the `baseType:`.

`DW_TAG_pointer_type`, `DW_TAG_reference_type`, `DW_TAG_const_type`,
`DW_TAG_volatile_type`, `DW_TAG_restrict_type` and `DW_TAG_atomic_type`
are used to qualify the `baseType:`.

Note that the `void *` type is expressed as a type derived from NULL.

##### DICompositeType {#DICompositeType}

`DICompositeType` nodes represent types composed of other types, like
structures and unions. `elements:` points to a tuple of the composed
types.

If the source language supports ODR, the `identifier:` field gives the
unique identifier used for type merging between modules. When specified,
`subprogram declarations <DISubprogramDeclaration>`{.interpreted-text
role="ref"} and `member
derived types <DIDerivedTypeMember>`{.interpreted-text role="ref"} that
reference the ODR-type in their `scope:` change uniquing rules.

For a given `identifier:`, there should only be a single composite type
that does not have `flags: DIFlagFwdDecl` set. LLVM tools that link
modules together will unique such definitions at parse time via the
`identifier:` field, even if the nodes are `distinct`.

``` {.text}
!0 = !DIEnumerator(name: "SixKind", value: 7)
!1 = !DIEnumerator(name: "SevenKind", value: 7)
!2 = !DIEnumerator(name: "NegEightKind", value: -8)
!3 = !DICompositeType(tag: DW_TAG_enumeration_type, name: "Enum", file: !12,
                      line: 2, size: 32, align: 32, identifier: "_M4Enum",
                      elements: !{!0, !1, !2})
```

The following `tag:` values are valid:

``` {.text}
DW_TAG_array_type       = 1
DW_TAG_class_type       = 2
DW_TAG_enumeration_type = 4
DW_TAG_structure_type   = 19
DW_TAG_union_type       = 23
```

For `DW_TAG_array_type`, the `elements:` should be `subrange
descriptors <DISubrange>`{.interpreted-text role="ref"}, each
representing the range of subscripts at that level of indexing. The
`DIFlagVector` flag to `flags:` indicates that an array type is a native
packed vector.

For `DW_TAG_enumeration_type`, the `elements:` should be `enumerator
descriptors <DIEnumerator>`{.interpreted-text role="ref"}, each
representing the definition of an enumeration value for the set. All
enumeration type descriptors are collected in the `enums:` field of the
`compile unit <DICompileUnit>`{.interpreted-text role="ref"}.

For `DW_TAG_structure_type`, `DW_TAG_class_type`, and
`DW_TAG_union_type`, the `elements:` should be `derived types
<DIDerivedType>`{.interpreted-text role="ref"} with
`tag: DW_TAG_member`, `tag: DW_TAG_inheritance`, or
`tag: DW_TAG_friend`; or `subprograms <DISubprogram>`{.interpreted-text
role="ref"} with `isDefinition: false`.

##### DISubrange {#DISubrange}

`DISubrange` nodes are the elements for `DW_TAG_array_type` variants of
`DICompositeType`{.interpreted-text role="ref"}.

-   `count: -1` indicates an empty array.
-   `count: !9` describes the count with a
    `DILocalVariable`{.interpreted-text role="ref"}.
-   `count: !11` describes the count with a
    `DIGlobalVariable`{.interpreted-text role="ref"}.

``` {.text}
!0 = !DISubrange(count: 5, lowerBound: 0) ; array counting from 0
!1 = !DISubrange(count: 5, lowerBound: 1) ; array counting from 1
!2 = !DISubrange(count: -1) ; empty array.

; Scopes used in rest of example
!6 = !DIFile(filename: "vla.c", directory: "/path/to/file")
!7 = distinct !DICompileUnit(language: DW_LANG_C99, file: !6)
!8 = distinct !DISubprogram(name: "foo", scope: !7, file: !6, line: 5)

; Use of local variable as count value
!9 = !DIBasicType(name: "int", size: 32, encoding: DW_ATE_signed)
!10 = !DILocalVariable(name: "count", scope: !8, file: !6, line: 42, type: !9)
!11 = !DISubrange(count: !10, lowerBound: 0)

; Use of global variable as count value
!12 = !DIGlobalVariable(name: "count", scope: !8, file: !6, line: 22, type: !9)
!13 = !DISubrange(count: !12, lowerBound: 0)
```

##### DIEnumerator {#DIEnumerator}

`DIEnumerator` nodes are the elements for `DW_TAG_enumeration_type`
variants of `DICompositeType`{.interpreted-text role="ref"}.

``` {.text}
!0 = !DIEnumerator(name: "SixKind", value: 7)
!1 = !DIEnumerator(name: "SevenKind", value: 7)
!2 = !DIEnumerator(name: "NegEightKind", value: -8)
```

##### DITemplateTypeParameter

`DITemplateTypeParameter` nodes represent type parameters to generic
source language constructs. They are used (optionally) in
`DICompositeType`{.interpreted-text role="ref"} and
`DISubprogram`{.interpreted-text role="ref"} `templateParams:` fields.

``` {.text}
!0 = !DITemplateTypeParameter(name: "Ty", type: !1)
```

##### DITemplateValueParameter

`DITemplateValueParameter` nodes represent value parameters to generic
source language constructs. `tag:` defaults to
`DW_TAG_template_value_parameter`, but if specified can also be set to
`DW_TAG_GNU_template_template_param` or
`DW_TAG_GNU_template_param_pack`. They are used (optionally) in
`DICompositeType`{.interpreted-text role="ref"} and
`DISubprogram`{.interpreted-text role="ref"} `templateParams:` fields.

``` {.text}
!0 = !DITemplateValueParameter(name: "Ty", type: !1, value: i32 7)
```

##### DINamespace

`DINamespace` nodes represent namespaces in the source language.

``` {.text}
!0 = !DINamespace(name: "myawesomeproject", scope: !1, file: !2, line: 7)
```

##### DIGlobalVariable {#DIGlobalVariable}

`DIGlobalVariable` nodes represent global variables in the source
language.

``` {.text}
!0 = !DIGlobalVariable(name: "foo", linkageName: "foo", scope: !1,
                       file: !2, line: 7, type: !3, isLocal: true,
                       isDefinition: false, variable: i32* @foo,
                       declaration: !4)
```

All global variables should be referenced by the [globals:]{.title-ref}
field of a `compile unit <DICompileUnit>`{.interpreted-text role="ref"}.

##### DISubprogram {#DISubprogram}

`DISubprogram` nodes represent functions from the source language. A
`DISubprogram` may be attached to a function definition using `!dbg`
metadata. The `variables:` field points at
`variables <DILocalVariable>`{.interpreted-text role="ref"} that must be
retained, even if their IR counterparts are optimized out of the IR. The
`type:` field must point at an `DISubroutineType`{.interpreted-text
role="ref"}.

::: {#DISubprogramDeclaration}
When `isDefinition: false`, subprograms describe a declaration in the
type tree as opposed to a definition of a function. If the scope is a
composite type with an ODR `identifier:` and that does not set
`flags: DIFwdDecl`, then the subprogram declaration is uniqued based
only on its `linkageName:` and `scope:`.
:::

``` {.text}
define void @_Z3foov() !dbg !0 {
  ...
}

!0 = distinct !DISubprogram(name: "foo", linkageName: "_Zfoov", scope: !1,
                            file: !2, line: 7, type: !3, isLocal: true,
                            isDefinition: true, scopeLine: 8,
                            containingType: !4,
                            virtuality: DW_VIRTUALITY_pure_virtual,
                            virtualIndex: 10, flags: DIFlagPrototyped,
                            isOptimized: true, unit: !5, templateParams: !6,
                            declaration: !7, variables: !8, thrownTypes: !9)
```

##### DILexicalBlock {#DILexicalBlock}

`DILexicalBlock` nodes describe nested blocks within a `subprogram
<DISubprogram>`{.interpreted-text role="ref"}. The line number and
column numbers are used to distinguish two lexical blocks at same depth.
They are valid targets for `scope:` fields.

``` {.text}
!0 = distinct !DILexicalBlock(scope: !1, file: !2, line: 7, column: 35)
```

Usually lexical blocks are `distinct` to prevent node merging based on
operands.

##### DILexicalBlockFile {#DILexicalBlockFile}

`DILexicalBlockFile` nodes are used to discriminate between sections of
a `lexical block <DILexicalBlock>`{.interpreted-text role="ref"}. The
`file:` field can be changed to indicate textual inclusion, or the
`discriminator:` field can be used to discriminate between control flow
within a single block in the source language.

``` {.text}
!0 = !DILexicalBlock(scope: !3, file: !4, line: 7, column: 35)
!1 = !DILexicalBlockFile(scope: !0, file: !4, discriminator: 0)
!2 = !DILexicalBlockFile(scope: !0, file: !4, discriminator: 1)
```

##### DILocation {#DILocation}

`DILocation` nodes represent source debug locations. The `scope:` field
is mandatory, and points at an `DILexicalBlockFile`{.interpreted-text
role="ref"}, an `DILexicalBlock`{.interpreted-text role="ref"}, or an
`DISubprogram`{.interpreted-text role="ref"}.

``` {.text}
!0 = !DILocation(line: 2900, column: 42, scope: !1, inlinedAt: !2)
```

##### DILocalVariable {#DILocalVariable}

`DILocalVariable` nodes represent local variables in the source
language. If the `arg:` field is set to non-zero, then this variable is
a subprogram parameter, and it will be included in the `variables:`
field of its `DISubprogram`{.interpreted-text role="ref"}.

``` {.text}
!0 = !DILocalVariable(name: "this", arg: 1, scope: !3, file: !2, line: 7,
                      type: !3, flags: DIFlagArtificial)
!1 = !DILocalVariable(name: "x", arg: 2, scope: !4, file: !2, line: 7,
                      type: !3)
!2 = !DILocalVariable(name: "y", scope: !5, file: !2, line: 7, type: !3)
```

##### DIExpression

`DIExpression` nodes represent expressions that are inspired by the
DWARF expression language. They are used in
`debug intrinsics<dbg_intrinsics>`{.interpreted-text role="ref"} (such
as `llvm.dbg.declare` and `llvm.dbg.value`) to describe how the
referenced LLVM variable relates to the source language variable. Debug
intrinsics are interpreted left-to-right: start by pushing the
value/address operand of the intrinsic onto a stack, then repeatedly
push and evaluate opcodes from the DIExpression until the final variable
description is produced.

The current supported opcode vocabulary is limited:

-   `DW_OP_deref` dereferences the top of the expression stack.
-   `DW_OP_plus` pops the last two entries from the expression stack,
    adds them together and appends the result to the expression stack.
-   `DW_OP_minus` pops the last two entries from the expression stack,
    subtracts the last entry from the second last entry and appends the
    result to the expression stack.
-   `DW_OP_plus_uconst, 93` adds `93` to the working expression.
-   `DW_OP_LLVM_fragment, 16, 8` specifies the offset and size (`16` and
    `8` here, respectively) of the variable fragment from the working
    expression. Note that contrary to DW\_OP\_bit\_piece, the offset is
    describing the location within the described source variable.
-   `DW_OP_swap` swaps top two stack entries.
-   `DW_OP_xderef` provides extended dereference mechanism. The entry at
    the top of the stack is treated as an address. The second stack
    entry is treated as an address space identifier.
-   `DW_OP_stack_value` marks a constant value.

DWARF specifies three kinds of simple location descriptions: Register,
memory, and implicit location descriptions. Note that a location
description is defined over certain ranges of a program, i.e the
location of a variable may change over the course of the program.
Register and memory location descriptions describe the *concrete
location* of a source variable (in the sense that a debugger might
modify its value), whereas *implicit locations* describe merely the
actual *value* of a source variable which might not exist in registers
or in memory (see `DW_OP_stack_value`).

A `llvm.dbg.addr` or `llvm.dbg.declare` intrinsic describes an indirect
value (the address) of a source variable. The first operand of the
intrinsic must be an address of some kind. A DIExpression attached to
the intrinsic refines this address to produce a concrete location for
the source variable.

A `llvm.dbg.value` intrinsic describes the direct value of a source
variable. The first operand of the intrinsic may be a direct or indirect
value. A DIExpresion attached to the intrinsic refines the first operand
to produce a direct value. For example, if the first operand is an
indirect value, it may be necessary to insert `DW_OP_deref` into the
DIExpresion in order to produce a valid debug intrinsic.

::: {.note}
::: {.admonition-title}
Note
:::

A DIExpression is interpreted in the same way regardless of which kind
of debug intrinsic it\'s attached to.
:::

``` {.text}
!0 = !DIExpression(DW_OP_deref)
!1 = !DIExpression(DW_OP_plus_uconst, 3)
!1 = !DIExpression(DW_OP_constu, 3, DW_OP_plus)
!2 = !DIExpression(DW_OP_bit_piece, 3, 7)
!3 = !DIExpression(DW_OP_deref, DW_OP_constu, 3, DW_OP_plus, DW_OP_LLVM_fragment, 3, 7)
!4 = !DIExpression(DW_OP_constu, 2, DW_OP_swap, DW_OP_xderef)
!5 = !DIExpression(DW_OP_constu, 42, DW_OP_stack_value)
```

##### DIObjCProperty

`DIObjCProperty` nodes represent Objective-C property nodes.

``` {.text}
!3 = !DIObjCProperty(name: "foo", file: !1, line: 7, setter: "setFoo",
                     getter: "getFoo", attributes: 7, type: !2)
```

##### DIImportedEntity

`DIImportedEntity` nodes represent entities (such as modules) imported
into a compile unit.

``` {.text}
!2 = !DIImportedEntity(tag: DW_TAG_imported_module, name: "foo", scope: !0,
                       entity: !1, line: 7)
```

##### DIMacro

`DIMacro` nodes represent definition or undefinition of a macro
identifiers. The `name:` field is the macro identifier, followed by
macro parameters when defining a function-like macro, and the `value`
field is the token-string used to expand the macro identifier.

``` {.text}
!2 = !DIMacro(macinfo: DW_MACINFO_define, line: 7, name: "foo(x)",
              value: "((x) + 1)")
!3 = !DIMacro(macinfo: DW_MACINFO_undef, line: 30, name: "foo")
```

##### DIMacroFile

`DIMacroFile` nodes represent inclusion of source files. The `nodes:`
field is a list of `DIMacro` and `DIMacroFile` nodes that appear in the
included source file.

``` {.text}
!2 = !DIMacroFile(macinfo: DW_MACINFO_start_file, line: 7, file: !2,
                  nodes: !3)
```

#### \'`tbaa`\' Metadata

In LLVM IR, memory does not have types, so LLVM\'s own type system is
not suitable for doing type based alias analysis (TBAA). Instead,
metadata is added to the IR to describe a type system of a higher level
language. This can be used to implement C/C++ strict type aliasing
rules, but it can also be used to implement custom alias analysis
behavior for other languages.

This description of LLVM\'s TBAA system is broken into two parts:
`Semantics<tbaa_node_semantics>`{.interpreted-text role="ref"} talks
about high level issues, and
`Representation<tbaa_node_representation>`{.interpreted-text role="ref"}
talks about the metadata encoding of various entities.

It is always possible to trace any TBAA node to a \"root\" TBAA node
(details in the
`Representation<tbaa_node_representation>`{.interpreted-text role="ref"}
section). TBAA nodes with different roots have an unknown aliasing
relationship, and LLVM conservatively infers `MayAlias` between them.
The rules mentioned in this section only pertain to TBAA nodes living
under the same root.

##### Semantics {#tbaa_node_semantics}

The TBAA metadata system, referred to as \"struct path TBAA\" (not to be
confused with `tbaa.struct`), consists of the following high level
concepts: *Type Descriptors*, further subdivided into scalar type
descriptors and struct type descriptors; and *Access Tags*.

**Type descriptors** describe the type system of the higher level
language being compiled. **Scalar type descriptors** describe types that
do not contain other types. Each scalar type has a parent type, which
must also be a scalar type or the TBAA root. Via this parent relation,
scalar types within a TBAA root form a tree. **Struct type descriptors**
denote types that contain a sequence of other type descriptors, at known
offsets. These contained type descriptors can either be struct type
descriptors themselves or scalar type descriptors.

**Access tags** are metadata nodes attached to load and store
instructions. Access tags use type descriptors to describe the
*location* being accessed in terms of the type system of the higher
level language. Access tags are tuples consisting of a base type, an
access type and an offset. The base type is a scalar type descriptor or
a struct type descriptor, the access type is a scalar type descriptor,
and the offset is a constant integer.

The access tag `(BaseTy, AccessTy, Offset)` can describe one of two
things:

> -   If `BaseTy` is a struct type, the tag describes a memory access
>     (load or store) of a value of type `AccessTy` contained in the
>     struct type `BaseTy` at offset `Offset`.
> -   If `BaseTy` is a scalar type, `Offset` must be 0 and `BaseTy` and
>     `AccessTy` must be the same; and the access tag describes a scalar
>     access with scalar type `AccessTy`.

We first define an `ImmediateParent` relation on `(BaseTy, Offset)`
tuples this way:

> -   If `BaseTy` is a scalar type then `ImmediateParent(BaseTy, 0)` is
>     `(ParentTy, 0)` where `ParentTy` is the parent of the scalar type
>     as described in the TBAA metadata.
>     `ImmediateParent(BaseTy, Offset)` is undefined if `Offset` is
>     non-zero.
> -   If `BaseTy` is a struct type then
>     `ImmediateParent(BaseTy, Offset)` is `(NewTy, NewOffset)` where
>     `NewTy` is the type contained in `BaseTy` at offset `Offset` and
>     `NewOffset` is `Offset` adjusted to be relative within that inner
>     type.

A memory access with an access tag `(BaseTy1, AccessTy1, Offset1)`
aliases a memory access with an access tag
`(BaseTy2, AccessTy2, Offset2)` if either `(BaseTy1, Offset1)` is
reachable from `(Base2, Offset2)` via the `Parent` relation or vice
versa.

As a concrete example, the type descriptor graph for the following
program

``` {.c}
struct Inner {
  int i;    // offset 0
  float f;  // offset 4
};

struct Outer {
  float f;  // offset 0
  double d; // offset 4
  struct Inner inner_a;  // offset 12
};

void f(struct Outer* outer, struct Inner* inner, float* f, int* i, char* c) {
  outer->f = 0;            // tag0: (OuterStructTy, FloatScalarTy, 0)
  outer->inner_a.i = 0;    // tag1: (OuterStructTy, IntScalarTy, 12)
  outer->inner_a.f = 0.0;  // tag2: (OuterStructTy, FloatScalarTy, 16)
  *f = 0.0;                // tag3: (FloatScalarTy, FloatScalarTy, 0)
}
```

is (note that in C and C++, `char` can be used to access any arbitrary
type):

``` {.text}
Root = "TBAA Root"
CharScalarTy = ("char", Root, 0)
FloatScalarTy = ("float", CharScalarTy, 0)
DoubleScalarTy = ("double", CharScalarTy, 0)
IntScalarTy = ("int", CharScalarTy, 0)
InnerStructTy = {"Inner" (IntScalarTy, 0), (FloatScalarTy, 4)}
OuterStructTy = {"Outer", (FloatScalarTy, 0), (DoubleScalarTy, 4),
                 (InnerStructTy, 12)}
```

with (e.g.) `ImmediateParent(OuterStructTy, 12)` = `(InnerStructTy, 0)`,
`ImmediateParent(InnerStructTy, 0)` = `(IntScalarTy, 0)`, and
`ImmediateParent(IntScalarTy, 0)` = `(CharScalarTy, 0)`.

##### Representation {#tbaa_node_representation}

The root node of a TBAA type hierarchy is an `MDNode` with 0 operands or
with exactly one `MDString` operand.

Scalar type descriptors are represented as an `MDNode` s with two
operands. The first operand is an `MDString` denoting the name of the
struct type. LLVM does not assign meaning to the value of this operand,
it only cares about it being an `MDString`. The second operand is an
`MDNode` which points to the parent for said scalar type descriptor,
which is either another scalar type descriptor or the TBAA root. Scalar
type descriptors can have an optional third argument, but that must be
the constant integer zero.

Struct type descriptors are represented as `MDNode` s with an odd number
of operands greater than 1. The first operand is an `MDString` denoting
the name of the struct type. Like in scalar type descriptors the actual
value of this name operand is irrelevant to LLVM. After the name
operand, the struct type descriptors have a sequence of alternating
`MDNode` and `ConstantInt` operands. With N starting from 1, the 2N - 1
th operand, an `MDNode`, denotes a contained field, and the 2N th
operand, a `ConstantInt`, is the offset of the said contained field. The
offsets must be in non-decreasing order.

Access tags are represented as `MDNode` s with either 3 or 4 operands.
The first operand is an `MDNode` pointing to the node representing the
base type. The second operand is an `MDNode` pointing to the node
representing the access type. The third operand is a `ConstantInt` that
states the offset of the access. If a fourth field is present, it must
be a `ConstantInt` valued at 0 or 1. If it is 1 then the access tag
states that the location being accessed is \"constant\" (meaning
`pointsToConstantMemory` should return true; see [other useful
AliasAnalysis methods](AliasAnalysis.html#OtherItfs)). The TBAA root of
the access type and the base type of an access tag must be the same, and
that is the TBAA root of the access tag.

#### \'`tbaa.struct`\' Metadata

The `llvm.memcpy <int_memcpy>`{.interpreted-text role="ref"} is often
used to implement aggregate assignment operations in C and similar
languages, however it is defined to copy a contiguous region of memory,
which is more than strictly necessary for aggregate types which contain
holes due to padding. Also, it doesn\'t contain any TBAA information
about the fields of the aggregate.

`!tbaa.struct` metadata can describe which memory subregions in a memcpy
are padding and what the TBAA tags of the struct are.

The current metadata format is very simple. `!tbaa.struct` metadata
nodes are a list of operands which are in conceptual groups of three.
For each group of three, the first operand gives the byte offset of a
field in bytes, the second gives its size in bytes, and the third gives
its tbaa tag. e.g.:

``` {.llvm}
!4 = !{ i64 0, i64 4, !1, i64 8, i64 4, !2 }
```

This describes a struct with two fields. The first is at offset 0 bytes
with size 4 bytes, and has tbaa tag !1. The second is at offset 8 bytes
and has size 4 bytes and has tbaa tag !2.

Note that the fields need not be contiguous. In this example, there is a
4 byte gap between the two fields. This gap represents padding which
does not carry useful data and need not be preserved.

#### \'`noalias`\' and \'`alias.scope`\' Metadata

`noalias` and `alias.scope` metadata provide the ability to specify
generic noalias memory-access sets. This means that some collection of
memory access instructions (loads, stores, memory-accessing calls, etc.)
that carry `noalias` metadata can specifically be specified not to alias
with some other collection of memory access instructions that carry
`alias.scope` metadata. Each type of metadata specifies a list of scopes
where each scope has an id and a domain.

When evaluating an aliasing query, if for some domain, the set of scopes
with that domain in one instruction\'s `alias.scope` list is a subset of
(or equal to) the set of scopes for that domain in another
instruction\'s `noalias` list, then the two memory accesses are assumed
not to alias.

Because scopes in one domain don\'t affect scopes in other domains,
separate domains can be used to compose multiple independent noalias
sets. This is used for example during inlining. As the noalias function
parameters are turned into noalias scope metadata, a new domain is used
every time the function is inlined.

The metadata identifying each domain is itself a list containing one or
two entries. The first entry is the name of the domain. Note that if the
name is a string then it can be combined across functions and
translation units. A self-reference can be used to create globally
unique domain names. A descriptive string may optionally be provided as
a second list entry.

The metadata identifying each scope is also itself a list containing two
or three entries. The first entry is the name of the scope. Note that if
the name is a string then it can be combined across functions and
translation units. A self-reference can be used to create globally
unique scope names. A metadata reference to the scope\'s domain is the
second entry. A descriptive string may optionally be provided as a third
list entry.

For example,

``` {.llvm}
; Two scope domains:
!0 = !{!0}
!1 = !{!1}

; Some scopes in these domains:
!2 = !{!2, !0}
!3 = !{!3, !0}
!4 = !{!4, !1}

; Some scope lists:
!5 = !{!4} ; A list containing only scope !4
!6 = !{!4, !3, !2}
!7 = !{!3}

; These two instructions don't alias:
%0 = load float, float* %c, align 4, !alias.scope !5
store float %0, float* %arrayidx.i, align 4, !noalias !5

; These two instructions also don't alias (for domain !1, the set of scopes
; in the !alias.scope equals that in the !noalias list):
%2 = load float, float* %c, align 4, !alias.scope !5
store float %2, float* %arrayidx.i2, align 4, !noalias !6

; These two instructions may alias (for domain !0, the set of scopes in
; the !noalias list is not a superset of, or equal to, the scopes in the
; !alias.scope list):
%2 = load float, float* %c, align 4, !alias.scope !6
store float %0, float* %arrayidx.i, align 4, !noalias !7
```

#### \'`fpmath`\' Metadata

`fpmath` metadata may be attached to any instruction of floating-point
type. It can be used to express the maximum acceptable error in the
result of that instruction, in ULPs, thus potentially allowing the
compiler to use a more efficient but less accurate method of computing
it. ULP is defined as follows:

> If `x` is a real number that lies between two finite consecutive
> floating-point numbers `a` and `b`, without being equal to one of
> them, then `ulp(x) = |b - a|`, otherwise `ulp(x)` is the distance
> between the two non-equal finite floating-point numbers nearest `x`.
> Moreover, `ulp(NaN)` is `NaN`.

The metadata node shall consist of a single positive float type number
representing the maximum relative error, for example:

``` {.llvm}
!0 = !{ float 2.5 } ; maximum acceptable inaccuracy is 2.5 ULPs
```

#### \'`range`\' Metadata

`range` metadata may be attached only to `load`, `call` and `invoke` of
integer types. It expresses the possible ranges the loaded value or the
value returned by the called function at this call site is in. If the
loaded or returned value is not in the specified range, the behavior is
undefined. The ranges are represented with a flattened list of integers.
The loaded value or the value returned is known to be in the union of
the ranges defined by each consecutive pair. Each pair has the following
properties:

-   The type must match the type loaded by the instruction.
-   The pair `a,b` represents the range `[a,b)`.
-   Both `a` and `b` are constants.
-   The range is allowed to wrap.
-   The range should not represent the full or empty set. That is,
    `a!=b`.

In addition, the pairs must be in signed order of the lower bound and
they must be non-contiguous.

Examples:

``` {.llvm}
%a = load i8, i8* %x, align 1, !range !0 ; Can only be 0 or 1
%b = load i8, i8* %y, align 1, !range !1 ; Can only be 255 (-1), 0 or 1
%c = call i8 @foo(),       !range !2 ; Can only be 0, 1, 3, 4 or 5
%d = invoke i8 @bar() to label %cont
       unwind label %lpad, !range !3 ; Can only be -2, -1, 3, 4 or 5
```

> \... !0 = !{ i8 0, i8 2 } !1 = !{ i8 255, i8 2 } !2 = !{ i8 0, i8 2,
> i8 3, i8 6 } !3 = !{ i8 -2, i8 0, i8 3, i8 6 }

#### \'`absolute_symbol`\' Metadata

`absolute_symbol` metadata may be attached to a global variable
declaration. It marks the declaration as a reference to an absolute
symbol, which causes the backend to use absolute relocations for the
symbol even in position independent code, and expresses the possible
ranges that the global variable\'s *address* (not its value) is in, in
the same format as `range` metadata, with the extension that the pair
`all-ones,all-ones` may be used to represent the full set.

Example (assuming 64-bit pointers):

``` {.llvm}
@a = external global i8, !absolute_symbol !0 ; Absolute symbol in range [0,256)
@b = external global i8, !absolute_symbol !1 ; Absolute symbol in range [0,2^64)
```

> \... !0 = !{ i64 0, i64 256 } !1 = !{ i64 -1, i64 -1 }

#### \'`callees`\' Metadata

`callees` metadata may be attached to indirect call sites. If `callees`
metadata is attached to a call site, and any callee is not among the set
of functions provided by the metadata, the behavior is undefined. The
intent of this metadata is to facilitate optimizations such as
indirect-call promotion. For example, in the code below, the call
instruction may only target the `add` or `sub` functions:

``` {.llvm}
%result = call i64 %binop(i64 %x, i64 %y), !callees !0

...
!0 = !{i64 (i64, i64)* @add, i64 (i64, i64)* @sub}
```

#### \'`unpredictable`\' Metadata

`unpredictable` metadata may be attached to any branch or switch
instruction. It can be used to express the unpredictability of control
flow. Similar to the llvm.expect intrinsic, it may be used to alter
optimizations related to compare and branch instructions. The metadata
is treated as a boolean value; if it exists, it signals that the branch
or switch that it is attached to is completely unpredictable.

#### \'`llvm.loop`\'

It is sometimes useful to attach information to loop constructs.
Currently, loop metadata is implemented as metadata attached to the
branch instruction in the loop latch block. This type of metadata refer
to a metadata node that is guaranteed to be separate for each loop. The
loop identifier metadata is specified with the name `llvm.loop`.

The loop identifier metadata is implemented using a metadata that refers
to itself to avoid merging it with any other identifier metadata, e.g.,
during module linkage or function inlining. That is, each loop should
refer to their own identification metadata even if they reside in
separate functions. The following example contains loop identifier
metadata for two separate loop constructs:

``` {.llvm}
!0 = !{!0}
!1 = !{!1}
```

The loop identifier metadata can be used to specify additional per-loop
metadata. Any operands after the first operand can be treated as
user-defined metadata. For example the `llvm.loop.unroll.count` suggests
an unroll factor to the loop unroller:

``` {.llvm}
br i1 %exitcond, label %._crit_edge, label %.lr.ph, !llvm.loop !0
```

> \... !0 = !{!0, !1} !1 = !{!\"llvm.loop.unroll.count\", i32 4}

#### \'`llvm.loop.disable_nonforced`\'

This metadata disables all optional loop transformations unless
explicitly instructed using other transformation metdata such as
`llvm.loop.unroll.enable`. That is, no heuristic will try to determine
whether a transformation is profitable. The purpose is to avoid that the
loop is transformed to a different loop before an explicitly requested
(forced) transformation is applied. For instance, loop fusion can make
other transformations impossible. Mandatory loop canonicalizations such
as loop rotation are still applied.

It is recommended to use this metadata in addition to any llvm.loop.\*
transformation directive. Also, any loop should have at most one
directive applied to it (and a sequence of transformations built using
followup-attributes). Otherwise, which transformation will be applied
depends on implementation details such as the pass pipeline order.

See `transformation-metadata`{.interpreted-text role="ref"} for details.

#### \'`llvm.loop.vectorize`\' and \'`llvm.loop.interleave`\'

Metadata prefixed with `llvm.loop.vectorize` or `llvm.loop.interleave`
are used to control per-loop vectorization and interleaving parameters
such as vectorization width and interleave count. These metadata should
be used in conjunction with `llvm.loop` loop identification metadata.
The `llvm.loop.vectorize` and `llvm.loop.interleave` metadata are only
optimization hints and the optimizer will only interleave and vectorize
loops if it believes it is safe to do so. The
`llvm.loop.parallel_accesses` metadata which contains information about
loop-carried memory dependencies can be helpful in determining the
safety of these transformations.

#### \'`llvm.loop.interleave.count`\' Metadata

This metadata suggests an interleave count to the loop interleaver. The
first operand is the string `llvm.loop.interleave.count` and the second
operand is an integer specifying the interleave count. For example:

``` {.llvm}
!0 = !{!"llvm.loop.interleave.count", i32 4}
```

Note that setting `llvm.loop.interleave.count` to 1 disables
interleaving multiple iterations of the loop. If
`llvm.loop.interleave.count` is set to 0 then the interleave count will
be determined automatically.

#### \'`llvm.loop.vectorize.enable`\' Metadata

This metadata selectively enables or disables vectorization for the
loop. The first operand is the string `llvm.loop.vectorize.enable` and
the second operand is a bit. If the bit operand value is 1 vectorization
is enabled. A value of 0 disables vectorization:

``` {.llvm}
!0 = !{!"llvm.loop.vectorize.enable", i1 0}
!1 = !{!"llvm.loop.vectorize.enable", i1 1}
```

#### \'`llvm.loop.vectorize.width`\' Metadata

This metadata sets the target width of the vectorizer. The first operand
is the string `llvm.loop.vectorize.width` and the second operand is an
integer specifying the width. For example:

``` {.llvm}
!0 = !{!"llvm.loop.vectorize.width", i32 4}
```

Note that setting `llvm.loop.vectorize.width` to 1 disables
vectorization of the loop. If `llvm.loop.vectorize.width` is set to 0 or
if the loop does not have this metadata the width will be determined
automatically.

#### \'`llvm.loop.vectorize.followup_vectorized`\' Metadata

This metadata defines which loop attributes the vectorized loop will
have. See `transformation-metadata`{.interpreted-text role="ref"} for
details.

#### \'`llvm.loop.vectorize.followup_epilogue`\' Metadata

This metadata defines which loop attributes the epilogue will have. The
epilogue is not vectorized and is executed when either the vectorized
loop is not known to preserve semantics (because e.g., it processes two
arrays that are found to alias by a runtime check) or for the last
iterations that do not fill a complete set of vector lanes. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.vectorize.followup_all`\' Metadata

Attributes in the metadata will be added to both the vectorized and
epilogue loop. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll`\'

Metadata prefixed with `llvm.loop.unroll` are loop unrolling
optimization hints such as the unroll factor. `llvm.loop.unroll`
metadata should be used in conjunction with `llvm.loop` loop
identification metadata. The `llvm.loop.unroll` metadata are only
optimization hints and the unrolling will only be performed if the
optimizer believes it is safe to do so.

#### \'`llvm.loop.unroll.count`\' Metadata

This metadata suggests an unroll factor to the loop unroller. The first
operand is the string `llvm.loop.unroll.count` and the second operand is
a positive integer specifying the unroll factor. For example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll.count", i32 4}
```

If the trip count of the loop is less than the unroll count the loop
will be partially unrolled.

#### \'`llvm.loop.unroll.disable`\' Metadata

This metadata disables loop unrolling. The metadata has a single operand
which is the string `llvm.loop.unroll.disable`. For example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll.disable"}
```

#### \'`llvm.loop.unroll.runtime.disable`\' Metadata

This metadata disables runtime loop unrolling. The metadata has a single
operand which is the string `llvm.loop.unroll.runtime.disable`. For
example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll.runtime.disable"}
```

#### \'`llvm.loop.unroll.enable`\' Metadata

This metadata suggests that the loop should be fully unrolled if the
trip count is known at compile time and partially unrolled if the trip
count is not known at compile time. The metadata has a single operand
which is the string `llvm.loop.unroll.enable`. For example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll.enable"}
```

#### \'`llvm.loop.unroll.full`\' Metadata

This metadata suggests that the loop should be unrolled fully. The
metadata has a single operand which is the string
`llvm.loop.unroll.full`. For example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll.full"}
```

#### \'`llvm.loop.unroll.followup`\' Metadata

This metadata defines which loop attributes the unrolled loop will have.
See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll.followup_remainder`\' Metadata

This metadata defines which loop attributes the remainder loop after
partial/runtime unrolling will have. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll_and_jam`\'

This metadata is treated very similarly to the `llvm.loop.unroll`
metadata above, but affect the unroll and jam pass. In addition any loop
with `llvm.loop.unroll` metadata but no `llvm.loop.unroll_and_jam`
metadata will disable unroll and jam (so `llvm.loop.unroll` metadata
will be left to the unroller, plus `llvm.loop.unroll.disable` metadata
will disable unroll and jam too.)

The metadata for unroll and jam otherwise is the same as for `unroll`.
`llvm.loop.unroll_and_jam.enable`, `llvm.loop.unroll_and_jam.disable`
and `llvm.loop.unroll_and_jam.count` do the same as for unroll.
`llvm.loop.unroll_and_jam.full` is not supported. Again these are only
hints and the normal safety checks will still be performed.

#### \'`llvm.loop.unroll_and_jam.count`\' Metadata

This metadata suggests an unroll and jam factor to use, similarly to
`llvm.loop.unroll.count`. The first operand is the string
`llvm.loop.unroll_and_jam.count` and the second operand is a positive
integer specifying the unroll factor. For example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll_and_jam.count", i32 4}
```

If the trip count of the loop is less than the unroll count the loop
will be partially unroll and jammed.

#### \'`llvm.loop.unroll_and_jam.disable`\' Metadata

This metadata disables loop unroll and jamming. The metadata has a
single operand which is the string `llvm.loop.unroll_and_jam.disable`.
For example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll_and_jam.disable"}
```

#### \'`llvm.loop.unroll_and_jam.enable`\' Metadata

This metadata suggests that the loop should be fully unroll and jammed
if the trip count is known at compile time and partially unrolled if the
trip count is not known at compile time. The metadata has a single
operand which is the string `llvm.loop.unroll_and_jam.enable`. For
example:

``` {.llvm}
!0 = !{!"llvm.loop.unroll_and_jam.enable"}
```

#### \'`llvm.loop.unroll_and_jam.followup_outer`\' Metadata

This metadata defines which loop attributes the outer unrolled loop will
have. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll_and_jam.followup_inner`\' Metadata

This metadata defines which loop attributes the inner jammed loop will
have. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll_and_jam.followup_remainder_outer`\' Metadata

This metadata defines which attributes the epilogue of the outer loop
will have. This loop is usually unrolled, meaning there is no such loop.
This attribute will be ignored in this case. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll_and_jam.followup_remainder_inner`\' Metadata

This metadata defines which attributes the inner loop of the epilogue
will have. The outer epilogue will usually be unrolled, meaning there
can be multiple inner remainder loops. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.unroll_and_jam.followup_all`\' Metadata

Attributes specified in the metadata is added to all
`llvm.loop.unroll_and_jam.*` loops. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.licm_versioning.disable`\' Metadata

This metadata indicates that the loop should not be versioned for the
purpose of enabling loop-invariant code motion (LICM). The metadata has
a single operand which is the string
`llvm.loop.licm_versioning.disable`. For example:

``` {.llvm}
!0 = !{!"llvm.loop.licm_versioning.disable"}
```

#### \'`llvm.loop.distribute.enable`\' Metadata

Loop distribution allows splitting a loop into multiple loops.
Currently, this is only performed if the entire loop cannot be
vectorized due to unsafe memory dependencies. The transformation will
attempt to isolate the unsafe dependencies into their own loop.

This metadata can be used to selectively enable or disable distribution
of the loop. The first operand is the string
`llvm.loop.distribute.enable` and the second operand is a bit. If the
bit operand value is 1 distribution is enabled. A value of 0 disables
distribution:

``` {.llvm}
!0 = !{!"llvm.loop.distribute.enable", i1 0}
!1 = !{!"llvm.loop.distribute.enable", i1 1}
```

This metadata should be used in conjunction with `llvm.loop` loop
identification metadata.

#### \'`llvm.loop.distribute.followup_coincident`\' Metadata

This metadata defines which attributes extracted loops with no cyclic
dependencies will have (i.e. can be vectorized). See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.distribute.followup_sequential`\' Metadata

This metadata defines which attributes the isolated loops with unsafe
memory dependencies will have. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.distribute.followup_fallback`\' Metadata

If loop versioning is necessary, this metadata defined the attributes
the non-distributed fallback version will have. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.loop.distribute.followup_all`\' Metadata

Thes attributes in this metdata is added to all followup loops of the
loop distribution pass. See
`Transformation Metadata <transformation-metadata>`{.interpreted-text
role="ref"} for details.

#### \'`llvm.access.group`\' Metadata

`llvm.access.group` metadata can be attached to any instruction that
potentially accesses memory. It can point to a single distinct metadata
node, which we call access group. This node represents all memory access
instructions referring to it via `llvm.access.group`. When an
instruction belongs to multiple access groups, it can also point to a
list of accesses groups, illustrated by the following example.

``` {.llvm}
%val = load i32, i32* %arrayidx, !llvm.access.group !0
...
!0 = !{!1, !2}
!1 = distinct !{}
!2 = distinct !{}
```

It is illegal for the list node to be empty since it might be confused
with an access group.

The access group metadata node must be \'distinct\' to avoid collapsing
multiple access groups by content. A access group metadata node must
always be empty which can be used to distinguish an access group
metadata node from a list of access groups. Being empty avoids the
situation that the content must be updated which, because metadata is
immutable by design, would required finding and updating all references
to the access group node.

The access group can be used to refer to a memory access instruction
without pointing to it directly (which is not possible in global
metadata). Currently, the only metadata making use of it is
`llvm.loop.parallel_accesses`.

#### \'`llvm.loop.parallel_accesses`\' Metadata

The `llvm.loop.parallel_accesses` metadata refers to one or more access
group metadata nodes (see `llvm.access.group`). It denotes that no
loop-carried memory dependence exist between it and other instructions
in the loop with this metadata.

Let `m1` and `m2` be two instructions that both have the
`llvm.access.group` metadata to the access group `g1`, respectively `g2`
(which might be identical). If a loop contains both access groups in its
`llvm.loop.parallel_accesses` metadata, then the compiler can assume
that there is no dependency between `m1` and `m2` carried by this loop.
Instructions that belong to multiple access groups are considered having
this property if at least one of the access groups matches the
`llvm.loop.parallel_accesses` list.

If all memory-accessing instructions in a loop have
`llvm.loop.parallel_accesses` metadata that refers to that loop, then
the loop has no loop carried memory dependences and is considered to be
a parallel loop.

Note that if not all memory access instructions belong to an access
group referred to by `llvm.loop.parallel_accesses`, then the loop must
not be considered trivially parallel. Additional memory dependence
analysis is required to make that determination. As a fail safe
mechanism, this causes loops that were originally parallel to be
considered sequential (if optimization passes that are unaware of the
parallel semantics insert new memory instructions into the loop body).

Example of a loop that is considered parallel due to its correct use of
both `llvm.access.group` and `llvm.loop.parallel_accesses` metadata
types.

``` {.llvm}
for.body:
  ...
  %val0 = load i32, i32* %arrayidx, !llvm.access.group !1
  ...
  store i32 %val0, i32* %arrayidx1, !llvm.access.group !1
  ...
  br i1 %exitcond, label %for.end, label %for.body, !llvm.loop !0

for.end:
...
!0 = distinct !{!0, !{!"llvm.loop.parallel_accesses", !1}}
!1 = distinct !{}
```

It is also possible to have nested parallel loops:

``` {.llvm}
outer.for.body:
  ...
  %val1 = load i32, i32* %arrayidx3, !llvm.access.group !4
  ...
  br label %inner.for.body

inner.for.body:
  ...
  %val0 = load i32, i32* %arrayidx1, !llvm.access.group !3
  ...
  store i32 %val0, i32* %arrayidx2, !llvm.access.group !3
  ...
  br i1 %exitcond, label %inner.for.end, label %inner.for.body, !llvm.loop !1

inner.for.end:
  ...
  store i32 %val1, i32* %arrayidx4, !llvm.access.group !4
  ...
  br i1 %exitcond, label %outer.for.end, label %outer.for.body, !llvm.loop !2

outer.for.end:                                          ; preds = %for.body
...
!1 = distinct !{!1, !{!"llvm.loop.parallel_accesses", !3}}     ; metadata for the inner loop
!2 = distinct !{!2, !{!"llvm.loop.parallel_accesses", !3, !4}} ; metadata for the outer loop
!3 = distinct !{} ; access group for instructions in the inner loop (which are implicitly contained in outer loop as well)
!4 = distinct !{} ; access group for instructions in the outer, but not the inner loop
```

#### \'`irr_loop`\' Metadata

`irr_loop` metadata may be attached to the terminator instruction of a
basic block that\'s an irreducible loop header (note that an irreducible
loop has more than once header basic blocks.) If `irr_loop` metadata is
attached to the terminator instruction of a basic block that is not
really an irreducible loop header, the behavior is undefined. The intent
of this metadata is to improve the accuracy of the block frequency
propagation. For example, in the code below, the block `header0` may
have a loop header weight (relative to the other headers of the
irreducible loop) of 100:

``` {.llvm}
header0:
...
br i1 %cmp, label %t1, label %t2, !irr_loop !0

...
!0 = !{"loop_header_weight", i64 100}
```

Irreducible loop header weights are typically based on profile data.

#### \'`invariant.group`\' Metadata

The experimental `invariant.group` metadata may be attached to
`load`/`store` instructions referencing a single metadata with no
entries. The existence of the `invariant.group` metadata on the
instruction tells the optimizer that every `load` and `store` to the
same pointer operand can be assumed to load or store the same value (but
see the `llvm.launder.invariant.group` intrinsic which affects when two
pointers are considered the same). Pointers returned by bitcast or
getelementptr with only zero indices are considered the same.

Examples:

``` {.llvm}
@unknownPtr = external global i8
...
%ptr = alloca i8
store i8 42, i8* %ptr, !invariant.group !0
call void @foo(i8* %ptr)

%a = load i8, i8* %ptr, !invariant.group !0 ; Can assume that value under %ptr didn't change
call void @foo(i8* %ptr)

%newPtr = call i8* @getPointer(i8* %ptr)
%c = load i8, i8* %newPtr, !invariant.group !0 ; Can't assume anything, because we only have information about %ptr

%unknownValue = load i8, i8* @unknownPtr
store i8 %unknownValue, i8* %ptr, !invariant.group !0 ; Can assume that %unknownValue == 42

call void @foo(i8* %ptr)
%newPtr2 = call i8* @llvm.launder.invariant.group(i8* %ptr)
%d = load i8, i8* %newPtr2, !invariant.group !0  ; Can't step through launder.invariant.group to get value of %ptr

...
declare void @foo(i8*)
declare i8* @getPointer(i8*)
declare i8* @llvm.launder.invariant.group(i8*)

!0 = !{}
```

The invariant.group metadata must be dropped when replacing one pointer
by another based on aliasing information. This is because
invariant.group is tied to the SSA value of the pointer operand.

``` {.llvm}
%v = load i8, i8* %x, !invariant.group !0
; if %x mustalias %y then we can replace the above instruction with
%v = load i8, i8* %y
```

Note that this is an experimental feature, which means that its
semantics might change in the future.

#### \'`type`\' Metadata

See `TypeMetadata`{.interpreted-text role="doc"}.

#### \'`associated`\' Metadata

The `associated` metadata may be attached to a global object declaration
with a single argument that references another global object.

This metadata prevents discarding of the global object in linker GC
unless the referenced object is also discarded. The linker support for
this feature is spotty. For best compatibility, globals carrying this
metadata may also:

-   Be in a comdat with the referenced global.
-   Be in \@llvm.compiler.used.
-   Have an explicit section with a name which is a valid C identifier.

It does not have any effect on non-ELF targets.

Example:

``` {.text}
$a = comdat any
@a = global i32 1, comdat $a
@b = internal global i32 2, comdat $a, section "abc", !associated !0
!0 = !{i32* @a}
```

#### \'`prof`\' Metadata

The `prof` metadata is used to record profile data in the IR. The first
operand of the metadata node indicates the profile metadata type. There
are currently 3 types:
`branch_weights<prof_node_branch_weights>`{.interpreted-text
role="ref"},
`function_entry_count<prof_node_function_entry_count>`{.interpreted-text
role="ref"}, and `VP<prof_node_VP>`{.interpreted-text role="ref"}.

##### branch\_weights {#prof_node_branch_weights}

Branch weight metadata attached to a branch, select, switch or call
instruction represents the likeliness of the associated branch being
taken. For more information, see
`BranchWeightMetadata`{.interpreted-text role="doc"}.

##### function\_entry\_count {#prof_node_function_entry_count}

Function entry count metadata can be attached to function definitions to
record the number of times the function is called. Used with BFI
information, it is also used to derive the basic block profile count.
For more information, see `BranchWeightMetadata`{.interpreted-text
role="doc"}.

##### VP {#prof_node_VP}

VP (value profile) metadata can be attached to instructions that have
value profile information. Currently this is indirect calls (where it
records the hottest callees) and calls to memory intrinsics such as
memcpy, memmove, and memset (where it records the hottest byte lengths).

Each VP metadata node contains \"VP\" string, then a uint32\_t value for
the value profiling kind, a uint64\_t value for the total number of
times the instruction is executed, followed by uint64\_t value and
execution count pairs. The value profiling kind is 0 for indirect call
targets and 1 for memory operations. For indirect call targets, each
profile value is a hash of the callee function name, and for memory
operations each value is the byte length.

Note that the value counts do not need to add up to the total count
listed in the third operand (in practice only the top hottest values are
tracked and reported).

Indirect call example:

``` {.llvm}
call void %f(), !prof !1
!1 = !{!"VP", i32 0, i64 1600, i64 7651369219802541373, i64 1030, i64 -4377547752858689819, i64 410}
```

Note that the VP type is 0 (the second operand), which indicates this is
an indirect call value profile data. The third operand indicates that
the indirect call executed 1600 times. The 4th and 6th operands give the
hashes of the 2 hottest target functions\' names (this is the same hash
used to represent function names in the profile database), and the 5th
and 7th operands give the execution count that each of the respective
prior target functions was called.

