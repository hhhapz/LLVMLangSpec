### Other Operations {#otherops}

The instructions in this category are the \"miscellaneous\"
instructions, which defy better classification.

#### \'`icmp`\' Instruction {#i_icmp}

##### Syntax:

    <result> = icmp <cond> <ty> <op1>, <op2>   ; yields i1 or <N x i1>:result

##### Overview:

The \'`icmp`\' instruction returns a boolean value or a vector of
boolean values based on comparison of its two integer, integer vector,
pointer, or pointer vector operands.

##### Arguments:

The \'`icmp`\' instruction takes three operands. The first operand is
the condition code indicating the kind of comparison to perform. It is
not a value, just a keyword. The possible condition codes are:

1.  `eq`: equal
2.  `ne`: not equal
3.  `ugt`: unsigned greater than
4.  `uge`: unsigned greater or equal
5.  `ult`: unsigned less than
6.  `ule`: unsigned less or equal
7.  `sgt`: signed greater than
8.  `sge`: signed greater or equal
9.  `slt`: signed less than
10. `sle`: signed less or equal

The remaining two arguments must be
`integer <t_integer>`{.interpreted-text role="ref"} or
`pointer <t_pointer>`{.interpreted-text role="ref"} or integer
`vector <t_vector>`{.interpreted-text role="ref"} typed. They must also
be identical types.

##### Semantics:

The \'`icmp`\' compares `op1` and `op2` according to the condition code
given as `cond`. The comparison performed always yields either an
`i1 <t_integer>`{.interpreted-text role="ref"} or vector of `i1` result,
as follows:

1.  `eq`: yields `true` if the operands are equal, `false` otherwise. No
    sign interpretation is necessary or performed.
2.  `ne`: yields `true` if the operands are unequal, `false` otherwise.
    No sign interpretation is necessary or performed.
3.  `ugt`: interprets the operands as unsigned values and yields `true`
    if `op1` is greater than `op2`.
4.  `uge`: interprets the operands as unsigned values and yields `true`
    if `op1` is greater than or equal to `op2`.
5.  `ult`: interprets the operands as unsigned values and yields `true`
    if `op1` is less than `op2`.
6.  `ule`: interprets the operands as unsigned values and yields `true`
    if `op1` is less than or equal to `op2`.
7.  `sgt`: interprets the operands as signed values and yields `true` if
    `op1` is greater than `op2`.
8.  `sge`: interprets the operands as signed values and yields `true` if
    `op1` is greater than or equal to `op2`.
9.  `slt`: interprets the operands as signed values and yields `true` if
    `op1` is less than `op2`.
10. `sle`: interprets the operands as signed values and yields `true` if
    `op1` is less than or equal to `op2`.

If the operands are `pointer <t_pointer>`{.interpreted-text role="ref"}
typed, the pointer values are compared as if they were integers.

If the operands are integer vectors, then they are compared element by
element. The result is an `i1` vector with the same number of elements
as the values being compared. Otherwise, the result is an `i1`.

##### Example:

``` {.text}
<result> = icmp eq i32 4, 5          ; yields: result=false
<result> = icmp ne float* %X, %X     ; yields: result=false
<result> = icmp ult i16  4, 5        ; yields: result=true
<result> = icmp sgt i16  4, 5        ; yields: result=false
<result> = icmp ule i16 -4, 5        ; yields: result=false
<result> = icmp sge i16  4, 5        ; yields: result=false
```

#### \'`fcmp`\' Instruction {#i_fcmp}

##### Syntax:

    <result> = fcmp [fast-math flags]* <cond> <ty> <op1>, <op2>     ; yields i1 or <N x i1>:result

##### Overview:

The \'`fcmp`\' instruction returns a boolean value or vector of boolean
values based on comparison of its operands.

If the operands are floating-point scalars, then the result type is a
boolean (`i1 <t_integer>`{.interpreted-text role="ref"}).

If the operands are floating-point vectors, then the result type is a
vector of boolean with the same number of elements as the operands being
compared.

##### Arguments:

The \'`fcmp`\' instruction takes three operands. The first operand is
the condition code indicating the kind of comparison to perform. It is
not a value, just a keyword. The possible condition codes are:

1.  `false`: no comparison, always returns false
2.  `oeq`: ordered and equal
3.  `ogt`: ordered and greater than
4.  `oge`: ordered and greater than or equal
5.  `olt`: ordered and less than
6.  `ole`: ordered and less than or equal
7.  `one`: ordered and not equal
8.  `ord`: ordered (no nans)
9.  `ueq`: unordered or equal
10. `ugt`: unordered or greater than
11. `uge`: unordered or greater than or equal
12. `ult`: unordered or less than
13. `ule`: unordered or less than or equal
14. `une`: unordered or not equal
15. `uno`: unordered (either nans)
16. `true`: no comparison, always returns true

*Ordered* means that neither operand is a QNAN while *unordered* means
that either operand may be a QNAN.

Each of `val1` and `val2` arguments must be either a `floating-point
<t_floating>`{.interpreted-text role="ref"} type or a
`vector <t_vector>`{.interpreted-text role="ref"} of floating-point
type. They must have identical types.

##### Semantics:

The \'`fcmp`\' instruction compares `op1` and `op2` according to the
condition code given as `cond`. If the operands are vectors, then the
vectors are compared element by element. Each comparison performed
always yields an `i1 <t_integer>`{.interpreted-text role="ref"} result,
as follows:

1.  `false`: always yields `false`, regardless of operands.
2.  `oeq`: yields `true` if both operands are not a QNAN and `op1` is
    equal to `op2`.
3.  `ogt`: yields `true` if both operands are not a QNAN and `op1` is
    greater than `op2`.
4.  `oge`: yields `true` if both operands are not a QNAN and `op1` is
    greater than or equal to `op2`.
5.  `olt`: yields `true` if both operands are not a QNAN and `op1` is
    less than `op2`.
6.  `ole`: yields `true` if both operands are not a QNAN and `op1` is
    less than or equal to `op2`.
7.  `one`: yields `true` if both operands are not a QNAN and `op1` is
    not equal to `op2`.
8.  `ord`: yields `true` if both operands are not a QNAN.
9.  `ueq`: yields `true` if either operand is a QNAN or `op1` is equal
    to `op2`.
10. `ugt`: yields `true` if either operand is a QNAN or `op1` is greater
    than `op2`.
11. `uge`: yields `true` if either operand is a QNAN or `op1` is greater
    than or equal to `op2`.
12. `ult`: yields `true` if either operand is a QNAN or `op1` is less
    than `op2`.
13. `ule`: yields `true` if either operand is a QNAN or `op1` is less
    than or equal to `op2`.
14. `une`: yields `true` if either operand is a QNAN or `op1` is not
    equal to `op2`.
15. `uno`: yields `true` if either operand is a QNAN.
16. `true`: always yields `true`, regardless of operands.

The `fcmp` instruction can also optionally take any number of
`fast-math flags <fastmath>`{.interpreted-text role="ref"}, which are
optimization hints to enable otherwise unsafe floating-point
optimizations.

Any set of fast-math flags are legal on an `fcmp` instruction, but the
only flags that have any effect on its semantics are those that allow
assumptions to be made about the values of input arguments; namely
`nnan`, `ninf`, and `reassoc`. See `fastmath`{.interpreted-text
role="ref"} for more information.

##### Example:

``` {.text}
<result> = fcmp oeq float 4.0, 5.0    ; yields: result=false
<result> = fcmp one float 4.0, 5.0    ; yields: result=true
<result> = fcmp olt float 4.0, 5.0    ; yields: result=true
<result> = fcmp ueq double 1.0, 2.0   ; yields: result=false
```

#### \'`phi`\' Instruction {#i_phi}

##### Syntax:

    <result> = phi <ty> [ <val0>, <label0>], ...

##### Overview:

The \'`phi`\' instruction is used to implement the φ node in the SSA
graph representing the function.

##### Arguments:

The type of the incoming values is specified with the first type field.
After this, the \'`phi`\' instruction takes a list of pairs as
arguments, with one pair for each predecessor basic block of the current
block. Only values of `first class <t_firstclass>`{.interpreted-text
role="ref"} type may be used as the value arguments to the PHI node.
Only labels may be used as the label arguments.

There must be no non-phi instructions between the start of a basic block
and the PHI instructions: i.e. PHI instructions must be first in a basic
block.

For the purposes of the SSA form, the use of each incoming value is
deemed to occur on the edge from the corresponding predecessor block to
the current block (but after any definition of an \'`invoke`\'
instruction\'s return value on the same edge).

##### Semantics:

At runtime, the \'`phi`\' instruction logically takes on the value
specified by the pair corresponding to the predecessor basic block that
executed just prior to the current block.

##### Example:

``` {.llvm}
Loop:       ; Infinite loop that counts from 0 on up...
  %indvar = phi i32 [ 0, %LoopHeader ], [ %nextindvar, %Loop ]
  %nextindvar = add i32 %indvar, 1
  br label %Loop
```

#### \'`select`\' Instruction {#i_select}

##### Syntax:

    <result> = select selty <cond>, <ty> <val1>, <ty> <val2>             ; yields ty

    selty is either i1 or {<N x i1>}

##### Overview:

The \'`select`\' instruction is used to choose one value based on a
condition, without IR-level branching.

##### Arguments:

The \'`select`\' instruction requires an \'i1\' value or a vector of
\'i1\' values indicating the condition, and two values of the same
`first
class <t_firstclass>`{.interpreted-text role="ref"} type.

##### Semantics:

If the condition is an i1 and it evaluates to 1, the instruction returns
the first value argument; otherwise, it returns the second value
argument.

If the condition is a vector of i1, then the value arguments must be
vectors of the same size, and the selection is done element by element.

If the condition is an i1 and the value arguments are vectors of the
same size, then an entire vector is selected.

##### Example:

``` {.llvm}
%X = select i1 true, i8 17, i8 42          ; yields i8:17
```

#### \'`call`\' Instruction {#i_call}

##### Syntax:

    <result> = [tail | musttail | notail ] call [fast-math flags] [cconv] [ret attrs] [addrspace(<num>)]
               [<ty>|<fnty> <fnptrval>(<function args>) [fn attrs] [ operand bundles ]

##### Overview:

The \'`call`\' instruction represents a simple function call.

##### Arguments:

This instruction requires several arguments:

1.  The optional `tail` and `musttail` markers indicate that the
    optimizers should perform tail call optimization. The `tail` marker
    is a hint that [can be ignored](CodeGenerator.html#sibcallopt). The
    `musttail` marker means that the call must be tail call optimized in
    order for the program to be correct. The `musttail` marker provides
    these guarantees:

    1.  The call will not cause unbounded stack growth if it is part of
        a recursive cycle in the call graph.
    2.  Arguments with the `inalloca <attr_inalloca>`{.interpreted-text
        role="ref"} attribute are forwarded in place.

    Both markers imply that the callee does not access allocas from the
    caller. The `tail` marker additionally implies that the callee does
    not access varargs from the caller, while `musttail` implies that
    varargs from the caller are passed to the callee. Calls marked
    `musttail` must obey the following additional rules:

    -   The call must immediately precede a
        `ret <i_ret>`{.interpreted-text role="ref"} instruction, or a
        pointer bitcast followed by a ret instruction.
    -   The ret instruction must return the (possibly bitcasted) value
        produced by the call or void.
    -   The caller and callee prototypes must match. Pointer types of
        parameters or return types may differ in pointee type, but not
        in address space.
    -   The calling conventions of the caller and callee must match.
    -   All ABI-impacting function attributes, such as sret, byval,
        inreg, returned, and inalloca, must match.
    -   The callee must be varargs iff the caller is varargs. Bitcasting
        a non-varargs function to the appropriate varargs type is legal
        so long as the non-varargs prefixes obey the other rules.

    Tail call optimization for calls marked `tail` is guaranteed to
    occur if the following conditions are met:

    -   Caller and callee both have the calling convention `fastcc`.
    -   The call is in tail position (ret immediately follows call and
        ret uses value of call or is void).
    -   Option `-tailcallopt` is enabled, or
        `llvm::GuaranteedTailCallOpt` is `true`.
    -   [Platform-specific constraints are
        met.](CodeGenerator.html#tailcallopt)

2.  The optional `notail` marker indicates that the optimizers should
    not add `tail` or `musttail` markers to the call. It is used to
    prevent tail call optimization from being performed on the call.

3.  The optional `fast-math flags` marker indicates that the call has
    one or more `fast-math flags <fastmath>`{.interpreted-text
    role="ref"}, which are optimization hints to enable otherwise unsafe
    floating-point optimizations. Fast-math flags are only valid for
    calls that return a floating-point scalar or vector type.

4.  The optional \"cconv\" marker indicates which `calling
    convention <callingconv>`{.interpreted-text role="ref"} the call
    should use. If none is specified, the call defaults to using C
    calling conventions. The calling convention of the call must match
    the calling convention of the target function, or else the behavior
    is undefined.

5.  The optional `Parameter Attributes <paramattrs>`{.interpreted-text
    role="ref"} list for return values. Only \'`zeroext`\',
    \'`signext`\', and \'`inreg`\' attributes are valid here.

6.  The optional addrspace attribute can be used to indicate the address
    space of the called function. If it is not specified, the program
    address space from the
    `datalayout string<langref_datalayout>`{.interpreted-text
    role="ref"} will be used.

7.  \'`ty`\': the type of the call instruction itself which is also the
    type of the return value. Functions that return no value are marked
    `void`.

8.  \'`fnty`\': shall be the signature of the function being called. The
    argument types must match the types implied by this signature. This
    type can be omitted if the function is not varargs.

9.  \'`fnptrval`\': An LLVM value containing a pointer to a function to
    be called. In most cases, this is a direct function call, but
    indirect `call`\'s are just as possible, calling an arbitrary
    pointer to function value.

10. \'`function args`\': argument list whose types match the function
    signature argument types and parameter attributes. All arguments
    must be of `first class <t_firstclass>`{.interpreted-text
    role="ref"} type. If the function signature indicates the function
    accepts a variable number of arguments, the extra arguments can be
    specified.

11. The optional `function attributes <fnattrs>`{.interpreted-text
    role="ref"} list.

12. The optional `operand bundles <opbundles>`{.interpreted-text
    role="ref"} list.

##### Semantics:

The \'`call`\' instruction is used to cause control flow to transfer to
a specified function, with its incoming arguments bound to the specified
values. Upon a \'`ret`\' instruction in the called function, control
flow continues with the instruction after the function call, and the
return value of the function is bound to the result argument.

##### Example:

``` {.llvm}
%retval = call i32 @test(i32 %argc)
call i32 (i8*, ...)* @printf(i8* %msg, i32 12, i8 42)        ; yields i32
%X = tail call i32 @foo()                                    ; yields i32
%Y = tail call fastcc i32 @foo()  ; yields i32
call void %foo(i8 97 signext)

%struct.A = type { i32, i8 }
%r = call %struct.A @foo()                        ; yields { i32, i8 }
%gr = extractvalue %struct.A %r, 0                ; yields i32
%gr1 = extractvalue %struct.A %r, 1               ; yields i8
%Z = call void @foo() noreturn                    ; indicates that %foo never returns normally
%ZZ = call zeroext i32 @bar()                     ; Return value is %zero extended
```

llvm treats calls to some functions with names and arguments that match
the standard C99 library as being the C99 library functions, and may
perform optimizations or generate code for them under that assumption.
This is something we\'d like to change in the future to provide better
support for freestanding environments and non-C-based languages.

#### \'`va_arg`\' Instruction {#i_va_arg}

##### Syntax:

    <resultval> = va_arg <va_list*> <arglist>, <argty>

##### Overview:

The \'`va_arg`\' instruction is used to access arguments passed through
the \"variable argument\" area of a function call. It is used to
implement the `va_arg` macro in C.

##### Arguments:

This instruction takes a `va_list*` value and the type of the argument.
It returns a value of the specified argument type and increments the
`va_list` to point to the next argument. The actual type of `va_list` is
target specific.

##### Semantics:

The \'`va_arg`\' instruction loads an argument of the specified type
from the specified `va_list` and causes the `va_list` to point to the
next argument. For more information, see the variable argument handling
`Intrinsic Functions <int_varargs>`{.interpreted-text role="ref"}.

It is legal for this instruction to be called in a function which does
not take a variable number of arguments, for example, the `vfprintf`
function.

`va_arg` is an LLVM instruction instead of an `intrinsic
function <intrinsics>`{.interpreted-text role="ref"} because it takes a
type as an argument.

##### Example:

See the `variable argument processing <int_varargs>`{.interpreted-text
role="ref"} section.

Note that the code generator does not yet fully support va\_arg on many
targets. Also, it does not currently support va\_arg with aggregate
types on any target.

#### \'`landingpad`\' Instruction {#i_landingpad}

##### Syntax:

    <resultval> = landingpad <resultty> <clause>+
    <resultval> = landingpad <resultty> cleanup <clause>*

    <clause> := catch <type> <value>
    <clause> := filter <array constant type> <array constant>

##### Overview:

The \'`landingpad`\' instruction is used by [LLVM\'s exception handling
system](ExceptionHandling.html#overview) to specify that a basic block
is a landing pad \-\-- one where the exception lands, and corresponds to
the code found in the `catch` portion of a `try`/`catch` sequence. It
defines values supplied by the
`personality function <personalityfn>`{.interpreted-text role="ref"}
upon re-entry to the function. The `resultval` has the type `resultty`.

##### Arguments:

The optional `cleanup` flag indicates that the landing pad block is a
cleanup.

A `clause` begins with the clause type \-\-- `catch` or `filter` \-\--
and contains the global variable representing the \"type\" that may be
caught or filtered respectively. Unlike the `catch` clause, the `filter`
clause takes an array constant as its argument. Use
\"`[0 x i8**] undef`\" for a filter which cannot throw. The
\'`landingpad`\' instruction must contain *at least* one `clause` or the
`cleanup` flag.

##### Semantics:

The \'`landingpad`\' instruction defines the values which are set by the
`personality function <personalityfn>`{.interpreted-text role="ref"}
upon re-entry to the function, and therefore the \"result type\" of the
`landingpad` instruction. As with calling conventions, how the
personality function results are represented in LLVM IR is target
specific.

The clauses are applied in order from top to bottom. If two `landingpad`
instructions are merged together through inlining, the clauses from the
calling function are appended to the list of clauses. When the call
stack is being unwound due to an exception being thrown, the exception
is compared against each `clause` in turn. If it doesn\'t match any of
the clauses, and the `cleanup` flag is not set, then unwinding continues
further up the call stack.

The `landingpad` instruction has several restrictions:

-   A landing pad block is a basic block which is the unwind destination
    of an \'`invoke`\' instruction.
-   A landing pad block must have a \'`landingpad`\' instruction as its
    first non-PHI instruction.
-   There can be only one \'`landingpad`\' instruction within the
    landing pad block.
-   A basic block that is not a landing pad block may not include a
    \'`landingpad`\' instruction.

##### Example:

``` {.llvm}
;; A landing pad which can catch an integer.
%res = landingpad { i8*, i32 }
         catch i8** @_ZTIi
;; A landing pad that is a cleanup.
%res = landingpad { i8*, i32 }
         cleanup
;; A landing pad which can catch an integer and can only throw a double.
%res = landingpad { i8*, i32 }
         catch i8** @_ZTIi
         filter [1 x i8**] [@_ZTId]
```

#### \'`catchpad`\' Instruction {#i_catchpad}

##### Syntax:

    <resultval> = catchpad within <catchswitch> [<args>*]

##### Overview:

The \'`catchpad`\' instruction is used by [LLVM\'s exception handling
system](ExceptionHandling.html#overview) to specify that a basic block
begins a catch handler \-\-- one where a personality routine attempts to
transfer control to catch an exception.

##### Arguments:

The `catchswitch` operand must always be a token produced by a
`catchswitch <i_catchswitch>`{.interpreted-text role="ref"} instruction
in a predecessor block. This ensures that each `catchpad` has exactly
one predecessor block, and it always terminates in a `catchswitch`.

The `args` correspond to whatever information the personality routine
requires to know if this is an appropriate handler for the exception.
Control will transfer to the `catchpad` if this is the first appropriate
handler for the exception.

The `resultval` has the type `token <t_token>`{.interpreted-text
role="ref"} and is used to match the `catchpad` to corresponding
`catchrets <i_catchret>`{.interpreted-text role="ref"} and other nested
EH pads.

##### Semantics:

When the call stack is being unwound due to an exception being thrown,
the exception is compared against the `args`. If it doesn\'t match,
control will not reach the `catchpad` instruction. The representation of
`args` is entirely target and personality function-specific.

Like the `landingpad <i_landingpad>`{.interpreted-text role="ref"}
instruction, the `catchpad` instruction must be the first non-phi of its
parent basic block.

The meaning of the tokens produced and consumed by `catchpad` and other
\"pad\" instructions is described in the [Windows exception handling
documentation](ExceptionHandling.html#wineh).

When a `catchpad` has been \"entered\" but not yet \"exited\" (as
described in the [EH
documentation](ExceptionHandling.html#wineh-constraints)), it is
undefined behavior to execute a `call <i_call>`{.interpreted-text
role="ref"} or `invoke <i_invoke>`{.interpreted-text role="ref"} that
does not carry an appropriate
`"funclet" bundle <ob_funclet>`{.interpreted-text role="ref"}.

##### Example:

``` {.text}
dispatch:
  %cs = catchswitch within none [label %handler0] unwind to caller
  ;; A catch block which can catch an integer.
handler0:
  %tok = catchpad within %cs [i8** @_ZTIi]
```

#### \'`cleanuppad`\' Instruction {#i_cleanuppad}

##### Syntax:

    <resultval> = cleanuppad within <parent> [<args>*]

##### Overview:

The \'`cleanuppad`\' instruction is used by [LLVM\'s exception handling
system](ExceptionHandling.html#overview) to specify that a basic block
is a cleanup block \-\-- one where a personality routine attempts to
transfer control to run cleanup actions. The `args` correspond to
whatever additional information the
`personality function <personalityfn>`{.interpreted-text role="ref"}
requires to execute the cleanup. The `resultval` has the type
`token <t_token>`{.interpreted-text role="ref"} and is used to match the
`cleanuppad` to corresponding
`cleanuprets <i_cleanupret>`{.interpreted-text role="ref"}. The `parent`
argument is the token of the funclet that contains the `cleanuppad`
instruction. If the `cleanuppad` is not inside a funclet, this operand
may be the token `none`.

##### Arguments:

The instruction takes a list of arbitrary values which are interpreted
by the `personality function <personalityfn>`{.interpreted-text
role="ref"}.

##### Semantics:

When the call stack is being unwound due to an exception being thrown,
the `personality function <personalityfn>`{.interpreted-text role="ref"}
transfers control to the `cleanuppad` with the aid of the
personality-specific arguments. As with calling conventions, how the
personality function results are represented in LLVM IR is target
specific.

The `cleanuppad` instruction has several restrictions:

-   A cleanup block is a basic block which is the unwind destination of
    an exceptional instruction.
-   A cleanup block must have a \'`cleanuppad`\' instruction as its
    first non-PHI instruction.
-   There can be only one \'`cleanuppad`\' instruction within the
    cleanup block.
-   A basic block that is not a cleanup block may not include a
    \'`cleanuppad`\' instruction.

When a `cleanuppad` has been \"entered\" but not yet \"exited\" (as
described in the [EH
documentation](ExceptionHandling.html#wineh-constraints)), it is
undefined behavior to execute a `call <i_call>`{.interpreted-text
role="ref"} or `invoke <i_invoke>`{.interpreted-text role="ref"} that
does not carry an appropriate
`"funclet" bundle <ob_funclet>`{.interpreted-text role="ref"}.

##### Example:

``` {.text}
%tok = cleanuppad within %cs []
```

