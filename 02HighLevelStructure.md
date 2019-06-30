High Level Structure
--------------------

### Module Structure

LLVM programs are composed of `Module`\'s, each of which is a
translation unit of the input programs. Each module consists of
functions, global variables, and symbol table entries. Modules may be
combined together with the LLVM linker, which merges function (and
global variable) definitions, resolves forward declarations, and merges
symbol table entries. Here is an example of the \"hello world\" module:

``` {.llvm}
; Declare the string constant as a global constant.
@.str = private unnamed_addr constant [13 x i8] c"hello world\0A\00"

; External declaration of the puts function
declare i32 @puts(i8* nocapture) nounwind

; Definition of main function
define i32 @main() {   ; i32()*
  ; Convert [13 x i8]* to i8*...
  %cast210 = getelementptr [13 x i8], [13 x i8]* @.str, i64 0, i64 0

  ; Call puts function to write out the string to stdout.
  call i32 @puts(i8* %cast210)
  ret i32 0
}

; Named metadata
!0 = !{i32 42, null, !"string"}
!foo = !{!0}
```

This example is made up of a
`global variable <globalvars>`{.interpreted-text role="ref"} named
\"`.str`\", an external declaration of the \"`puts`\" function, a
`function definition <functionstructure>`{.interpreted-text role="ref"}
for \"`main`\" and
`named metadata <namedmetadatastructure>`{.interpreted-text role="ref"}
\"`foo`\".

In general, a module is made up of a list of global values (where both
functions and global variables are global values). Global values are
represented by a pointer to a memory location (in this case, a pointer
to an array of char, and a pointer to a function), and have one of the
following `linkage types <linkage>`{.interpreted-text role="ref"}.

### Linkage Types

All Global Variables and Functions have one of the following types of
linkage:

`private`

Global values with \"`private`\" linkage are only directly
    accessible by objects in the current module. In particular, linking
    code into a module with a private global value may cause the private
    to be renamed as necessary to avoid collisions. Because the symbol
    is private to the module, all references can be updated. This
    doesn\'t show up in any symbol table in the object file.

`internal`

Similar to private, but the value shows as a local symbol
    (`STB_LOCAL` in the case of ELF) in the object file. This
    corresponds to the notion of the \'`static`\' keyword in C.

`available_externally`

Globals with \"`available_externally`\" linkage are never emitted
    into the object file corresponding to the LLVM module. From the
    linker\'s perspective, an `available_externally` global is
    equivalent to an external declaration. They exist to allow inlining
    and other optimizations to take place given knowledge of the
    definition of the global, which is known to be somewhere outside the
    module. Globals with `available_externally` linkage are allowed to
    be discarded at will, and allow inlining and other optimizations.
    This linkage type is only allowed on definitions, not declarations.

`linkonce`

Globals with \"`linkonce`\" linkage are merged with other globals of
    the same name when linkage occurs. This can be used to implement
    some forms of inline functions, templates, or other code which must
    be generated in each translation unit that uses it, but where the
    body may be overridden with a more definitive definition later.
    Unreferenced `linkonce` globals are allowed to be discarded. Note
    that `linkonce` linkage does not actually allow the optimizer to
    inline the body of this function into callers because it doesn\'t
    know if this definition of the function is the definitive definition
    within the program or whether it will be overridden by a stronger
    definition. To enable inlining and other optimizations, use
    \"`linkonce_odr`\" linkage.

`weak`

\"`weak`\" linkage has the same merging semantics as `linkonce`
    linkage, except that unreferenced globals with `weak` linkage may
    not be discarded. This is used for globals that are declared
    \"weak\" in C source code.

`common`

\"`common`\" linkage is most similar to \"`weak`\" linkage, but they
    are used for tentative definitions in C, such as \"`int X;`\" at
    global scope. Symbols with \"`common`\" linkage are merged in the
    same way as `weak symbols`, and they may not be deleted if
    unreferenced. `common` symbols may not have an explicit section,
    must have a zero initializer, and may not be marked
    \'`constant <globalvars>`{.interpreted-text role="ref"}\'. Functions
    and aliases may not have common linkage.



`appending`

\"`appending`\" linkage may only be applied to global variables of
    pointer to array type. When two global variables with appending
    linkage are linked together, the two global arrays are appended
    together. This is the LLVM, typesafe, equivalent of having the
    system linker append together \"sections\" with identical names when
    .o files are linked.

    Unfortunately this doesn\'t correspond to any feature in .o files,
    so it can only be used for variables like `llvm.global_ctors` which
    llvm interprets specially.

`extern_weak`

The semantics of this linkage follow the ELF object file model: the
    symbol is weak until linked, if not linked, the symbol becomes null
    instead of being an undefined reference.

`linkonce_odr`, `weak_odr`

Some languages allow differing globals to be merged, such as two
    functions with different semantics. Other languages, such as `C++`,
    ensure that only equivalent globals are ever merged (the \"one
    definition rule\" \-\-- \"ODR\"). Such languages can use the
    `linkonce_odr` and `weak_odr` linkage types to indicate that the
    global will only be merged with equivalent globals. These linkage
    types are otherwise the same as their non-`odr` versions.

`external`

If none of the above identifiers are used, the global is externally
    visible, meaning that it participates in linkage and can be used to
    resolve external symbol references.


It is illegal for a function *declaration* to have any linkage type
other than `external` or `extern_weak`.

### Calling Conventions

LLVM `functions <functionstructure>`{.interpreted-text role="ref"},
`calls <i_call>`{.interpreted-text role="ref"} and
`invokes <i_invoke>`{.interpreted-text role="ref"} can all have an
optional calling convention specified for the call. The calling
convention of any pair of dynamic caller/callee must match, or the
behavior of the program is undefined. The following calling conventions
are supported by LLVM, and more may be added in the future:

\"`ccc`\" - The C calling convention

This calling convention (the default if no other calling convention
    is specified) matches the target C calling conventions. This calling
    convention supports varargs function calls and tolerates some
    mismatch in the declared prototype and implemented declaration of
    the function (as does normal C).

\"`fastcc`\" - The fast calling convention

This calling convention attempts to make calls as fast as possible
    (e.g. by passing things in registers). This calling convention
    allows the target to use whatever tricks it wants to produce fast
    code for the target, without having to conform to an externally
    specified ABI (Application Binary Interface). [Tail calls can only
    be optimized when this, the GHC or the HiPE convention is
    used.](CodeGenerator.html#id80) This calling convention does not
    support varargs and requires the prototype of all callees to exactly
    match the prototype of the function definition.

\"`coldcc`\" - The cold calling convention

This calling convention attempts to make code in the caller as
    efficient as possible under the assumption that the call is not
    commonly executed. As such, these calls often preserve all registers
    so that the call does not break any live ranges in the caller side.
    This calling convention does not support varargs and requires the
    prototype of all callees to exactly match the prototype of the
    function definition. Furthermore the inliner doesn\'t consider such
    function calls for inlining.

\"`cc 10`\" - GHC convention

This calling convention has been implemented specifically for use by
    the [Glasgow Haskell Compiler (GHC)](http://www.haskell.org/ghc). It
    passes everything in registers, going to extremes to achieve this by
    disabling callee save registers. This calling convention should not
    be used lightly but only for specific situations such as an
    alternative to the *register pinning* performance technique often
    used when implementing functional programming languages. At the
    moment only X86 supports this convention and it has the following
    limitations:

    -   On *X86-32* only supports up to 4 bit type parameters. No
        floating-point types are supported.
    -   On *X86-64* only supports up to 10 bit type parameters and 6
        floating-point parameters.

    This calling convention supports [tail call
    optimization](CodeGenerator.html#id80) but requires both the caller
    and callee are using it.

\"`cc 11`\" - The HiPE calling convention

This calling convention has been implemented specifically for use by
    the [High-Performance Erlang
    (HiPE)](http://www.it.uu.se/research/group/hipe/) compiler, *the*
    native code compiler of the [Ericsson\'s Open Source Erlang/OTP
    system](http://www.erlang.org/download.shtml). It uses more
    registers for argument passing than the ordinary C calling
    convention and defines no callee-saved registers. The calling
    convention properly supports [tail call
    optimization](CodeGenerator.html#id80) but requires that both the
    caller and the callee use it. It uses a *register pinning*
    mechanism, similar to GHC\'s convention, for keeping frequently
    accessed runtime components pinned to specific hardware registers.
    At the moment only X86 supports this convention (both 32 and 64
    bit).

\"`webkit_jscc`\" - WebKit\'s JavaScript calling convention

This calling convention has been implemented for [WebKit FTL
    JIT](https://trac.webkit.org/wiki/FTLJIT). It passes arguments on
    the stack right to left (as cdecl does), and returns a value in the
    platform\'s customary return register.

\"`anyregcc`\" - Dynamic calling convention for code patching

This is a special convention that supports patching an arbitrary
    code sequence in place of a call site. This convention forces the
    call arguments into registers but allows them to be dynamically
    allocated. This can currently only be used with calls to
    llvm.experimental.patchpoint because only this intrinsic records the
    location of its arguments in a side table. See
    `StackMaps`{.interpreted-text role="doc"}.

\"`preserve_mostcc`\" - The [PreserveMost]{.title-ref} calling convention

This calling convention attempts to make the code in the caller as
    unintrusive as possible. This convention behaves identically to the
    [C]{.title-ref} calling convention on how arguments and return
    values are passed, but it uses a different set of
    caller/callee-saved registers. This alleviates the burden of saving
    and recovering a large register set before and after the call in the
    caller. If the arguments are passed in callee-saved registers, then
    they will be preserved by the callee across the call. This doesn\'t
    apply for values returned in callee-saved registers.

    -   On X86-64 the callee preserves all general purpose registers,
        except for R11. R11 can be used as a scratch register.
        Floating-point registers (XMMs/YMMs) are not preserved and need
        to be saved by the caller.

    The idea behind this convention is to support calls to runtime
    functions that have a hot path and a cold path. The hot path is
    usually a small piece of code that doesn\'t use many registers. The
    cold path might need to call out to another function and therefore
    only needs to preserve the caller-saved registers, which haven\'t
    already been saved by the caller. The [PreserveMost]{.title-ref}
    calling convention is very similar to the [cold]{.title-ref} calling
    convention in terms of caller/callee-saved registers, but they are
    used for different types of function calls. [coldcc]{.title-ref} is
    for function calls that are rarely executed, whereas
    [preserve\_mostcc]{.title-ref} function calls are intended to be on
    the hot path and definitely executed a lot. Furthermore
    [preserve\_mostcc]{.title-ref} doesn\'t prevent the inliner from
    inlining the function call.

    This calling convention will be used by a future version of the
    ObjectiveC runtime and should therefore still be considered
    experimental at this time. Although this convention was created to
    optimize certain runtime calls to the ObjectiveC runtime, it is not
    limited to this runtime and might be used by other runtimes in the
    future too. The current implementation only supports X86-64, but the
    intention is to support more architectures in the future.

\"`preserve_allcc`\" - The [PreserveAll]{.title-ref} calling convention

This calling convention attempts to make the code in the caller even
    less intrusive than the [PreserveMost]{.title-ref} calling
    convention. This calling convention also behaves identical to the
    [C]{.title-ref} calling convention on how arguments and return
    values are passed, but it uses a different set of
    caller/callee-saved registers. This removes the burden of saving and
    recovering a large register set before and after the call in the
    caller. If the arguments are passed in callee-saved registers, then
    they will be preserved by the callee across the call. This doesn\'t
    apply for values returned in callee-saved registers.

    -   On X86-64 the callee preserves all general purpose registers,
        except for R11. R11 can be used as a scratch register.
        Furthermore it also preserves all floating-point registers
        (XMMs/YMMs).

    The idea behind this convention is to support calls to runtime
    functions that don\'t need to call out to any other functions.

    This calling convention, like the [PreserveMost]{.title-ref} calling
    convention, will be used by a future version of the ObjectiveC
    runtime and should be considered experimental at this time.

\"`cxx_fast_tlscc`\" - The [CXX\_FAST\_TLS]{.title-ref} calling convention for access functions

Clang generates an access function to access C++-style TLS. The
    access function generally has an entry block, an exit block and an
    initialization block that is run at the first time. The entry and
    exit blocks can access a few TLS IR variables, each access will be
    lowered to a platform-specific sequence.

    This calling convention aims to minimize overhead in the caller by
    preserving as many registers as possible (all the registers that are
    perserved on the fast path, composed of the entry and exit blocks).

    This calling convention behaves identical to the [C]{.title-ref}
    calling convention on how arguments and return values are passed,
    but it uses a different set of caller/callee-saved registers.

    Given that each platform has its own lowering sequence, hence its
    own set of preserved registers, we can\'t use the existing
    [PreserveMost]{.title-ref}.

    -   On X86-64 the callee preserves all general purpose registers,
        except for RDI and RAX.

\"`swiftcc`\" - This calling convention is used for Swift language.

-   On X86-64 RCX and R8 are available for additional integer
        returns, and XMM2 and XMM3 are available for additional
        FP/vector returns.
    -   On iOS platforms, we use AAPCS-VFP calling convention.

\"`cc <n>`\" - Numbered convention

Any calling convention may be specified by number, allowing
    target-specific calling conventions to be used. Target specific
    calling conventions start at 64.

More calling conventions can be added/defined on an as-needed basis, to
support Pascal conventions or any other well-known target-independent
convention.

### Visibility Styles

All Global Variables and Functions have one of the following visibility
styles:

\"`default`\" - Default style

On targets that use the ELF object file format, default visibility
    means that the declaration is visible to other modules and, in
    shared libraries, means that the declared entity may be overridden.
    On Darwin, default visibility means that the declaration is visible
    to other modules. Default visibility corresponds to \"external
    linkage\" in the language.

\"`hidden`\" - Hidden style

Two declarations of an object with hidden visibility refer to the
    same object if they are in the same shared object. Usually, hidden
    visibility indicates that the symbol will not be placed into the
    dynamic symbol table, so no other module (executable or shared
    library) can reference it directly.

\"`protected`\" - Protected style

On ELF, protected visibility indicates that the symbol will be
    placed in the dynamic symbol table, but that references within the
    defining module will bind to the local symbol. That is, the symbol
    cannot be overridden by another module.

A symbol with `internal` or `private` linkage must have `default`
visibility.

### DLL Storage Classes

All Global Variables, Functions and Aliases can have one of the
following DLL storage class:

`dllimport`

\"`dllimport`\" causes the compiler to reference a function or
    variable via a global pointer to a pointer that is set up by the DLL
    exporting the symbol. On Microsoft Windows targets, the pointer name
    is formed by combining `__imp_` and the function or variable name.

`dllexport`

\"`dllexport`\" causes the compiler to provide a global pointer to a
    pointer in a DLL, so that it can be referenced with the `dllimport`
    attribute. On Microsoft Windows targets, the pointer name is formed
    by combining `__imp_` and the function or variable name. Since this
    storage class exists for defining a dll interface, the compiler,
    assembler and linker know it is externally referenced and must
    refrain from deleting the symbol.

### Thread Local Storage Models

A variable may be defined as `thread_local`, which means that it will
not be shared by threads (each thread will have a separated copy of the
variable). Not all targets support thread-local variables. Optionally, a
TLS model may be specified:

`localdynamic`

For variables that are only used within the current shared library.

`initialexec`

For variables in modules that will not be loaded dynamically.

`localexec`

For variables defined in the executable and only used within it.

If no explicit model is given, the \"general dynamic\" model is used.

The models correspond to the ELF TLS models; see [ELF Handling For
Thread-Local Storage](http://people.redhat.com/drepper/tls.pdf) for more
information on under which circumstances the different models may be
used. The target may choose a different TLS model if the specified model
is not supported, or if a better choice of model can be made.

A model can also be specified in an alias, but then it only governs how
the alias is accessed. It will not have any effect in the aliasee.

For platforms without linker support of ELF TLS model, the
-femulated-tls flag can be used to generate GCC compatible emulated TLS
code.

### Runtime Preemption Specifiers

Global variables, functions and aliases may have an optional runtime
preemption specifier. If a preemption specifier isn\'t given explicitly,
then a symbol is assumed to be `dso_preemptable`.

`dso_preemptable`

Indicates that the function or variable may be replaced by a symbol
    from outside the linkage unit at runtime.

`dso_local`

The compiler may assume that a function or variable marked as
    `dso_local` will resolve to a symbol within the same linkage unit.
    Direct access will be generated even if the definition is not within
    this compilation unit.

### Structure Types

LLVM IR allows you to specify both \"identified\" and \"literal\"
`structure
types <t_struct>`{.interpreted-text role="ref"}. Literal types are
uniqued structurally, but identified types are never uniqued. An
`opaque structural type <t_opaque>`{.interpreted-text role="ref"} can
also be used to forward declare a type that is not yet available.

An example of an identified structure specification is:

``` {.llvm}
%mytype = type { %mytype*, i32 }
```

Prior to the LLVM 3.0 release, identified types were structurally
uniqued. Only literal types are uniqued in recent versions of LLVM.

### Non-Integral Pointer Type

Note: non-integral pointer types are a work in progress, and they should
be considered experimental at this time.

LLVM IR optionally allows the frontend to denote pointers in certain
address spaces as \"non-integral\" via the
`datalayout string<langref_datalayout>`{.interpreted-text role="ref"}.
Non-integral pointer types represent pointers that have an *unspecified*
bitwise representation; that is, the integral representation may be
target dependent or unstable (not backed by a fixed integer).

`inttoptr` instructions converting integers to non-integral pointer
types are ill-typed, and so are `ptrtoint` instructions converting
values of non-integral pointer types to integers. Vector versions of
said instructions are ill-typed as well.

### Global Variables

Global variables define regions of memory allocated at compilation time
instead of run-time.

Global variable definitions must be initialized.

Global variables in other translation units can also be declared, in
which case they don\'t have an initializer.

Either global variable definitions or declarations may have an explicit
section to be placed in and may have an optional explicit alignment
specified. If there is a mismatch between the explicit or inferred
section information for the variable declaration and its definition the
resulting behavior is undefined.

A variable may be defined as a global `constant`, which indicates that
the contents of the variable will **never** be modified (enabling better
optimization, allowing the global data to be placed in the read-only
section of an executable, etc). Note that variables that need runtime
initialization cannot be marked `constant` as there is a store to the
variable.

LLVM explicitly allows *declarations* of global variables to be marked
constant, even if the final definition of the global is not. This
capability can be used to enable slightly better optimization of the
program, but requires the language definition to guarantee that
optimizations based on the \'constantness\' are valid for the
translation units that do not include the definition.

As SSA values, global variables define pointer values that are in scope
(i.e. they dominate) all basic blocks in the program. Global variables
always define a pointer to their \"content\" type because they describe
a region of memory, and all memory objects in LLVM are accessed through
pointers.

Global variables can be marked with `unnamed_addr` which indicates that
the address is not significant, only the content. Constants marked like
this can be merged with other constants if they have the same
initializer. Note that a constant with significant address *can* be
merged with a `unnamed_addr` constant, the result being a constant whose
address is significant.

If the `local_unnamed_addr` attribute is given, the address is known to
not be significant within the module.

A global variable may be declared to reside in a target-specific
numbered address space. For targets that support them, address spaces
may affect how optimizations are performed and/or what target
instructions are used to access the variable. The default address space
is zero. The address space qualifier must precede any other attributes.

LLVM allows an explicit section to be specified for globals. If the
target supports it, it will emit globals to the section specified.
Additionally, the global can placed in a comdat if the target has the
necessary support.

External declarations may have an explicit section specified. Section
information is retained in LLVM IR for targets that make use of this
information. Attaching section information to an external declaration is
an assertion that its definition is located in the specified section. If
the definition is located in a different section, the behavior is
undefined.

By default, global initializers are optimized by assuming that global
variables defined within the module are not modified from their initial
values before the start of the global initializer. This is true even for
variables potentially accessible from outside the module, including
those with external linkage or appearing in `@llvm.used` or dllexported
variables. This assumption may be suppressed by marking the variable
with `externally_initialized`.

An explicit alignment may be specified for a global, which must be a
power of 2. If not present, or if the alignment is set to zero, the
alignment of the global is set by the target to whatever it feels
convenient. If an explicit alignment is specified, the global is forced
to have exactly that alignment. Targets and optimizers are not allowed
to over-align the global if the global has an assigned section. In this
case, the extra alignment could be observable: for example, code could
assume that the globals are densely packed in their section and try to
iterate over them as an array, alignment padding would break this
iteration. The maximum alignment is `1 << 29`.

Globals can also have a
`DLL storage class <dllstorageclass>`{.interpreted-text role="ref"}, an
optional
`runtime preemption specifier <runtime_preemption_model>`{.interpreted-text
role="ref"}, an optional `global attributes <glattrs>`{.interpreted-text
role="ref"} and an optional list of attached
`metadata <metadata>`{.interpreted-text role="ref"}.

Variables and aliases can have a
`Thread Local Storage Model <tls_model>`{.interpreted-text role="ref"}.

Syntax:

    @<GlobalVarName> = [Linkage] [PreemptionSpecifier] [Visibility]
                       [DLLStorageClass] [ThreadLocal]
                       [(unnamed_addr|local_unnamed_addr)] [AddrSpace]
                       [ExternallyInitialized]
                       <global | constant> <Type> [<InitializerConstant>]
                       [, section "name"] [, comdat [($name)]]
                       [, align <Alignment>] (, !name !N)*

For example, the following defines a global in a numbered address space
with an initializer, section, and alignment:

``` {.llvm}
@G = addrspace(5) constant float 1.0, section "foo", align 4
```

The following example just declares a global variable

``` {.llvm}
@G = external global i32
```

The following example defines a thread-local global with the
`initialexec` TLS model:

``` {.llvm}
@G = thread_local(initialexec) global i32 0, align 4
```

### Functions

LLVM function definitions consist of the \"`define`\" keyword, an
optional `linkage type <linkage>`{.interpreted-text role="ref"}, an
optional `runtime preemption
specifier <runtime_preemption_model>`{.interpreted-text role="ref"}, an
optional `visibility
style <visibility>`{.interpreted-text role="ref"}, an optional
`DLL storage class <dllstorageclass>`{.interpreted-text role="ref"}, an
optional `calling convention <callingconv>`{.interpreted-text
role="ref"}, an optional `unnamed_addr` attribute, a return type, an
optional `parameter attribute <paramattrs>`{.interpreted-text
role="ref"} for the return type, a function name, a (possibly empty)
argument list (each with optional `parameter
attributes <paramattrs>`{.interpreted-text role="ref"}), optional
`function attributes <fnattrs>`{.interpreted-text role="ref"}, an
optional address space, an optional section, an optional alignment, an
optional `comdat <langref_comdats>`{.interpreted-text role="ref"}, an
optional `garbage collector name <gc>`{.interpreted-text role="ref"}, an
optional `prefix <prefixdata>`{.interpreted-text role="ref"}, an
optional `prologue <prologuedata>`{.interpreted-text role="ref"}, an
optional `personality <personalityfn>`{.interpreted-text role="ref"}, an
optional list of attached `metadata <metadata>`{.interpreted-text
role="ref"}, an opening curly brace, a list of basic blocks, and a
closing curly brace.

LLVM function declarations consist of the \"`declare`\" keyword, an
optional `linkage type <linkage>`{.interpreted-text role="ref"}, an
optional `visibility style
<visibility>`{.interpreted-text role="ref"}, an optional
`DLL storage class <dllstorageclass>`{.interpreted-text role="ref"}, an
optional `calling convention <callingconv>`{.interpreted-text
role="ref"}, an optional `unnamed_addr` or `local_unnamed_addr`
attribute, an optional address space, a return type, an optional
`parameter attribute <paramattrs>`{.interpreted-text role="ref"} for the
return type, a function name, a possibly empty list of arguments, an
optional alignment, an optional `garbage
collector name <gc>`{.interpreted-text role="ref"}, an optional
`prefix <prefixdata>`{.interpreted-text role="ref"}, and an optional
`prologue <prologuedata>`{.interpreted-text role="ref"}.

A function definition contains a list of basic blocks, forming the CFG
(Control Flow Graph) for the function. Each basic block may optionally
start with a label (giving the basic block a symbol table entry),
contains a list of instructions, and ends with a
`terminator <terminators>`{.interpreted-text role="ref"} instruction
(such as a branch or function return). If an explicit label is not
provided, a block is assigned an implicit numbered label, using the next
value from the same counter as used for unnamed temporaries
(`see above<identifiers>`{.interpreted-text role="ref"}). For example,
if a function entry block does not have an explicit label, it will be
assigned label \"%0\", then the first unnamed temporary in that block
will be \"%1\", etc.

The first basic block in a function is special in two ways: it is
immediately executed on entrance to the function, and it is not allowed
to have predecessor basic blocks (i.e. there can not be any branches to
the entry block of a function). Because the block can have no
predecessors, it also cannot have any
`PHI nodes <i_phi>`{.interpreted-text role="ref"}.

LLVM allows an explicit section to be specified for functions. If the
target supports it, it will emit functions to the section specified.
Additionally, the function can be placed in a COMDAT.

An explicit alignment may be specified for a function. If not present,
or if the alignment is set to zero, the alignment of the function is set
by the target to whatever it feels convenient. If an explicit alignment
is specified, the function is forced to have at least that much
alignment. All alignments must be a power of 2.

If the `unnamed_addr` attribute is given, the address is known to not be
significant and two identical functions can be merged.

If the `local_unnamed_addr` attribute is given, the address is known to
not be significant within the module.

If an explicit address space is not given, it will default to the
program address space from the
`datalayout string<langref_datalayout>`{.interpreted-text role="ref"}.

Syntax:

    define [linkage] [PreemptionSpecifier] [visibility] [DLLStorageClass]
           [cconv] [ret attrs]
           <ResultType> @<FunctionName> ([argument list])
           [(unnamed_addr|local_unnamed_addr)] [AddrSpace] [fn Attrs]
           [section "name"] [comdat [($name)]] [align N] [gc] [prefix Constant]
           [prologue Constant] [personality Constant] (!name !N)* { ... }

The argument list is a comma separated sequence of arguments where each
argument is of the following form:

Syntax:

    <type> [parameter Attrs] [name]

### Aliases

Aliases, unlike function or variables, don\'t create any new data. They
are just a new symbol and metadata for an existing position.

Aliases have a name and an aliasee that is either a global value or a
constant expression.

Aliases may have an optional `linkage type <linkage>`{.interpreted-text
role="ref"}, an optional
`runtime preemption specifier <runtime_preemption_model>`{.interpreted-text
role="ref"}, an optional
`visibility style <visibility>`{.interpreted-text role="ref"}, an
optional `DLL storage class
<dllstorageclass>`{.interpreted-text role="ref"} and an optional
`tls model <tls_model>`{.interpreted-text role="ref"}.

Syntax:

    @<Name> = [Linkage] [PreemptionSpecifier] [Visibility] [DLLStorageClass] [ThreadLocal] [(unnamed_addr|local_unnamed_addr)] alias <AliaseeTy>, <AliaseeTy>* @<Aliasee>

The linkage must be one of `private`, `internal`, `linkonce`, `weak`,
`linkonce_odr`, `weak_odr`, `external`. Note that some system linkers
might not correctly handle dropping a weak symbol that is aliased.

Aliases that are not `unnamed_addr` are guaranteed to have the same
address as the aliasee expression. `unnamed_addr` ones are only
guaranteed to point to the same content.

If the `local_unnamed_addr` attribute is given, the address is known to
not be significant within the module.

Since aliases are only a second name, some restrictions apply, of which
some can only be checked when producing an object file:

-   The expression defining the aliasee must be computable at assembly
    time. Since it is just a name, no relocations can be used.
-   No alias in the expression can be weak as the possibility of the
    intermediate alias being overridden cannot be represented in an
    object file.
-   No global value in the expression can be a declaration, since that
    would require a relocation, which is not possible.

### IFuncs

IFuncs, like as aliases, don\'t create any new data or func. They are
just a new symbol that dynamic linker resolves at runtime by calling a
resolver function.

IFuncs have a name and a resolver that is a function called by dynamic
linker that returns address of another function associated with the
name.

IFunc may have an optional `linkage type <linkage>`{.interpreted-text
role="ref"} and an optional
`visibility style <visibility>`{.interpreted-text role="ref"}.

Syntax:

    @<Name> = [Linkage] [Visibility] ifunc <IFuncTy>, <ResolverTy>* @<Resolver>

### Comdats

Comdat IR provides access to COFF and ELF object file COMDAT
functionality.

Comdats have a name which represents the COMDAT key. All global objects
that specify this key will only end up in the final object file if the
linker chooses that key over some other key. Aliases are placed in the
same COMDAT that their aliasee computes to, if any.

Comdats have a selection kind to provide input on how the linker should
choose between keys in two different object files.

Syntax:

    $<Name> = comdat SelectionKind

The selection kind must be one of the following:

`any`

The linker may choose any COMDAT key, the choice is arbitrary.

`exactmatch`

The linker may choose any COMDAT key but the sections must contain
    the same data.

`largest`

The linker will choose the section containing the largest COMDAT
    key.

`noduplicates`

The linker requires that only section with this COMDAT key exist.

`samesize`

The linker may choose any COMDAT key but the sections must contain
    the same amount of data.

Note that the Mach-O platform doesn\'t support COMDATs, and ELF and
WebAssembly only support `any` as a selection kind.

Here is an example of a COMDAT group where a function will only be
selected if the COMDAT key\'s section is the largest:

``` {.text}
$foo = comdat largest
@foo = global i32 2, comdat($foo)

define void @bar() comdat($foo) {
  ret void
}
```

As a syntactic sugar the `$name` can be omitted if the name is the same
as the global name:

``` {.text}
$foo = comdat any
@foo = global i32 2, comdat
```

In a COFF object file, this will create a COMDAT section with selection
kind `IMAGE_COMDAT_SELECT_LARGEST` containing the contents of the `@foo`
symbol and another COMDAT section with selection kind
`IMAGE_COMDAT_SELECT_ASSOCIATIVE` which is associated with the first
COMDAT section and contains the contents of the `@bar` symbol.

There are some restrictions on the properties of the global object. It,
or an alias to it, must have the same name as the COMDAT group when
targeting COFF. The contents and size of this object may be used during
link-time to determine which COMDAT groups get selected depending on the
selection kind. Because the name of the object must match the name of
the COMDAT group, the linkage of the global object must not be local;
local symbols can get renamed if a collision occurs in the symbol table.

The combined use of COMDATS and section attributes may yield surprising
results. For example:

``` {.text}
$foo = comdat any
$bar = comdat any
@g1 = global i32 42, section "sec", comdat($foo)
@g2 = global i32 42, section "sec", comdat($bar)
```

From the object file perspective, this requires the creation of two
sections with the same name. This is necessary because both globals
belong to different COMDAT groups and COMDATs, at the object file level,
are represented by sections.

Note that certain IR constructs like global variables and functions may
create COMDATs in the object file in addition to any which are specified
using COMDAT IR. This arises when the code generator is configured to
emit globals in individual sections (e.g. when
[-data-sections]{.title-ref} or [-function-sections]{.title-ref} is
supplied to [llc]{.title-ref}).

### Named Metadata

Named metadata is a collection of metadata. `Metadata
nodes <metadata>`{.interpreted-text role="ref"} (but not metadata
strings) are the only valid operands for a named metadata.

1.  Named metadata are represented as a string of characters with the
    metadata prefix. The rules for metadata names are the same as for
    identifiers, but quoted names are not allowed. `"\xx"` type escapes
    are still valid, which allows any character to be part of a name.

Syntax:

    ; Some unnamed metadata nodes, which are referenced by the named metadata.
    !0 = !{!"zero"}
    !1 = !{!"one"}
    !2 = !{!"two"}
    ; A named metadata.
    !name = !{!0, !1, !2}

### Parameter Attributes

The return type and each parameter of a function type may have a set of
*parameter attributes* associated with them. Parameter attributes are
used to communicate additional information about the result or
parameters of a function. Parameter attributes are considered to be part
of the function, not of the function type, so functions with different
parameter attributes can have the same function type.

Parameter attributes are simple keywords that follow the type specified.
If multiple parameter attributes are needed, they are space separated.
For example:

``` {.llvm}
declare i32 @printf(i8* noalias nocapture, ...)
declare i32 @atoi(i8 zeroext)
declare signext i8 @returns_signed_char()
```

Note that any attributes for the function result (`nounwind`,
`readonly`) come immediately after the argument list.

Currently, only the following parameter attributes are defined:

`zeroext`

This indicates to the code generator that the parameter or return
    value should be zero-extended to the extent required by the
    target\'s ABI by the caller (for a parameter) or the callee (for a
    return value).

`signext`

This indicates to the code generator that the parameter or return
    value should be sign-extended to the extent required by the
    target\'s ABI (which is usually 32-bits) by the caller (for a
    parameter) or the callee (for a return value).

`inreg`

This indicates that this parameter or return value should be treated
    in a special target-dependent fashion while emitting code for a
    function call or return (usually, by putting it in a register as
    opposed to memory, though some targets use it to distinguish between
    two different kinds of registers). Use of this attribute is
    target-specific.

`byval`

This indicates that the pointer parameter should really be passed by
    value to the function. The attribute implies that a hidden copy of
    the pointee is made between the caller and the callee, so the callee
    is unable to modify the value in the caller. This attribute is only
    valid on LLVM pointer arguments. It is generally used to pass
    structs and arrays by value, but is also valid on pointers to
    scalars. The copy is considered to belong to the caller not the
    callee (for example, `readonly` functions should not write to
    `byval` parameters). This is not a valid attribute for return
    values.

    The byval attribute also supports specifying an alignment with the
    align attribute. It indicates the alignment of the stack slot to
    form and the known alignment of the pointer specified to the call
    site. If the alignment is not specified, then the code generator
    makes a target-specific assumption.


`inalloca`


> The `inalloca` argument attribute allows the caller to take the
> address of outgoing stack arguments. An `inalloca` argument must be a
> pointer to stack memory produced by an `alloca` instruction. The
> alloca, or argument allocation, must also be tagged with the inalloca
> keyword. Only the last argument may have the `inalloca` attribute, and
> that argument is guaranteed to be passed in memory.
>
> An argument allocation may be used by a call at most once because the
> call may deallocate it. The `inalloca` attribute cannot be used in
> conjunction with other attributes that affect argument storage, like
> `inreg`, `nest`, `sret`, or `byval`. The `inalloca` attribute also
> disables LLVM\'s implicit lowering of large aggregate return values,
> which means that frontend authors must lower them with `sret`
> pointers.
>
> When the call site is reached, the argument allocation must have been
> the most recent stack allocation that is still live, or the behavior
> is undefined. It is possible to allocate additional stack space after
> an argument allocation and before its call site, but it must be
> cleared off with `llvm.stackrestore
> <int_stackrestore>`{.interpreted-text role="ref"}.
>
> See `InAlloca`{.interpreted-text role="doc"} for more information on
> how to use this attribute.

`sret`

This indicates that the pointer parameter specifies the address of a
    structure that is the return value of the function in the source
    program. This pointer must be guaranteed by the caller to be valid:
    loads and stores to the structure may be assumed by the callee not
    to trap and to be properly aligned. This is not a valid attribute
    for return values.



`align <n>`

This indicates that the pointer value may be assumed by the
    optimizer to have the specified alignment.

    Note that this attribute has additional semantics when combined with
    the `byval` attribute.




`noalias`

This indicates that objects accessed via pointer values
    `based <pointeraliasing>`{.interpreted-text role="ref"} on the
    argument or return value are not also accessed, during the execution
    of the function, via pointer values not *based* on the argument or
    return value. The attribute on a return value also has additional
    semantics described below. The caller shares the responsibility with
    the callee for ensuring that these requirements are met. For further
    details, please see the discussion of the NoAlias response in
    `alias analysis <Must, May, or No>`{.interpreted-text role="ref"}.

    Note that this definition of `noalias` is intentionally similar to
    the definition of `restrict` in C99 for function arguments.

    For function return values, C99\'s `restrict` is not meaningful,
    while LLVM\'s `noalias` is. Furthermore, the semantics of the
    `noalias` attribute on return values are stronger than the semantics
    of the attribute when used on function arguments. On function return
    values, the `noalias` attribute indicates that the function acts
    like a system memory allocation function, returning a pointer to
    allocated storage disjoint from the storage for any other object
    accessible to the caller.

`nocapture`

This indicates that the callee does not make any copies of the
    pointer that outlive the callee itself. This is not a valid
    attribute for return values. Addresses used in volatile operations
    are considered to be captured.




`nest`

This indicates that the pointer parameter can be excised using the
    `trampoline intrinsics <int_trampoline>`{.interpreted-text
    role="ref"}. This is not a valid attribute for return values and can
    only be applied to one parameter.

`returned`

This indicates that the function always returns the argument as its
    return value. This is a hint to the optimizer and code generator
    used when generating the caller, allowing value propagation, tail
    call optimization, and omission of register saves and restores in
    some cases; it is not checked or enforced when generating the
    callee. The parameter and the function return type must be valid
    operands for the `bitcast instruction <i_bitcast>`{.interpreted-text
    role="ref"}. This is not a valid attribute for return values and can
    only be applied to one parameter.

`nonnull`

This indicates that the parameter or return pointer is not null.
    This attribute may only be applied to pointer typed parameters. This
    is not checked or enforced by LLVM; if the parameter or return
    pointer is null, the behavior is undefined.

`dereferenceable(<n>)`

This indicates that the parameter or return pointer is
    dereferenceable. This attribute may only be applied to pointer typed
    parameters. A pointer that is dereferenceable can be loaded from
    speculatively without a risk of trapping. The number of bytes known
    to be dereferenceable must be provided in parentheses. It is legal
    for the number of bytes to be less than the size of the pointee
    type. The `nonnull` attribute does not imply dereferenceability
    (consider a pointer to one element past the end of an array),
    however `dereferenceable(<n>)` does imply `nonnull` in
    `addrspace(0)` (which is the default address space).

`dereferenceable_or_null(<n>)`

This indicates that the parameter or return value isn\'t both
    non-null and non-dereferenceable (up to `<n>` bytes) at the same
    time. All non-null pointers tagged with
    `dereferenceable_or_null(<n>)` are `dereferenceable(<n>)`. For
    address space 0 `dereferenceable_or_null(<n>)` implies that a
    pointer is exactly one of `dereferenceable(<n>)` or `null`, and in
    other address spaces `dereferenceable_or_null(<n>)` implies that a
    pointer is at least one of `dereferenceable(<n>)` or `null` (i.e. it
    may be both `null` and `dereferenceable(<n>)`). This attribute may
    only be applied to pointer typed parameters.

`swiftself`

This indicates that the parameter is the self/context parameter.
    This is not a valid attribute for return values and can only be
    applied to one parameter.

`swifterror`

This attribute is motivated to model and optimize Swift error
    handling. It can be applied to a parameter with pointer to pointer
    type or a pointer-sized alloca. At the call site, the actual
    argument that corresponds to a `swifterror` parameter has to come
    from a `swifterror` alloca or the `swifterror` parameter of the
    caller. A `swifterror` value (either the parameter or the alloca)
    can only be loaded and stored from, or used as a `swifterror`
    argument. This is not a valid attribute for return values and can
    only be applied to one parameter.

    These constraints allow the calling convention to optimize access to
    `swifterror` variables by associating them with a specific register
    at call boundaries rather than placing them in memory. Since this
    does change the calling convention, a function which uses the
    `swifterror` attribute on a parameter is not ABI-compatible with one
    which does not.

    These constraints also allow LLVM to assume that a `swifterror`
    argument does not alias any other memory visible within a function
    and that a `swifterror` alloca passed as an argument does not
    escape.


### Garbage Collector Strategy Names

Each function may specify a garbage collector strategy name, which is
simply a string:

``` {.llvm}
define void @f() gc "name" { ... }
```

The supported values of *name* includes those `built in to LLVM
<builtin-gc-strategies>`{.interpreted-text role="ref"} and any provided
by loaded plugins. Specifying a GC strategy will cause the compiler to
alter its output in order to support the named garbage collection
algorithm. Note that LLVM itself does not contain a garbage collector,
this functionality is restricted to generating machine code which can
interoperate with a collector provided externally.

### Prefix Data

Prefix data is data associated with a function which the code generator
will emit immediately before the function\'s entrypoint. The purpose of
this feature is to allow frontends to associate language-specific
runtime metadata with specific functions and make it available through
the function pointer while still allowing the function pointer to be
called.

To access the data for a given function, a program may bitcast the
function pointer to a pointer to the constant\'s type and dereference
index -1. This implies that the IR symbol points just past the end of
the prefix data. For instance, take the example of a function annotated
with a single `i32`,

``` {.llvm}
define void @f() prefix i32 123 { ... }
```

The prefix data can be referenced as,

``` {.llvm}
%0 = bitcast void* () @f to i32*
%a = getelementptr inbounds i32, i32* %0, i32 -1
%b = load i32, i32* %a
```

Prefix data is laid out as if it were an initializer for a global
variable of the prefix data\'s type. The function will be placed such
that the beginning of the prefix data is aligned. This means that if the
size of the prefix data is not a multiple of the alignment size, the
function\'s entrypoint will not be aligned. If alignment of the
function\'s entrypoint is desired, padding must be added to the prefix
data.

A function may have prefix data but no body. This has similar semantics
to the `available_externally` linkage in that the data may be used by
the optimizers but will not be emitted in the object file.

### Prologue Data

The `prologue` attribute allows arbitrary code (encoded as bytes) to be
inserted prior to the function body. This can be used for enabling
function hot-patching and instrumentation.

To maintain the semantics of ordinary function calls, the prologue data
must have a particular format. Specifically, it must begin with a
sequence of bytes which decode to a sequence of machine instructions,
valid for the module\'s target, which transfer control to the point
immediately succeeding the prologue data, without performing any other
visible action. This allows the inliner and other passes to reason about
the semantics of the function definition without needing to reason about
the prologue data. Obviously this makes the format of the prologue data
highly target dependent.

A trivial example of valid prologue data for the x86 architecture is
`i8 144`, which encodes the `nop` instruction:

``` {.text}
define void @f() prologue i8 144 { ... }
```

Generally prologue data can be formed by encoding a relative branch
instruction which skips the metadata, as in this example of valid
prologue data for the x86\_64 architecture, where the first two bytes
encode `jmp .+10`:

``` {.text}
%0 = type <{ i8, i8, i8* }>

define void @f() prologue %0 <{ i8 235, i8 8, i8* @md}> { ... }
```

A function may have prologue data but no body. This has similar
semantics to the `available_externally` linkage in that the data may be
used by the optimizers but will not be emitted in the object file.

### Personality Function

The `personality` attribute permits functions to specify what function
to use for exception handling.

### Attribute Groups

Attribute groups are groups of attributes that are referenced by objects
within the IR. They are important for keeping `.ll` files readable,
because a lot of functions will use the same set of attributes. In the
degenerative case of a `.ll` file that corresponds to a single `.c`
file, the single attribute group will capture the important command line
flags used to build that file.

An attribute group is a module-level object. To use an attribute group,
an object references the attribute group\'s ID (e.g. `#37`). An object
may refer to more than one attribute group. In that situation, the
attributes from the different groups are merged.

Here is an example of attribute groups for a function that should always
be inlined, has a stack alignment of 4, and which shouldn\'t use SSE
instructions:

``` {.llvm}
; Target-independent attributes:
attributes #0 = { alwaysinline alignstack=4 }

; Target-dependent attributes:
attributes #1 = { "no-sse" }

; Function @f has attributes: alwaysinline, alignstack=4, and "no-sse".
define void @f() #0 #1 { ... }
```

### Function Attributes

Function attributes are set to communicate additional information about
a function. Function attributes are considered to be part of the
function, not of the function type, so functions with different function
attributes can have the same function type.

Function attributes are simple keywords that follow the type specified.
If multiple attributes are needed, they are space separated. For
example:

``` {.llvm}
define void @f() noinline { ... }
define void @f() alwaysinline { ... }
define void @f() alwaysinline optsize { ... }
define void @f() optsize { ... }
```

`alignstack(<n>)`

This attribute indicates that, when emitting the prologue and
    epilogue, the backend should forcibly align the stack pointer.
    Specify the desired alignment, which must be a power of two, in
    parentheses.

`allocsize(<EltSizeParam>[, <NumEltsParam>])`

This attribute indicates that the annotated function will always
    return at least a given number of bytes (or null). Its arguments are
    zero-indexed parameter numbers; if one argument is provided, then
    it\'s assumed that at least `CallSite.Args[EltSizeParam]` bytes will
    be available at the returned pointer. If two are provided, then
    it\'s assumed that
    `CallSite.Args[EltSizeParam] * CallSite.Args[NumEltsParam]` bytes
    are available. The referenced parameters must be integer types. No
    assumptions are made about the contents of the returned block of
    memory.

`alwaysinline`

This attribute indicates that the inliner should attempt to inline
    this function into callers whenever possible, ignoring any active
    inlining size threshold for this caller.

`builtin`

This indicates that the callee function at a call site should be
    recognized as a built-in function, even though the function\'s
    declaration uses the `nobuiltin` attribute. This is only valid at
    call sites for direct calls to functions that are declared with the
    `nobuiltin` attribute.

`cold`

This attribute indicates that this function is rarely called. When
    computing edge weights, basic blocks post-dominated by a cold
    function call are also considered to be cold; and, thus, given low
    weight.

`convergent`

In some parallel execution models, there exist operations that
    cannot be made control-dependent on any additional values. We call
    such operations `convergent`, and mark them with this attribute.

    The `convergent` attribute may appear on functions or call/invoke
    instructions. When it appears on a function, it indicates that calls
    to this function should not be made control-dependent on additional
    values. For example, the intrinsic `llvm.nvvm.barrier0` is
    `convergent`, so calls to this intrinsic cannot be made
    control-dependent on additional values.

    When it appears on a call/invoke, the `convergent` attribute
    indicates that we should treat the call as though we\'re calling a
    convergent function. This is particularly useful on indirect calls;
    without this we may treat such calls as though the target is
    non-convergent.

    The optimizer may remove the `convergent` attribute on functions
    when it can prove that the function does not execute any convergent
    operations. Similarly, the optimizer may remove `convergent` on
    calls/invokes when it can prove that the call/invoke cannot call a
    convergent function.

`inaccessiblememonly`

This attribute indicates that the function may only access memory
    that is not accessible by the module being compiled. This is a
    weaker form of `readnone`. If the function reads or writes other
    memory, the behavior is undefined.

`inaccessiblemem_or_argmemonly`

This attribute indicates that the function may only access memory
    that is either not accessible by the module being compiled, or is
    pointed to by its pointer arguments. This is a weaker form of
    `argmemonly`. If the function reads or writes other memory, the
    behavior is undefined.

`inlinehint`

This attribute indicates that the source code contained a hint that
    inlining this function is desirable (such as the \"inline\" keyword
    in C/C++). It is just a hint; it imposes no requirements on the
    inliner.

`jumptable`

This attribute indicates that the function should be added to a
    jump-instruction table at code-generation time, and that all
    address-taken references to this function should be replaced with a
    reference to the appropriate jump-instruction-table function
    pointer. Note that this creates a new pointer for the original
    function, which means that code that depends on function-pointer
    identity can break. So, any function annotated with `jumptable` must
    also be `unnamed_addr`.

`minsize`

This attribute suggests that optimization passes and code generator
    passes make choices that keep the code size of this function as
    small as possible and perform optimizations that may sacrifice
    runtime performance in order to minimize the size of the generated
    code.

`naked`

This attribute disables prologue / epilogue emission for the
    function. This can have very system-specific consequences.

`no-jump-tables`

When this attribute is set to true, the jump tables and lookup
    tables that can be generated from a switch case lowering are
    disabled.

`nobuiltin`

This indicates that the callee function at a call site is not
    recognized as a built-in function. LLVM will retain the original
    call and not replace it with equivalent code based on the semantics
    of the built-in function, unless the call site uses the `builtin`
    attribute. This is valid at call sites and on function declarations
    and definitions.

`noduplicate`

This attribute indicates that calls to the function cannot be
    duplicated. A call to a `noduplicate` function may be moved within
    its parent function, but may not be duplicated within its parent
    function.

    A function containing a `noduplicate` call may still be an inlining
    candidate, provided that the call is not duplicated by inlining.
    That implies that the function has internal linkage and only has one
    call site, so the original call is dead after inlining.

`noimplicitfloat`

This attributes disables implicit floating-point instructions.

`noinline`

This attribute indicates that the inliner should never inline this
    function in any situation. This attribute may not be used together
    with the `alwaysinline` attribute.

`nonlazybind`

This attribute suppresses lazy symbol binding for the function. This
    may make calls to the function faster, at the cost of extra program
    startup time if the function is not called during program startup.

`noredzone`

This attribute indicates that the code generator should not use a
    red zone, even if the target-specific ABI normally permits it.

`indirect-tls-seg-refs`

This attribute indicates that the code generator should not use
    direct TLS access through segment registers, even if the
    target-specific ABI normally permits it.

`noreturn`

This function attribute indicates that the function never returns
    normally. This produces undefined behavior at runtime if the
    function ever does dynamically return.

`norecurse`

This function attribute indicates that the function does not call
    itself either directly or indirectly down any possible call path.
    This produces undefined behavior at runtime if the function ever
    does recurse.

`nounwind`

This function attribute indicates that the function never raises an
    exception. If the function does raise an exception, its runtime
    behavior is undefined. However, functions marked nounwind may still
    trap or generate asynchronous exceptions. Exception handling schemes
    that are recognized by LLVM to handle asynchronous exceptions, such
    as SEH, will still provide their implementation defined semantics.

`"null-pointer-is-valid"`

If `"null-pointer-is-valid"` is set to `"true"`, then `null` address
    in address-space 0 is considered to be a valid address for memory
    loads and stores. Any analysis or optimization should not treat
    dereferencing a pointer to `null` as undefined behavior in this
    function. Note: Comparing address of a global variable to `null` may
    still evaluate to false because of a limitation in querying this
    attribute inside constant expressions.

`optforfuzzing`

This attribute indicates that this function should be optimized for
    maximum fuzzing signal.

`optnone`

This function attribute indicates that most optimization passes will
    skip this function, with the exception of interprocedural
    optimization passes. Code generation defaults to the \"fast\"
    instruction selector. This attribute cannot be used together with
    the `alwaysinline` attribute; this attribute is also incompatible
    with the `minsize` attribute and the `optsize` attribute.

    This attribute requires the `noinline` attribute to be specified on
    the function as well, so the function is never inlined into any
    caller. Only functions with the `alwaysinline` attribute are valid
    candidates for inlining into the body of this function.

`optsize`

This attribute suggests that optimization passes and code generator
    passes make choices that keep the code size of this function low,
    and otherwise do optimizations specifically to reduce code size as
    long as they do not significantly impact runtime performance.

`"patchable-function"`

This attribute tells the code generator that the code generated for
    this function needs to follow certain conventions that make it
    possible for a runtime function to patch over it later. The exact
    effect of this attribute depends on its string value, for which
    there currently is one legal possibility:

    > -   `"prologue-short-redirect"` - This style of patchable function
    >     is intended to support patching a function prologue to
    >     redirect control away from the function in a thread safe
    >     manner. It guarantees that the first instruction of the
    >     function will be large enough to accommodate a short jump
    >     instruction, and will be sufficiently aligned to allow being
    >     fully changed via an atomic compare-and-swap instruction.
    >     While the first requirement can be satisfied by inserting
    >     large enough NOP, LLVM can and will try to re-purpose an
    >     existing instruction (i.e. one that would have to be emitted
    >     anyway) as the patchable instruction larger than a short jump.
    >
    >     `"prologue-short-redirect"` is currently only supported on
    >     x86-64.
    >
    This attribute by itself does not imply restrictions on
    inter-procedural optimizations. All of the semantic effects the
    patching may have to be separately conveyed via the linkage type.

`"probe-stack"`

This attribute indicates that the function will trigger a guard
    region in the end of the stack. It ensures that accesses to the
    stack must be no further apart than the size of the guard region to
    a previous access of the stack. It takes one required string value,
    the name of the stack probing function that will be called.

    If a function that has a `"probe-stack"` attribute is inlined into a
    function with another `"probe-stack"` attribute, the resulting
    function has the `"probe-stack"` attribute of the caller. If a
    function that has a `"probe-stack"` attribute is inlined into a
    function that has no `"probe-stack"` attribute at all, the resulting
    function has the `"probe-stack"` attribute of the callee.

`readnone`

On a function, this attribute indicates that the function computes
    its result (or decides to unwind an exception) based strictly on its
    arguments, without dereferencing any pointer arguments or otherwise
    accessing any mutable state (e.g. memory, control registers, etc)
    visible to caller functions. It does not write through any pointer
    arguments (including `byval` arguments) and never changes any state
    visible to callers. This means while it cannot unwind exceptions by
    calling the `C++` exception throwing methods (since they write to
    memory), there may be non-`C++` mechanisms that throw exceptions
    without writing to LLVM visible memory.

    On an argument, this attribute indicates that the function does not
    dereference that pointer argument, even though it may read or write
    the memory that the pointer points to if accessed through other
    pointers.

    If a readnone function reads or writes memory visible to the
    program, or has other side-effects, the behavior is undefined. If a
    function reads from or writes to a readnone pointer argument, the
    behavior is undefined.

`readonly`

On a function, this attribute indicates that the function does not
    write through any pointer arguments (including `byval` arguments) or
    otherwise modify any state (e.g. memory, control registers, etc)
    visible to caller functions. It may dereference pointer arguments
    and read state that may be set in the caller. A readonly function
    always returns the same value (or unwinds an exception identically)
    when called with the same set of arguments and global state. This
    means while it cannot unwind exceptions by calling the `C++`
    exception throwing methods (since they write to memory), there may
    be non-`C++` mechanisms that throw exceptions without writing to
    LLVM visible memory.

    On an argument, this attribute indicates that the function does not
    write through this pointer argument, even though it may write to the
    memory that the pointer points to.

    If a readonly function writes memory visible to the program, or has
    other side-effects, the behavior is undefined. If a function writes
    to a readonly pointer argument, the behavior is undefined.

`"stack-probe-size"`

This attribute controls the behavior of stack probes: either the
    `"probe-stack"` attribute, or ABI-required stack probes, if any. It
    defines the size of the guard region. It ensures that if the
    function may use more stack space than the size of the guard region,
    stack probing sequence will be emitted. It takes one required
    integer value, which is 4096 by default.

    If a function that has a `"stack-probe-size"` attribute is inlined
    into a function with another `"stack-probe-size"` attribute, the
    resulting function has the `"stack-probe-size"` attribute that has
    the lower numeric value. If a function that has a
    `"stack-probe-size"` attribute is inlined into a function that has
    no `"stack-probe-size"` attribute at all, the resulting function has
    the `"stack-probe-size"` attribute of the callee.

`"no-stack-arg-probe"`

This attribute disables ABI-required stack probes, if any.

`writeonly`

On a function, this attribute indicates that the function may write
    to but does not read from memory.

    On an argument, this attribute indicates that the function may write
    to but does not read through this pointer argument (even though it
    may read from the memory that the pointer points to).

    If a writeonly function reads memory visible to the program, or has
    other side-effects, the behavior is undefined. If a function reads
    from a writeonly pointer argument, the behavior is undefined.

`argmemonly`

This attribute indicates that the only memory accesses inside
    function are loads and stores from objects pointed to by its
    pointer-typed arguments, with arbitrary offsets. Or in other words,
    all memory operations in the function can refer to memory only using
    pointers based on its function arguments.

    Note that `argmemonly` can be used together with `readonly`
    attribute in order to specify that function reads only from its
    arguments.

    If an argmemonly function reads or writes memory other than the
    pointer arguments, or has other side-effects, the behavior is
    undefined.

`returns_twice`

This attribute indicates that this function can return twice. The C
    `setjmp` is an example of such a function. The compiler disables
    some optimizations (like tail calls) in the caller of these
    functions.

`safestack`

This attribute indicates that
    [SafeStack](http://clang.llvm.org/docs/SafeStack.html) protection is
    enabled for this function.

    If a function that has a `safestack` attribute is inlined into a
    function that doesn\'t have a `safestack` attribute or which has an
    `ssp`, `sspstrong` or `sspreq` attribute, then the resulting
    function will have a `safestack` attribute.

`sanitize_address`

This attribute indicates that AddressSanitizer checks (dynamic
    address safety analysis) are enabled for this function.

`sanitize_memory`

This attribute indicates that MemorySanitizer checks (dynamic
    detection of accesses to uninitialized memory) are enabled for this
    function.

`sanitize_thread`

This attribute indicates that ThreadSanitizer checks (dynamic thread
    safety analysis) are enabled for this function.

`sanitize_hwaddress`

This attribute indicates that HWAddressSanitizer checks (dynamic
    address safety analysis based on tagged pointers) are enabled for
    this function.

`speculative_load_hardening`

This attribute indicates that [Speculative Load
    Hardening](https://llvm.org/docs/SpeculativeLoadHardening.html)
    should be enabled for the function body.

    Speculative Load Hardening is a best-effort mitigation against
    information leak attacks that make use of control flow
    miss-speculation - specifically miss-speculation of whether a branch
    is taken or not. Typically vulnerabilities enabling such attacks are
    classified as \"Spectre variant \#1\". Notably, this does not
    attempt to mitigate against miss-speculation of branch target,
    classified as \"Spectre variant \#2\" vulnerabilities.

    When inlining, the attribute is sticky. Inlining a function that
    carries this attribute will cause the caller to gain the attribute.
    This is intended to provide a maximally conservative model where the
    code in a function annotated with this attribute will always (even
    after inlining) end up hardened.

`speculatable`

This function attribute indicates that the function does not have
    any effects besides calculating its result and does not have
    undefined behavior. Note that `speculatable` is not enough to
    conclude that along any particular execution path the number of
    calls to this function will not be externally observable. This
    attribute is only valid on functions and declarations, not on
    individual call sites. If a function is incorrectly marked as
    speculatable and really does exhibit undefined behavior, the
    undefined behavior may be observed even if the call site is dead
    code.

`ssp`

This attribute indicates that the function should emit a stack
    smashing protector. It is in the form of a \"canary\" \-\-- a random
    value placed on the stack before the local variables that\'s checked
    upon return from the function to see if it has been overwritten. A
    heuristic is used to determine if a function needs stack protectors
    or not. The heuristic used will enable protectors for functions
    with:

    -   Character arrays larger than `ssp-buffer-size` (default 8).
    -   Aggregates containing character arrays larger than
        `ssp-buffer-size`.
    -   Calls to alloca() with variable sizes or constant sizes greater
        than `ssp-buffer-size`.

    Variables that are identified as requiring a protector will be
    arranged on the stack such that they are adjacent to the stack
    protector guard.

    If a function that has an `ssp` attribute is inlined into a function
    that doesn\'t have an `ssp` attribute, then the resulting function
    will have an `ssp` attribute.

`sspreq`

This attribute indicates that the function should *always* emit a
    stack smashing protector. This overrides the `ssp` function
    attribute.

    Variables that are identified as requiring a protector will be
    arranged on the stack such that they are adjacent to the stack
    protector guard. The specific layout rules are:

    1.  Large arrays and structures containing large arrays
        (`>= ssp-buffer-size`) are closest to the stack protector.
    2.  Small arrays and structures containing small arrays
        (`< ssp-buffer-size`) are 2nd closest to the protector.
    3.  Variables that have had their address taken are 3rd closest to
        the protector.

    If a function that has an `sspreq` attribute is inlined into a
    function that doesn\'t have an `sspreq` attribute or which has an
    `ssp` or `sspstrong` attribute, then the resulting function will
    have an `sspreq` attribute.

`sspstrong`

This attribute indicates that the function should emit a stack
    smashing protector. This attribute causes a strong heuristic to be
    used when determining if a function needs stack protectors. The
    strong heuristic will enable protectors for functions with:

    -   Arrays of any size and type
    -   Aggregates containing an array of any size and type.
    -   Calls to alloca().
    -   Local variables that have had their address taken.

    Variables that are identified as requiring a protector will be
    arranged on the stack such that they are adjacent to the stack
    protector guard. The specific layout rules are:

    1.  Large arrays and structures containing large arrays
        (`>= ssp-buffer-size`) are closest to the stack protector.
    2.  Small arrays and structures containing small arrays
        (`< ssp-buffer-size`) are 2nd closest to the protector.
    3.  Variables that have had their address taken are 3rd closest to
        the protector.

    This overrides the `ssp` function attribute.

    If a function that has an `sspstrong` attribute is inlined into a
    function that doesn\'t have an `sspstrong` attribute, then the
    resulting function will have an `sspstrong` attribute.

`strictfp`

This attribute indicates that the function was called from a scope
    that requires strict floating-point semantics. LLVM will not attempt
    any optimizations that require assumptions about the floating-point
    rounding mode or that might alter the state of floating-point status
    flags that might otherwise be set or cleared by calling this
    function.

`"thunk"`

This attribute indicates that the function will delegate to some
    other function with a tail call. The prototype of a thunk should not
    be used for optimization purposes. The caller is expected to cast
    the thunk prototype to match the thunk target prototype.

`uwtable`

This attribute indicates that the ABI being targeted requires that
    an unwind table entry be produced for this function even if we can
    show that no exceptions passes by it. This is normally the case for
    the ELF x86-64 abi, but it can be disabled for some compilation
    units.

`nocf_check`

This attribute indicates that no control-flow check will be
    performed on the attributed entity. It disables -fcf-protection=\<\>
    for a specific entity to fine grain the HW control flow protection
    mechanism. The flag is target independent and currently appertains
    to a function or function pointer.

`shadowcallstack`

This attribute indicates that the ShadowCallStack checks are enabled
    for the function. The instrumentation checks that the return address
    for the function has not changed between the function prolog and
    eiplog. It is currently x86\_64-specific.

### Global Attributes

Attributes may be set to communicate additional information about a
global variable. Unlike
`function attributes <fnattrs>`{.interpreted-text role="ref"},
attributes on a global variable are grouped into a single
`attribute group <attrgrp>`{.interpreted-text role="ref"}.

### Operand Bundles

Operand bundles are tagged sets of SSA values that can be associated
with certain LLVM instructions (currently only `call` s and `invoke` s).
In a way they are like metadata, but dropping them is incorrect and will
change program semantics.

Syntax:

    operand bundle set ::= '[' operand bundle (, operand bundle )* ']'
    operand bundle ::= tag '(' [ bundle operand ] (, bundle operand )* ')'
    bundle operand ::= SSA value
    tag ::= string constant

Operand bundles are **not** part of a function\'s signature, and a given
function may be called from multiple places with different kinds of
operand bundles. This reflects the fact that the operand bundles are
conceptually a part of the `call` (or `invoke`), not the callee being
dispatched to.

Operand bundles are a generic mechanism intended to support
runtime-introspection-like functionality for managed languages. While
the exact semantics of an operand bundle depend on the bundle tag, there
are certain limitations to how much the presence of an operand bundle
can influence the semantics of a program. These restrictions are
described as the semantics of an \"unknown\" operand bundle. As long as
the behavior of an operand bundle is describable within these
restrictions, LLVM does not need to have special knowledge of the
operand bundle to not miscompile programs containing it.

-   The bundle operands for an unknown operand bundle escape in unknown
    ways before control is transferred to the callee or invokee.
-   Calls and invokes with operand bundles have unknown read / write
    effect on the heap on entry and exit (even if the call target is
    `readnone` or `readonly`), unless they\'re overridden with callsite
    specific attributes.
-   An operand bundle at a call site cannot change the implementation of
    the called function. Inter-procedural optimizations work as usual as
    long as they take into account the first two properties.

More specific types of operand bundles are described below.

#### Deoptimization Operand Bundles

Deoptimization operand bundles are characterized by the `"deopt"`
operand bundle tag. These operand bundles represent an alternate
\"safe\" continuation for the call site they\'re attached to, and can be
used by a suitable runtime to deoptimize the compiled frame at the
specified call site. There can be at most one `"deopt"` operand bundle
attached to a call site. Exact details of deoptimization is out of scope
for the language reference, but it usually involves rewriting a compiled
frame into a set of interpreted frames.

From the compiler\'s perspective, deoptimization operand bundles make
the call sites they\'re attached to at least `readonly`. They read
through all of their pointer typed operands (even if they\'re not
otherwise escaped) and the entire visible heap. Deoptimization operand
bundles do not capture their operands except during deoptimization, in
which case control will not be returned to the compiled frame.

The inliner knows how to inline through calls that have deoptimization
operand bundles. Just like inlining through a normal call site involves
composing the normal and exceptional continuations, inlining through a
call site with a deoptimization operand bundle needs to appropriately
compose the \"safe\" deoptimization continuation. The inliner does this
by prepending the parent\'s deoptimization continuation to every
deoptimization continuation in the inlined body. E.g. inlining `@f` into
`@g` in the following example

``` {.llvm}
define void @f() {
  call void @x()  ;; no deopt state
  call void @y() [ "deopt"(i32 10) ]
  call void @y() [ "deopt"(i32 10), "unknown"(i8* null) ]
  ret void
}

define void @g() {
  call void @f() [ "deopt"(i32 20) ]
  ret void
}
```

will result in

``` {.llvm}
define void @g() {
  call void @x()  ;; still no deopt state
  call void @y() [ "deopt"(i32 20, i32 10) ]
  call void @y() [ "deopt"(i32 20, i32 10), "unknown"(i8* null) ]
  ret void
}
```

It is the frontend\'s responsibility to structure or encode the
deoptimization state in a way that syntactically prepending the
caller\'s deoptimization state to the callee\'s deoptimization state is
semantically equivalent to composing the caller\'s deoptimization
continuation after the callee\'s deoptimization continuation.

#### Funclet Operand Bundles

Funclet operand bundles are characterized by the `"funclet"` operand
bundle tag. These operand bundles indicate that a call site is within a
particular funclet. There can be at most one `"funclet"` operand bundle
attached to a call site and it must have exactly one bundle operand.

If any funclet EH pads have been \"entered\" but not \"exited\" (per the
[description in the EH doc](ExceptionHandling.html#wineh-constraints)),
it is undefined behavior to execute a `call` or `invoke` which:

-   does not have a `"funclet"` bundle and is not a `call` to a nounwind
    intrinsic, or
-   has a `"funclet"` bundle whose operand is not the
    most-recently-entered not-yet-exited funclet EH pad.

Similarly, if no funclet EH pads have been entered-but-not-yet-exited,
executing a `call` or `invoke` with a `"funclet"` bundle is undefined
behavior.

#### GC Transition Operand Bundles

GC transition operand bundles are characterized by the `"gc-transition"`
operand bundle tag. These operand bundles mark a call as a transition
between a function with one GC strategy to a function with a different
GC strategy. If coordinating the transition between GC strategies
requires additional code generation at the call site, these bundles may
contain any values that are needed by the generated code. For more
details, see `GC Transitions
<gc_transition_args>`{.interpreted-text role="ref"}.

### Module-Level Inline Assembly

Modules may contain \"module-level inline asm\" blocks, which
corresponds to the GCC \"file scope inline asm\" blocks. These blocks
are internally concatenated by LLVM and treated as a single unit, but
may be separated in the `.ll` file if desired. The syntax is very
simple:

``` {.llvm}
module asm "inline asm code goes here"
module asm "more can go here"
```

The strings can contain any character by escaping non-printable
characters. The escape sequence used is simply \"\\xx\" where \"xx\" is
the two digit hex code for the number.

Note that the assembly string *must* be parseable by LLVM\'s integrated
assembler (unless it is disabled), even when emitting a `.s` file.

### Data Layout

A module may specify a target specific data layout string that specifies
how data is to be laid out in memory. The syntax for the data layout is
simply:

``` {.llvm}
target datalayout = "layout specification"
```

The *layout specification* consists of a list of specifications
separated by the minus sign character (\'-\'). Each specification starts
with a letter and may include other information after the letter to
define some aspect of the data layout. The specifications accepted are
as follows:

`E`

Specifies that the target lays out data in big-endian form. That is,
    the bits with the most significance have the lowest address
    location.

`e`

Specifies that the target lays out data in little-endian form. That
    is, the bits with the least significance have the lowest address
    location.

`S<size>`

Specifies the natural alignment of the stack in bits. Alignment
    promotion of stack variables is limited to the natural stack
    alignment to avoid dynamic stack realignment. The stack alignment
    must be a multiple of 8-bits. If omitted, the natural stack
    alignment defaults to \"unspecified\", which does not prevent any
    alignment promotions.

`P<address space>`

Specifies the address space that corresponds to program memory.
    Harvard architectures can use this to specify what space LLVM should
    place things such as functions into. If omitted, the program memory
    space defaults to the default address space of 0, which corresponds
    to a Von Neumann architecture that has code and data in the same
    space.

`A<address space>`

Specifies the address space of objects created by \'`alloca`\'.
    Defaults to the default address space of 0.

`p[n]:<size>:<abi>:<pref>:<idx>`

This specifies the *size* of a pointer and its `<abi>` and
    `<pref>`erred alignments for address space `n`. The fourth parameter
    `<idx>` is a size of index that used for address calculation. If not
    specified, the default index size is equal to the pointer size. All
    sizes are in bits. The address space, `n`, is optional, and if not
    specified, denotes the default address space 0. The value of `n`
    must be in the range \[1,2\^23).

`i<size>:<abi>:<pref>`

This specifies the alignment for an integer type of a given bit
    `<size>`. The value of `<size>` must be in the range \[1,2\^23).

`v<size>:<abi>:<pref>`

This specifies the alignment for a vector type of a given bit
    `<size>`.

`f<size>:<abi>:<pref>`

This specifies the alignment for a floating-point type of a given
    bit `<size>`. Only values of `<size>` that are supported by the
    target will work. 32 (float) and 64 (double) are supported on all
    targets; 80 or 128 (different flavors of long double) are also
    supported on some targets.

`a:<abi>:<pref>`

This specifies the alignment for an object of aggregate type.

`m:<mangling>`

If present, specifies that llvm names are mangled in the output.
    Symbols prefixed with the mangling escape character `\01` are passed
    through directly to the assembler without the escape character. The
    mangling style options are

    -   `e`: ELF mangling: Private symbols get a `.L` prefix.
    -   `m`: Mips mangling: Private symbols get a `$` prefix.
    -   `o`: Mach-O mangling: Private symbols get `L` prefix. Other
        symbols get a `_` prefix.
    -   `x`: Windows x86 COFF mangling: Private symbols get the usual
        prefix. Regular C symbols get a `_` prefix. Functions with
        `__stdcall`, `__fastcall`, and `__vectorcall` have custom
        mangling that appends `@N` where N is the number of bytes used
        to pass parameters. C++ symbols starting with `?` are not
        mangled in any way.
    -   `w`: Windows COFF mangling: Similar to `x`, except that normal C
        symbols do not receive a `_` prefix.

`n<size1>:<size2>:<size3>...`

This specifies a set of native integer widths for the target CPU in
    bits. For example, it might contain `n32` for 32-bit PowerPC,
    `n32:64` for PowerPC 64, or `n8:16:32:64` for X86-64. Elements of
    this set are considered to support most general arithmetic
    operations efficiently.

`ni:<address space0>:<address space1>:<address space2>...`

This specifies pointer types with the specified address spaces as
    `Non-Integral Pointer Type <nointptrtype>`{.interpreted-text
    role="ref"} s. The `0` address space cannot be specified as
    non-integral.

On every specification that takes a `<abi>:<pref>`, specifying the
`<pref>` alignment is optional. If omitted, the preceding `:` should be
omitted too and `<pref>` will be equal to `<abi>`.

When constructing the data layout for a given target, LLVM starts with a
default set of specifications which are then (possibly) overridden by
the specifications in the `datalayout` keyword. The default
specifications are given in this list:

-   `E` - big endian
-   `p:64:64:64` - 64-bit pointers with 64-bit alignment.
-   `p[n]:64:64:64` - Other address spaces are assumed to be the same as
    the default address space.
-   `S0` - natural stack alignment is unspecified
-   `i1:8:8` - i1 is 8-bit (byte) aligned
-   `i8:8:8` - i8 is 8-bit (byte) aligned
-   `i16:16:16` - i16 is 16-bit aligned
-   `i32:32:32` - i32 is 32-bit aligned
-   `i64:32:64` - i64 has ABI alignment of 32-bits but preferred
    alignment of 64-bits
-   `f16:16:16` - half is 16-bit aligned
-   `f32:32:32` - float is 32-bit aligned
-   `f64:64:64` - double is 64-bit aligned
-   `f128:128:128` - quad is 128-bit aligned
-   `v64:64:64` - 64-bit vector is 64-bit aligned
-   `v128:128:128` - 128-bit vector is 128-bit aligned
-   `a:0:64` - aggregates are 64-bit aligned

When LLVM is determining the alignment for a given type, it uses the
following rules:

1.  If the type sought is an exact match for one of the specifications,
    that specification is used.
2.  If no match is found, and the type sought is an integer type, then
    the smallest integer type that is larger than the bitwidth of the
    sought type is used. If none of the specifications are larger than
    the bitwidth then the largest integer type is used. For example,
    given the default specifications above, the i7 type will use the
    alignment of i8 (next largest) while both i65 and i256 will use the
    alignment of i64 (largest specified).
3.  If no match is found, and the type sought is a vector type, then the
    largest vector type that is smaller than the sought vector type will
    be used as a fall back. This happens because \<128 x double\> can be
    implemented in terms of 64 \<2 x double\>, for example.

The function of the data layout string may not be what you expect.
Notably, this is not a specification from the frontend of what alignment
the code generator should use.

Instead, if specified, the target data layout is required to match what
the ultimate *code generator* expects. This string is used by the
mid-level optimizers to improve code, and this only works if it matches
what the ultimate code generator uses. There is no way to generate IR
that does not embed this target-specific detail into the IR. If you
don\'t specify the string, the default specifications will be used to
generate a Data Layout and the optimization phases will operate
accordingly and introduce target specificity into the IR with respect to
these default specifications.

### Target Triple

A module may specify a target triple string that describes the target
host. The syntax for the target triple is simply:

``` {.llvm}
target triple = "x86_64-apple-macosx10.7.0"
```

The *target triple* string consists of a series of identifiers delimited
by the minus sign character (\'-\'). The canonical forms are:

    ARCHITECTURE-VENDOR-OPERATING_SYSTEM
    ARCHITECTURE-VENDOR-OPERATING_SYSTEM-ENVIRONMENT

This information is passed along to the backend so that it generates
code for the proper architecture. It\'s possible to override this on the
command line with the `-mtriple` command line option.

### Pointer Aliasing Rules

Any memory access must be done through a pointer value associated with
an address range of the memory access, otherwise the behavior is
undefined. Pointer values are associated with address ranges according
to the following rules:

-   A pointer value is associated with the addresses associated with any
    value it is *based* on.
-   An address of a global variable is associated with the address range
    of the variable\'s storage.
-   The result value of an allocation instruction is associated with the
    address range of the allocated storage.
-   A null pointer in the default address-space is associated with no
    address.
-   An integer constant other than zero or a pointer value returned from
    a function not defined within LLVM may be associated with address
    ranges allocated through mechanisms other than those provided by
    LLVM. Such ranges shall not overlap with any ranges of addresses
    allocated by mechanisms provided by LLVM.

A pointer value is *based* on another pointer value according to the
following rules:

-   A pointer value formed from a scalar `getelementptr` operation is
    *based* on the pointer-typed operand of the `getelementptr`.
-   The pointer in lane *l* of the result of a vector `getelementptr`
    operation is *based* on the pointer in lane *l* of the
    vector-of-pointers-typed operand of the `getelementptr`.
-   The result value of a `bitcast` is *based* on the operand of the
    `bitcast`.
-   A pointer value formed by an `inttoptr` is *based* on all pointer
    values that contribute (directly or indirectly) to the computation
    of the pointer\'s value.
-   The \"*based* on\" relationship is transitive.

Note that this definition of *\"based\"* is intentionally similar to the
definition of *\"based\"* in C99, though it is slightly weaker.

LLVM IR does not associate types with memory. The result type of a
`load` merely indicates the size and alignment of the memory from which
to load, as well as the interpretation of the value. The first operand
type of a `store` similarly only indicates the size and alignment of the
store.

Consequently, type-based alias analysis, aka TBAA, aka
`-fstrict-aliasing`, is not applicable to general unadorned LLVM IR.
`Metadata <metadata>`{.interpreted-text role="ref"} may be used to
encode additional information which specialized optimization passes may
use to implement type-based alias analysis.

### Volatile Memory Accesses

Certain memory accesses, such as `load <i_load>`{.interpreted-text
role="ref"}\'s, `store <i_store>`{.interpreted-text role="ref"}\'s, and
`llvm.memcpy <int_memcpy>`{.interpreted-text role="ref"}\'s may be
marked `volatile`. The optimizers must not change the number of volatile
operations or change their order of execution relative to other volatile
operations. The optimizers *may* change the order of volatile operations
relative to non-volatile operations. This is not Java\'s \"volatile\"
and has no cross-thread synchronization behavior.

IR-level volatile loads and stores cannot safely be optimized into
llvm.memcpy or llvm.memmove intrinsics even when those intrinsics are
flagged volatile. Likewise, the backend should never split or merge
target-legal volatile load/store instructions.

{.admonition}
Rationale

Platforms may rely on volatile loads and stores of natively supported
data width to be executed as single instruction. For example, in C this
holds for an l-value of volatile primitive type with native hardware
support, but not necessarily for aggregate types. The frontend upholds
these expectations, which are intentionally unspecified in the IR. The
rules above ensure that IR transformations do not violate the
frontend\'s contract with the language.


### Memory Model for Concurrent Operations

The LLVM IR does not define any way to start parallel threads of
execution or to register signal handlers. Nonetheless, there are
platform-specific ways to create them, and we define LLVM IR\'s behavior
in their presence. This model is inspired by the C++0x memory model.

For a more informal introduction to this model, see the
`Atomics`{.interpreted-text role="doc"}.

We define a *happens-before* partial order as the least partial order
that

-   Is a superset of single-thread program order, and
-   When a *synchronizes-with* `b`, includes an edge from `a` to `b`.
    *Synchronizes-with* pairs are introduced by platform-specific
    techniques, like pthread locks, thread creation, thread joining,
    etc., and by atomic instructions. (See also `Atomic Memory Ordering
    Constraints <ordering>`{.interpreted-text role="ref"}).

Note that program order does not introduce *happens-before* edges
between a thread and signals executing inside that thread.

Every (defined) read operation (load instructions, memcpy, atomic
loads/read-modify-writes, etc.) R reads a series of bytes written by
(defined) write operations (store instructions, atomic
stores/read-modify-writes, memcpy, etc.). For the purposes of this
section, initialized globals are considered to have a write of the
initializer which is atomic and happens before any other read or write
of the memory in question. For each byte of a read R, R~byte~ may see
any write to the same byte, except:

-   If write~1~ happens before write~2~, and write~2~ happens before
    R~byte~, then R~byte~ does not see write~1~.
-   If R~byte~ happens before write~3~, then R~byte~ does not see
    write~3~.

Given that definition, R~byte~ is defined as follows:

-   If R is volatile, the result is target-dependent. (Volatile is
    supposed to give guarantees which can support `sig_atomic_t` in
    C/C++, and may be used for accesses to addresses that do not behave
    like normal memory. It does not generally provide cross-thread
    synchronization.)
-   Otherwise, if there is no write to the same byte that happens before
    R~byte~, R~byte~ returns `undef` for that byte.
-   Otherwise, if R~byte~ may see exactly one write, R~byte~ returns the
    value written by that write.
-   Otherwise, if R is atomic, and all the writes R~byte~ may see are
    atomic, it chooses one of the values written. See the `Atomic
    Memory Ordering Constraints <ordering>`{.interpreted-text
    role="ref"} section for additional constraints on how the choice is
    made.
-   Otherwise R~byte~ returns `undef`.

R returns the value composed of the series of bytes it read. This
implies that some bytes within the value may be `undef` **without** the
entire value being `undef`. Note that this only defines the semantics of
the operation; it doesn\'t mean that targets will emit more than one
instruction to read the series of bytes.

Note that in cases where none of the atomic intrinsics are used, this
model places only one restriction on IR transformations on top of what
is required for single-threaded execution: introducing a store to a byte
which might not otherwise be stored is not allowed in general.
(Specifically, in the case where another thread might write to and read
from an address, introducing a store can change a load that may see
exactly one write into a load that may see multiple writes.)

### Atomic Memory Ordering Constraints

Atomic instructions (`cmpxchg <i_cmpxchg>`{.interpreted-text
role="ref"}, `atomicrmw <i_atomicrmw>`{.interpreted-text role="ref"},
`fence <i_fence>`{.interpreted-text role="ref"},
`atomic load <i_load>`{.interpreted-text role="ref"}, and
`atomic store <i_store>`{.interpreted-text role="ref"}) take ordering
parameters that determine which other atomic instructions on the same
address they *synchronize with*. These semantics are borrowed from Java
and C++0x, but are somewhat more colloquial. If these descriptions
aren\'t precise enough, check those specs (see spec references in the
`atomics guide <Atomics>`{.interpreted-text role="doc"}).
`fence <i_fence>`{.interpreted-text role="ref"} instructions treat these
orderings somewhat differently since they don\'t take an address. See
that instruction\'s documentation for details.

For a simpler introduction to the ordering constraints, see the
`Atomics`{.interpreted-text role="doc"}.

`unordered`

The set of values that can be read is governed by the happens-before
    partial order. A value cannot be read unless some operation wrote
    it. This is intended to provide a guarantee strong enough to model
    Java\'s non-volatile shared variables. This ordering cannot be
    specified for read-modify-write operations; it is not strong enough
    to make them atomic in any interesting way.

`monotonic`

In addition to the guarantees of `unordered`, there is a single
    total order for modifications by `monotonic` operations on each
    address. All modification orders must be compatible with the
    happens-before order. There is no guarantee that the modification
    orders can be combined to a global total order for the whole program
    (and this often will not be possible). The read in an atomic
    read-modify-write operation (`cmpxchg <i_cmpxchg>`{.interpreted-text
    role="ref"} and `atomicrmw <i_atomicrmw>`{.interpreted-text
    role="ref"}) reads the value in the modification order immediately
    before the value it writes. If one atomic read happens before
    another atomic read of the same address, the later read must see the
    same value or a later value in the address\'s modification order.
    This disallows reordering of `monotonic` (or stronger) operations on
    the same address. If an address is written `monotonic`-ally by one
    thread, and other threads `monotonic`-ally read that address
    repeatedly, the other threads must eventually see the write. This
    corresponds to the C++0x/C1x `memory_order_relaxed`.

`acquire`

In addition to the guarantees of `monotonic`, a *synchronizes-with*
    edge may be formed with a `release` operation. This is intended to
    model C++\'s `memory_order_acquire`.

`release`

In addition to the guarantees of `monotonic`, if this operation
    writes a value which is subsequently read by an `acquire` operation,
    it *synchronizes-with* that operation. (This isn\'t a complete
    description; see the C++0x definition of a release sequence.) This
    corresponds to the C++0x/C1x `memory_order_release`.

`acq_rel` (acquire+release)

Acts as both an `acquire` and `release` operation on its address.
    This corresponds to the C++0x/C1x `memory_order_acq_rel`.

`seq_cst` (sequentially consistent)

In addition to the guarantees of `acq_rel` (`acquire` for an
    operation that only reads, `release` for an operation that only
    writes), there is a global total order on all
    sequentially-consistent operations on all addresses, which is
    consistent with the *happens-before* partial order and with the
    modification orders of all the affected addresses. Each
    sequentially-consistent read sees the last preceding write to the
    same address in this global order. This corresponds to the C++0x/C1x
    `memory_order_seq_cst` and Java volatile.


If an atomic operation is marked `syncscope("singlethread")`, it only
*synchronizes with* and only participates in the seq\_cst total
orderings of other operations running in the same thread (for example,
in signal handlers).


If an atomic operation is marked `syncscope("<target-scope>")`, where
`<target-scope>` is a target specific synchronization scope, then it is
target dependent if it *synchronizes with* and participates in the
seq\_cst total orderings of other operations.

Otherwise, an atomic operation that is not marked
`syncscope("singlethread")` or `syncscope("<target-scope>")`
*synchronizes with* and participates in the seq\_cst total orderings of
other operations that are not marked `syncscope("singlethread")` or
`syncscope("<target-scope>")`.

### Floating-Point Environment

The default LLVM floating-point environment assumes that floating-point
instructions do not have side effects. Results assume the
round-to-nearest rounding mode. No floating-point exception state is
maintained in this environment. Therefore, there is no attempt to create
or preserve invalid operation (SNaN) or division-by-zero exceptions.

The benefit of this exception-free assumption is that floating-point
operations may be speculated freely without any other fast-math
relaxations to the floating-point model.

Code that requires different behavior than this should use the
`Constrained Floating-Point Intrinsics <constrainedfp>`{.interpreted-text
role="ref"}.

### Fast-Math Flags

LLVM IR floating-point operations (`fadd <i_fadd>`{.interpreted-text
role="ref"}, `fsub <i_fsub>`{.interpreted-text role="ref"},
`fmul <i_fmul>`{.interpreted-text role="ref"},
`fdiv <i_fdiv>`{.interpreted-text role="ref"},
`frem <i_frem>`{.interpreted-text role="ref"},
`fcmp <i_fcmp>`{.interpreted-text role="ref"}) and
`call <i_call>`{.interpreted-text role="ref"} may use the following
flags to enable otherwise unsafe floating-point transformations.

`nnan`

No NaNs - Allow optimizations to assume the arguments and result are
    not NaN. If an argument is a nan, or the result would be a nan, it
    produces a `poison value <poisonvalues>`{.interpreted-text
    role="ref"} instead.

`ninf`

No Infs - Allow optimizations to assume the arguments and result are
    not +/-Inf. If an argument is +/-Inf, or the result would be +/-Inf,
    it produces a `poison value <poisonvalues>`{.interpreted-text
    role="ref"} instead.

`nsz`

No Signed Zeros - Allow optimizations to treat the sign of a zero
    argument or result as insignificant.

`arcp`

Allow Reciprocal - Allow optimizations to use the reciprocal of an
    argument rather than perform division.

`contract`

Allow floating-point contraction (e.g. fusing a multiply followed by
    an addition into a fused multiply-and-add).

`afn`

Approximate functions - Allow substitution of approximate
    calculations for functions (sin, log, sqrt, etc). See floating-point
    intrinsic definitions for places where this can apply to LLVM\'s
    intrinsic math functions.

`reassoc`

Allow reassociation transformations for floating-point instructions.
    This may dramatically change results in floating-point.

`fast`

This flag implies all of the others.

### Use-list Order Directives

Use-list directives encode the in-memory order of each use-list,
allowing the order to be recreated. `<order-indexes>` is a
comma-separated list of indexes that are assigned to the referenced
value\'s uses. The referenced value\'s use-list is immediately sorted by
these indexes.

Use-list directives may appear at function scope or global scope. They
are not instructions, and have no effect on the semantics of the IR.
When they\'re at function scope, they must appear after the terminator
of the final basic block.

If basic blocks have their address taken via `blockaddress()`
expressions, `uselistorder_bb` can be used to reorder their use-lists
from outside their function\'s scope.

Syntax

<!-- -->

    uselistorder <ty> <value>, { <order-indexes> }
    uselistorder_bb @function, %block { <order-indexes> }

Examples

<!-- -->

    define void @foo(i32 %arg1, i32 %arg2) {
    entry:
      ; ... instructions ...
    bb:
      ; ... instructions ...

      ; At function scope.
      uselistorder i32 %arg1, { 1, 0, 2 }
      uselistorder label %bb, { 1, 0 }
    }

    ; At global scope.
    uselistorder i32* @global, { 1, 2, 0 }
    uselistorder i32 7, { 1, 0 }
    uselistorder i32 (i32) @bar, { 1, 0 }
    uselistorder_bb @foo, %bb, { 5, 1, 3, 2, 0, 4 }

### Source Filename

The *source filename* string is set to the original module identifier,
which will be the name of the compiled source file when compiling from
source through the clang front end, for example. It is then preserved
through the IR and bitcode.

This is currently necessary to generate a consistent unique global
identifier for local functions used in profile data, which prepends the
source file name to the local function name.

The syntax for the source file name is simply:

``` {.text}
source_filename = "/path/to/source.c"
```

