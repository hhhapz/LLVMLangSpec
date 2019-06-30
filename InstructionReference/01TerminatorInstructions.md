#### `ret` Instruction {#i_ret}

##### Syntax:

    ret <type> <value>       ; Return a value from a non-void function
    ret void                 ; Return from void function

##### Overview:

The `ret` instruction is used to return control flow (and optionally
a value) from a function back to the caller.

There are two forms of the `ret` instruction: one that returns a
value and then causes control flow, and one that just causes control
flow to occur.

##### Arguments:

The `ret` instruction optionally accepts a single argument, the
return value. The type of the return value must be a `first
class <t_firstclass>`{.interpreted-text role="ref"} type.

A function is not `well formed <wellformed>`{.interpreted-text
role="ref"} if it it has a non-void return type and contains a `ret`
instruction with no return value or a return value with a type that does
not match its type, or if it has a void return type and contains a
`ret` instruction with a return value.

##### Semantics:

When the `ret` instruction is executed, control flow returns back to
the calling functions context. If the caller is a
\"`call <i_call>`{.interpreted-text role="ref"}\" instruction, execution
continues at the instruction after the call. If the caller was an
\"`invoke <i_invoke>`{.interpreted-text role="ref"}\" instruction,
execution continues at the beginning of the \"normal\" destination
block. If the instruction returns a value, that value shall set the call
or invoke instructions return value.

##### Example:

``` {.llvm}
ret i32 5                       ; Return an integer value of 5
ret void                        ; Return from a void function
ret { i32, i8 } { i32 4, i8 2 } ; Return a struct of values 4 and 2
```

#### `br` Instruction {#i_br}

##### Syntax:

    br i1 <cond>, label <iftrue>, label <iffalse>
    br label <dest>          ; Unconditional branch

##### Overview:

The `br` instruction is used to cause control flow to transfer to a
different basic block in the current function. There are two forms of
this instruction, corresponding to a conditional branch and an
unconditional branch.

##### Arguments:

The conditional branch form of the `br` instruction takes a single
`i1` value and two `label` values. The unconditional form of the
`br` instruction takes a single `label` value as a target.

##### Semantics:

Upon execution of a conditional `br` instruction, the `i1`
argument is evaluated. If the value is `true`, control flows to the
`iftrue` `label` argument. If \"cond\" is `false`, control flows to
the `iffalse` `label` argument.

##### Example:

``` {.llvm}
Test:
  %cond = icmp eq i32 %a, %b
  br i1 %cond, label %IfEqual, label %IfUnequal
IfEqual:
  ret i32 1
IfUnequal:
  ret i32 0
```

#### `switch` Instruction {#i_switch}

##### Syntax:

    switch <intty> <value>, label <defaultdest> [ <intty> <val>, label <dest> ... ]

##### Overview:

The `switch` instruction is used to transfer control flow to one of
several different places. It is a generalization of the `br`
instruction, allowing a branch to occur to one of many possible
destinations.

##### Arguments:

The `switch` instruction uses three parameters: an integer
comparison value `value`, a default `label` destination, and an
array of pairs of comparison value constants and `label`s. The table
is not allowed to contain duplicate constant entries.

##### Semantics:

The `switch` instruction specifies a table of values and destinations.
When the `switch` instruction is executed, this table is searched
for the given value. If the value is found, control flow is transferred
to the corresponding destination; otherwise, control flow is transferred
to the default destination.

##### Implementation:

Depending on properties of the target machine and the particular
`switch` instruction, this instruction may be code generated in
different ways. For example, it could be generated as a series of
chained conditional branches or with a lookup table.

##### Example:

``` {.llvm}
; Emulate a conditional br instruction
%Val = zext i1 %value to i32
switch i32 %Val, label %truedest [ i32 0, label %falsedest ]

; Emulate an unconditional br instruction
switch i32 0, label %dest [ ]

; Implement a jump table:
switch i32 %val, label %otherwise [ i32 0, label %onzero
                                    i32 1, label %onone
                                    i32 2, label %ontwo ]
```

#### `indirectbr` Instruction {#i_indirectbr}

##### Syntax:

    indirectbr <somety>* <address>, [ label <dest1>, label <dest2>, ... ]

##### Overview:

The `indirectbr` instruction implements an indirect branch to a
label within the current function, whose address is specified by
\"`address`\". Address must be derived from a
`blockaddress <blockaddress>`{.interpreted-text role="ref"} constant.

##### Arguments:

The `address` argument is the address of the label to jump to. The
rest of the arguments indicate the full set of possible destinations
that the address may point to. Blocks are allowed to occur multiple
times in the destination list, though this isnt particularly useful.

This destination list is required so that dataflow analysis has an
accurate understanding of the CFG.

##### Semantics:

Control transfers to the block specified in the address argument. All
possible destination blocks must be listed in the label list, otherwise
this instruction has undefined behavior. This implies that jumps to
labels defined in other functions have undefined behavior as well.

##### Implementation:

This is typically implemented with a jump through a register.

##### Example:

``` {.llvm}
indirectbr i8* %Addr, [ label %bb1, label %bb2, label %bb3 ]
```

#### `invoke` Instruction {#i_invoke}

##### Syntax:

    <result> = invoke [cconv] [ret attrs] [addrspace(<num>)] [<ty>|<fnty> <fnptrval>(<function args>) [fn attrs]
                  [operand bundles] to label <normal label> unwind label <exception label>

##### Overview:

The `invoke` instruction causes control to transfer to a specified
function, with the possibility of control flow transfer to either the
`normal` label or the `exception` label. If the callee function
returns with the \"`ret`\" instruction, control flow will return to the
\"normal\" label. If the callee (or any indirect callees) returns via
the \"`resume <i_resume>`{.interpreted-text role="ref"}\" instruction or
other exception handling mechanism, control is interrupted and continued
at the dynamically nearest \"exception\" label.

The `exception` label is a [landing
pad](ExceptionHandling.html#overview) for the exception. As such,
`exception` label is required to have the
\"`landingpad <i_landingpad>`{.interpreted-text role="ref"}\"
instruction, which contains the information about the behavior of the
program after unwinding happens, as its first non-PHI instruction. The
restrictions on the \"`landingpad`\" instructions tightly couples it
to the \"`invoke`\" instruction, so that the important information
contained within the \"`landingpad`\" instruction cant be lost through
normal code motion.

##### Arguments:

This instruction requires several arguments:

1.  The optional \"cconv\" marker indicates which `calling
    convention <callingconv>`{.interpreted-text role="ref"} the call
    should use. If none is specified, the call defaults to using C
    calling conventions.
2.  The optional `Parameter Attributes <paramattrs>`{.interpreted-text
    role="ref"} list for return values. Only `zeroext`,
    `signext`, and `inreg` attributes are valid here.
3.  The optional addrspace attribute can be used to indicate the address
    space of the called function. If it is not specified, the program
    address space from the
    `datalayout string<langref_datalayout>`{.interpreted-text
    role="ref"} will be used.
4.  `ty`: the type of the call instruction itself which is also the
    type of the return value. Functions that return no value are marked
    `void`.
5.  `fnty`: shall be the signature of the function being invoked.
    The argument types must match the types implied by this signature.
    This type can be omitted if the function is not varargs.
6.  `fnptrval`: An LLVM value containing a pointer to a function to
    be invoked. In most cases, this is a direct function invocation, but
    indirect `invoke`s are just as possible, calling an arbitrary
    pointer to function value.
7.  `function args`: argument list whose types match the function
    signature argument types and parameter attributes. All arguments
    must be of `first class <t_firstclass>`{.interpreted-text
    role="ref"} type. If the function signature indicates the function
    accepts a variable number of arguments, the extra arguments can be
    specified.
8.  `normal label`: the label reached when the called function
    executes a `ret` instruction.
9.  `exception label`: the label reached when a callee returns via
    the `resume <i_resume>`{.interpreted-text role="ref"} instruction or
    other exception handling mechanism.
10. The optional `function attributes <fnattrs>`{.interpreted-text
    role="ref"} list.
11. The optional `operand bundles <opbundles>`{.interpreted-text
    role="ref"} list.

##### Semantics:

This instruction is designed to operate as a standard `call`
instruction in most regards. The primary difference is that it
establishes an association with a label, which is used by the runtime
library to unwind the stack.

This instruction is used in languages with destructors to ensure that
proper cleanup is performed in the case of either a `longjmp` or a
thrown exception. Additionally, this is important for implementation of
`catch` clauses in high-level languages that support them.

For the purposes of the SSA form, the definition of the value returned
by the `invoke` instruction is deemed to occur on the edge from the
current block to the \"normal\" label. If the callee unwinds then no
return value is available.

##### Example:

``` {.llvm}
%retval = invoke i32 @Test(i32 15) to label %Continue
            unwind label %TestCleanup              ; i32:retval set
%retval = invoke coldcc i32 %Testfnptr(i32 15) to label %Continue
            unwind label %TestCleanup              ; i32:retval set
```

#### `resume` Instruction {#i_resume}

##### Syntax:

    resume <type> <value>

##### Overview:

The `resume` instruction is a terminator instruction that has no
successors.

##### Arguments:

The `resume` instruction requires one argument, which must have the
same type as the result of any `landingpad` instruction in the same
function.

##### Semantics:

The `resume` instruction resumes propagation of an existing
(in-flight) exception whose unwinding was interrupted with a
`landingpad <i_landingpad>`{.interpreted-text role="ref"} instruction.

##### Example:

``` {.llvm}
resume { i8*, i32 } %exn
```

#### `catchswitch` Instruction {#i_catchswitch}

##### Syntax:

    <resultval> = catchswitch within <parent> [ label <handler1>, label <handler2>, ... ] unwind to caller
    <resultval> = catchswitch within <parent> [ label <handler1>, label <handler2>, ... ] unwind label <default>

##### Overview:

The `catchswitch` instruction is used by [LLVMs exception handling
system](ExceptionHandling.html#overview) to describe the set of possible
catch handlers that may be executed by the
`EH personality routine <personalityfn>`{.interpreted-text role="ref"}.

##### Arguments:

The `parent` argument is the token of the funclet that contains the
`catchswitch` instruction. If the `catchswitch` is not inside a funclet,
this operand may be the token `none`.

The `default` argument is the label of another basic block beginning
with either a `cleanuppad` or `catchswitch` instruction. This unwind
destination must be a legal target with respect to the `parent` links,
as described in the [exception handling
documentation](ExceptionHandling.html#wineh-constraints).

The `handlers` are a nonempty list of successor blocks that each begin
with a `catchpad <i_catchpad>`{.interpreted-text role="ref"}
instruction.

##### Semantics:

Executing this instruction transfers control to one of the successors in
`handlers`, if appropriate, or continues to unwind via the unwind label
if present.

The `catchswitch` is both a terminator and a \"pad\" instruction,
meaning that it must be both the first non-phi instruction and last
instruction in the basic block. Therefore, it must be the only non-phi
instruction in the block.

##### Example:

``` {.text}
dispatch1:
  %cs1 = catchswitch within none [label %handler0, label %handler1] unwind to caller
dispatch2:
  %cs2 = catchswitch within %parenthandler [label %handler0] unwind label %cleanup
```

#### `catchret` Instruction {#i_catchret}

##### Syntax:

    catchret from <token> to label <normal>

##### Overview:

The `catchret` instruction is a terminator instruction that has a
single successor.

##### Arguments:

The first argument to a `catchret` indicates which `catchpad` it
exits. It must be a `catchpad <i_catchpad>`{.interpreted-text
role="ref"}. The second argument to a `catchret` specifies where
control will transfer to next.

##### Semantics:

The `catchret` instruction ends an existing (in-flight) exception
whose unwinding was interrupted with a
`catchpad <i_catchpad>`{.interpreted-text role="ref"} instruction. The
`personality function <personalityfn>`{.interpreted-text role="ref"}
gets a chance to execute arbitrary code to, for example, destroy the
active exception. Control then transfers to `normal`.

The `token` argument must be a token produced by a `catchpad`
instruction. If the specified `catchpad` is not the
most-recently-entered not-yet-exited funclet pad (as described in the
[EH documentation](ExceptionHandling.html#wineh-constraints)), the
`catchret`s behavior is undefined.

##### Example:

``` {.text}
catchret from %catch label %continue
```

#### `cleanupret` Instruction {#i_cleanupret}

##### Syntax:

    cleanupret from <value> unwind label <continue>
    cleanupret from <value> unwind to caller

##### Overview:

The `cleanupret` instruction is a terminator instruction that has an
optional successor.

##### Arguments:

The `cleanupret` instruction requires one argument, which indicates
which `cleanuppad` it exits, and must be a
`cleanuppad <i_cleanuppad>`{.interpreted-text role="ref"}. If the
specified `cleanuppad` is not the most-recently-entered not-yet-exited
funclet pad (as described in the [EH
documentation](ExceptionHandling.html#wineh-constraints)), the
`cleanupret`s behavior is undefined.

The `cleanupret` instruction also has an optional successor,
`continue`, which must be the label of another basic block beginning
with either a `cleanuppad` or `catchswitch` instruction. This unwind
destination must be a legal target with respect to the `parent` links,
as described in the [exception handling
documentation](ExceptionHandling.html#wineh-constraints).

##### Semantics:

The `cleanupret` instruction indicates to the
`personality function <personalityfn>`{.interpreted-text role="ref"}
that one `cleanuppad <i_cleanuppad>`{.interpreted-text role="ref"} it
transferred control to has ended. It transfers control to `continue` or
unwinds out of the function.

##### Example:

``` {.text}
cleanupret from %cleanup unwind to caller
cleanupret from %cleanup unwind label %continue
```

#### `unreachable` Instruction {#i_unreachable}

##### Syntax:

    unreachable

##### Overview:

The `unreachable` instruction has no defined semantics. This
instruction is used to inform the optimizer that a particular portion of
the code is not reachable. This can be used to indicate that the code
after a no-return function cannot be reached, and other facts.

##### Semantics:

The `unreachable` instruction has no defined semantics.

