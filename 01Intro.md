LLVM Language Reference Manual
==============================

Abstract
--------

This document is a reference manual for the LLVM assembly language. LLVM
is a Static Single Assignment (SSA) based representation that provides
type safety, low-level operations, flexibility, and the capability of
representing \'all\' high-level languages cleanly. It is the common code
representation used throughout all phases of the LLVM compilation
strategy.

Introduction
------------

The LLVM code representation is designed to be used in three different
forms: as an in-memory compiler IR, as an on-disk bitcode representation
(suitable for fast loading by a Just-In-Time compiler), and as a human
readable assembly language representation. This allows LLVM to provide a
powerful intermediate representation for efficient compiler
transformations and analysis, while providing a natural means to debug
and visualize the transformations. The three different forms of LLVM are
all equivalent. This document describes the human readable
representation and notation.

The LLVM representation aims to be light-weight and low-level while
being expressive, typed, and extensible at the same time. It aims to be
a \"universal IR\" of sorts, by being at a low enough level that
high-level ideas may be cleanly mapped to it (similar to how
microprocessors are \"universal IR\'s\", allowing many source languages
to be mapped to them). By providing type information, LLVM can be used
as the target of optimizations: for example, through pointer analysis,
it can be proven that a C automatic variable is never accessed outside
of the current function, allowing it to be promoted to a simple SSA
value instead of a memory location.

### Well-Formedness {#wellformed}

It is important to note that this document describes \'well formed\'
LLVM assembly language. There is a difference between what the parser
accepts and what is considered \'well formed\'. For example, the
following instruction is syntactically okay, but not well formed:

``` {.llvm}
%x = add i32 1, %x
```

because the definition of `%x` does not dominate all of its uses. The
LLVM infrastructure provides a verification pass that may be used to
verify that an LLVM module is well formed. This pass is automatically
run by the parser after parsing input assembly and by the optimizer
before it outputs bitcode. The violations pointed out by the verifier
pass indicate bugs in transformation passes or input to the parser.

Identifiers
-----------

LLVM identifiers come in two basic types: global and local. Global
identifiers (functions, global variables) begin with the `'@'`
character. Local identifiers (register names, types) begin with the
`'%'` character. Additionally, there are three different formats for
identifiers, for different purposes:

1.  Named values are represented as a string of characters with their
    prefix. For example, `%foo`, `@DivisionByZero`,
    `%a.really.long.identifier`. The actual regular expression used is
    \'`[%@][-a-zA-Z$._][-a-zA-Z$._0-9]*`\'. Identifiers that require
    other characters in their names can be surrounded with quotes.
    Special characters may be escaped using `"\xx"` where `xx` is the
    ASCII code for the character in hexadecimal. In this way, any
    character can be used in a name value, even quotes themselves. The
    `"\01"` prefix can be used on global values to suppress mangling.
2.  Unnamed values are represented as an unsigned numeric value with
    their prefix. For example, `%12`, `@2`, `%44`.
3.  Constants, which are described in the section
    [Constants](#constants) below.

LLVM requires that values start with a prefix for two reasons: Compilers
don\'t need to worry about name clashes with reserved words, and the set
of reserved words may be expanded in the future without penalty.
Additionally, unnamed identifiers allow a compiler to quickly come up
with a temporary variable without having to avoid symbol table
conflicts.

Reserved words in LLVM are very similar to reserved words in other
languages. There are keywords for different opcodes (\'`add`\',
\'`bitcast`\', \'`ret`\', etc\...), for primitive type names
(\'`void`\', \'`i32`\', etc\...), and others. These reserved words
cannot conflict with variable names, because none of them start with a
prefix character (`'%'` or `'@'`).

Here is an example of LLVM code to multiply the integer variable
\'`%X`\' by 8:

The easy way:

``` {.llvm}
%result = mul i32 %X, 8
```

After strength reduction:

``` {.llvm}
%result = shl i32 %X, 3
```

And the hard way:

``` {.llvm}
%0 = add i32 %X, %X           ; yields i32:%0
%1 = add i32 %0, %0           ; yields i32:%1
%result = add i32 %1, %1
```

This last way of multiplying `%X` by 8 illustrates several important
lexical features of LLVM:

1.  Comments are delimited with a \'`;`\' and go until the end of line.
2.  Unnamed temporaries are created when the result of a computation is
    not assigned to a named value.
3.  Unnamed temporaries are numbered sequentially (using a per-function
    incrementing counter, starting with 0). Note that basic blocks and
    unnamed function parameters are included in this numbering. For
    example, if the entry basic block is not given a label name and all
    function parameters are named, then it will get number 0.

It also shows a convention that we follow in this document. When
demonstrating instructions, we will follow an instruction with a comment
that defines the type and name of value produced.

