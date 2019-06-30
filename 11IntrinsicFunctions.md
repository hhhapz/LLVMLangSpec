Intrinsic Functions {#intrinsics}
-------------------

LLVM supports the notion of an \"intrinsic function\". These functions
have well known names and semantics and are required to follow certain
restrictions. Overall, these intrinsics represent an extension mechanism
for the LLVM language that does not require changing all of the
transformations in LLVM when adding to the language (or the bitcode
reader/writer, the parser, etc\...).

Intrinsic function names must all start with an \"`llvm.`\" prefix. This
prefix is reserved in LLVM for intrinsic names; thus, function names may
not begin with this prefix. Intrinsic functions must always be external
functions: you cannot define the body of intrinsic functions. Intrinsic
functions may only be used in call or invoke instructions: it is illegal
to take the address of an intrinsic function. Additionally, because
intrinsic functions are part of the LLVM language, it is required if any
are added that they be documented here.

Some intrinsic functions can be overloaded, i.e., the intrinsic
represents a family of functions that perform the same operation but on
different data types. Because LLVM can represent over 8 million
different integer types, overloading is used commonly to allow an
intrinsic function to operate on any integer type. One or more of the
argument types or the result type can be overloaded to accept any
integer type. Argument types may also be defined as exactly matching a
previous argument\'s type or the result type. This allows an intrinsic
function which accepts multiple arguments, but needs all of them to be
of the same type, to only be overloaded with respect to a single
argument or the result.

Overloaded intrinsics will have the names of its overloaded argument
types encoded into its function name, each preceded by a period. Only
those types which are overloaded result in a name suffix. Arguments
whose type is matched against another type do not. For example, the
`llvm.ctpop` function can take an integer of any width and returns an
integer of exactly the same integer width. This leads to a family of
functions such as `i8 @llvm.ctpop.i8(i8 %val)` and
`i29 @llvm.ctpop.i29(i29 %val)`. Only one type, the return type, is
overloaded, and only one type suffix is required. Because the
argument\'s type is matched against the return type, it does not require
its own name suffix.

To learn how to add an intrinsic function, please see the [Extending
LLVM Guide](ExtendingLLVM.html).

### Variable Argument Handling Intrinsics {#int_varargs}

Variable argument support is defined in LLVM with the
`va_arg <i_va_arg>`{.interpreted-text role="ref"} instruction and these
three intrinsic functions. These functions are related to the similarly
named macros defined in the `<stdarg.h>` header file.

All of these functions operate on arguments that use a target-specific
value type \"`va_list`\". The LLVM assembly language reference manual
does not define what this type is, so all transformations should be
prepared to handle these functions regardless of the type used.

This example shows how the `va_arg <i_va_arg>`{.interpreted-text
role="ref"} instruction and the variable argument handling intrinsic
functions are used.

``` {.llvm}
; This struct is different for every platform. For most platforms,
; it is merely an i8*.
%struct.va_list = type { i8* }

; For Unix x86_64 platforms, va_list is the following struct:
; %struct.va_list = type { i32, i32, i8*, i8* }

define i32 @test(i32 %X, ...) {
  ; Initialize variable argument processing
  %ap = alloca %struct.va_list
  %ap2 = bitcast %struct.va_list* %ap to i8*
  call void @llvm.va_start(i8* %ap2)

  ; Read a single integer argument
  %tmp = va_arg i8* %ap2, i32

  ; Demonstrate usage of llvm.va_copy and llvm.va_end
  %aq = alloca i8*
  %aq2 = bitcast i8** %aq to i8*
  call void @llvm.va_copy(i8* %aq2, i8* %ap2)
  call void @llvm.va_end(i8* %aq2)

  ; Stop processing of arguments.
  call void @llvm.va_end(i8* %ap2)
  ret i32 %tmp
}

declare void @llvm.va_start(i8*)
declare void @llvm.va_copy(i8*, i8*)
declare void @llvm.va_end(i8*)
```

#### \'`llvm.va_start`\' Intrinsic {#int_va_start}

##### Syntax:

    declare void @llvm.va_start(i8* <arglist>)

##### Overview:

The \'`llvm.va_start`\' intrinsic initializes `*<arglist>` for
subsequent use by `va_arg`.

##### Arguments:

The argument is a pointer to a `va_list` element to initialize.

##### Semantics:

The \'`llvm.va_start`\' intrinsic works just like the `va_start` macro
available in C. In a target-dependent way, it initializes the `va_list`
element to which the argument points, so that the next call to `va_arg`
will produce the first variable argument passed to the function. Unlike
the C `va_start` macro, this intrinsic does not need to know the last
argument of the function as the compiler can figure that out.

#### \'`llvm.va_end`\' Intrinsic

##### Syntax:

    declare void @llvm.va_end(i8* <arglist>)

##### Overview:

The \'`llvm.va_end`\' intrinsic destroys `*<arglist>`, which has been
initialized previously with `llvm.va_start` or `llvm.va_copy`.

##### Arguments:

The argument is a pointer to a `va_list` to destroy.

##### Semantics:

The \'`llvm.va_end`\' intrinsic works just like the `va_end` macro
available in C. In a target-dependent way, it destroys the `va_list`
element to which the argument points. Calls to
`llvm.va_start <int_va_start>`{.interpreted-text role="ref"} and
`llvm.va_copy <int_va_copy>`{.interpreted-text role="ref"} must be
matched exactly with calls to `llvm.va_end`.

#### \'`llvm.va_copy`\' Intrinsic {#int_va_copy}

##### Syntax:

    declare void @llvm.va_copy(i8* <destarglist>, i8* <srcarglist>)

##### Overview:

The \'`llvm.va_copy`\' intrinsic copies the current argument position
from the source argument list to the destination argument list.

##### Arguments:

The first argument is a pointer to a `va_list` element to initialize.
The second argument is a pointer to a `va_list` element to copy from.

##### Semantics:

The \'`llvm.va_copy`\' intrinsic works just like the `va_copy` macro
available in C. In a target-dependent way, it copies the source
`va_list` element into the destination `va_list` element. This intrinsic
is necessary because the `llvm.va_start` intrinsic may be arbitrarily
complex and require, for example, memory allocation.

### Accurate Garbage Collection Intrinsics

LLVM\'s support for [Accurate Garbage
Collection](GarbageCollection.html) (GC) requires the frontend to
generate code containing appropriate intrinsic calls and select an
appropriate GC strategy which knows how to lower these intrinsics in a
manner which is appropriate for the target collector.

These intrinsics allow identification of `GC roots on the
stack <int_gcroot>`{.interpreted-text role="ref"}, as well as garbage
collector implementations that require
`read <int_gcread>`{.interpreted-text role="ref"} and
`write <int_gcwrite>`{.interpreted-text role="ref"} barriers. Frontends
for type-safe garbage collected languages should generate these
intrinsics to make use of the LLVM garbage collectors. For more details,
see [Garbage Collection with LLVM](GarbageCollection.html).

#### Experimental Statepoint Intrinsics

LLVM provides an second experimental set of intrinsics for describing
garbage collection safepoints in compiled code. These intrinsics are an
alternative to the `llvm.gcroot` intrinsics, but are compatible with the
ones for `read <int_gcread>`{.interpreted-text role="ref"} and
`write <int_gcwrite>`{.interpreted-text role="ref"} barriers. The
differences in approach are covered in the [Garbage Collection with
LLVM](GarbageCollection.html) documentation. The intrinsics themselves
are described in `Statepoints`{.interpreted-text role="doc"}.

#### \'`llvm.gcroot`\' Intrinsic {#int_gcroot}

##### Syntax:

    declare void @llvm.gcroot(i8** %ptrloc, i8* %metadata)

##### Overview:

The \'`llvm.gcroot`\' intrinsic declares the existence of a GC root to
the code generator, and allows some metadata to be associated with it.

##### Arguments:

The first argument specifies the address of a stack object that contains
the root pointer. The second pointer (which must be either a constant or
a global value address) contains the meta-data to be associated with the
root.

##### Semantics:

At runtime, a call to this intrinsic stores a null pointer into the
\"ptrloc\" location. At compile-time, the code generator generates
information to allow the runtime to find the pointer at GC safe points.
The \'`llvm.gcroot`\' intrinsic may only be used in a function which
`specifies a GC algorithm <gc>`{.interpreted-text role="ref"}.

#### \'`llvm.gcread`\' Intrinsic {#int_gcread}

##### Syntax:

    declare i8* @llvm.gcread(i8* %ObjPtr, i8** %Ptr)

##### Overview:

The \'`llvm.gcread`\' intrinsic identifies reads of references from heap
locations, allowing garbage collector implementations that require read
barriers.

##### Arguments:

The second argument is the address to read from, which should be an
address allocated from the garbage collector. The first object is a
pointer to the start of the referenced object, if needed by the language
runtime (otherwise null).

##### Semantics:

The \'`llvm.gcread`\' intrinsic has the same semantics as a load
instruction, but may be replaced with substantially more complex code by
the garbage collector runtime, as needed. The \'`llvm.gcread`\'
intrinsic may only be used in a function which `specifies a GC
algorithm <gc>`{.interpreted-text role="ref"}.

#### \'`llvm.gcwrite`\' Intrinsic {#int_gcwrite}

##### Syntax:

    declare void @llvm.gcwrite(i8* %P1, i8* %Obj, i8** %P2)

##### Overview:

The \'`llvm.gcwrite`\' intrinsic identifies writes of references to heap
locations, allowing garbage collector implementations that require write
barriers (such as generational or reference counting collectors).

##### Arguments:

The first argument is the reference to store, the second is the start of
the object to store it to, and the third is the address of the field of
Obj to store to. If the runtime does not require a pointer to the
object, Obj may be null.

##### Semantics:

The \'`llvm.gcwrite`\' intrinsic has the same semantics as a store
instruction, but may be replaced with substantially more complex code by
the garbage collector runtime, as needed. The \'`llvm.gcwrite`\'
intrinsic may only be used in a function which `specifies a GC
algorithm <gc>`{.interpreted-text role="ref"}.

### Code Generator Intrinsics

These intrinsics are provided by LLVM to expose special features that
may only be implemented with code generator support.

#### \'`llvm.returnaddress`\' Intrinsic

##### Syntax:

    declare i8* @llvm.returnaddress(i32 <level>)

##### Overview:

The \'`llvm.returnaddress`\' intrinsic attempts to compute a
target-specific value indicating the return address of the current
function or one of its callers.

##### Arguments:

The argument to this intrinsic indicates which function to return the
address for. Zero indicates the calling function, one indicates its
caller, etc. The argument is **required** to be a constant integer
value.

##### Semantics:

The \'`llvm.returnaddress`\' intrinsic either returns a pointer
indicating the return address of the specified call frame, or zero if it
cannot be identified. The value returned by this intrinsic is likely to
be incorrect or 0 for arguments other than zero, so it should only be
used for debugging purposes.

Note that calling this intrinsic does not prevent function inlining or
other aggressive transformations, so the value returned may not be that
of the obvious source-language caller.

#### \'`llvm.addressofreturnaddress`\' Intrinsic

##### Syntax:

    declare i8* @llvm.addressofreturnaddress()

##### Overview:

The \'`llvm.addressofreturnaddress`\' intrinsic returns a
target-specific pointer to the place in the stack frame where the return
address of the current function is stored.

##### Semantics:

Note that calling this intrinsic does not prevent function inlining or
other aggressive transformations, so the value returned may not be that
of the obvious source-language caller.

This intrinsic is only implemented for x86 and aarch64.

#### \'`llvm.sponentry`\' Intrinsic

##### Syntax:

    declare i8* @llvm.sponentry()

##### Overview:

The \'`llvm.sponentry`\' intrinsic returns the stack pointer value at
the entry of the current function calling this intrinsic.

##### Semantics:

Note this intrinsic is only verified on AArch64.

#### \'`llvm.frameaddress`\' Intrinsic

##### Syntax:

    declare i8* @llvm.frameaddress(i32 <level>)

##### Overview:

The \'`llvm.frameaddress`\' intrinsic attempts to return the
target-specific frame pointer value for the specified stack frame.

##### Arguments:

The argument to this intrinsic indicates which function to return the
frame pointer for. Zero indicates the calling function, one indicates
its caller, etc. The argument is **required** to be a constant integer
value.

##### Semantics:

The \'`llvm.frameaddress`\' intrinsic either returns a pointer
indicating the frame address of the specified call frame, or zero if it
cannot be identified. The value returned by this intrinsic is likely to
be incorrect or 0 for arguments other than zero, so it should only be
used for debugging purposes.

Note that calling this intrinsic does not prevent function inlining or
other aggressive transformations, so the value returned may not be that
of the obvious source-language caller.

#### \'`llvm.localescape`\' and \'`llvm.localrecover`\' Intrinsics

##### Syntax:

    declare void @llvm.localescape(...)
    declare i8* @llvm.localrecover(i8* %func, i8* %fp, i32 %idx)

##### Overview:

The \'`llvm.localescape`\' intrinsic escapes offsets of a collection of
static allocas, and the \'`llvm.localrecover`\' intrinsic applies those
offsets to a live frame pointer to recover the address of the
allocation. The offset is computed during frame layout of the caller of
`llvm.localescape`.

##### Arguments:

All arguments to \'`llvm.localescape`\' must be pointers to static
allocas or casts of static allocas. Each function can only call
\'`llvm.localescape`\' once, and it can only do so from the entry block.

The `func` argument to \'`llvm.localrecover`\' must be a constant
bitcasted pointer to a function defined in the current module. The code
generator cannot determine the frame allocation offset of functions
defined in other modules.

The `fp` argument to \'`llvm.localrecover`\' must be a frame pointer of
a call frame that is currently live. The return value of
\'`llvm.localaddress`\' is one way to produce such a value, but various
runtimes also expose a suitable pointer in platform-specific ways.

The `idx` argument to \'`llvm.localrecover`\' indicates which alloca
passed to \'`llvm.localescape`\' to recover. It is zero-indexed.

##### Semantics:

These intrinsics allow a group of functions to share access to a set of
local stack allocations of a one parent function. The parent function
may call the \'`llvm.localescape`\' intrinsic once from the function
entry block, and the child functions can use \'`llvm.localrecover`\' to
access the escaped allocas. The \'`llvm.localescape`\' intrinsic blocks
inlining, as inlining changes where the escaped allocas are allocated,
which would break attempts to use \'`llvm.localrecover`\'.

#### \'`llvm.read_register`\' and \'`llvm.write_register`\' Intrinsics[]{#int_read_register} {#int_write_register}

##### Syntax:

    declare i32 @llvm.read_register.i32(metadata)
    declare i64 @llvm.read_register.i64(metadata)
    declare void @llvm.write_register.i32(metadata, i32 @value)
    declare void @llvm.write_register.i64(metadata, i64 @value)
    !0 = !{!"sp\00"}

##### Overview:

The \'`llvm.read_register`\' and \'`llvm.write_register`\' intrinsics
provides access to the named register. The register must be valid on the
architecture being compiled to. The type needs to be compatible with the
register being read.

##### Semantics:

The \'`llvm.read_register`\' intrinsic returns the current value of the
register, where possible. The \'`llvm.write_register`\' intrinsic sets
the current value of the register, where possible.

This is useful to implement named register global variables that need to
always be mapped to a specific register, as is common practice on
bare-metal programs including OS kernels.

The compiler doesn\'t check for register availability or use of the used
register in surrounding code, including inline assembly. Because of
that, allocatable registers are not supported.

Warning: So far it only works with the stack pointer on selected
architectures (ARM, AArch64, PowerPC and x86\_64). Significant amount of
work is needed to support other registers and even more so, allocatable
registers.

#### \'`llvm.stacksave`\' Intrinsic {#int_stacksave}

##### Syntax:

    declare i8* @llvm.stacksave()

##### Overview:

The \'`llvm.stacksave`\' intrinsic is used to remember the current state
of the function stack, for use with
`llvm.stackrestore <int_stackrestore>`{.interpreted-text role="ref"}.
This is useful for implementing language features like scoped automatic
variable sized arrays in C99.

##### Semantics:

This intrinsic returns a opaque pointer value that can be passed to
`llvm.stackrestore <int_stackrestore>`{.interpreted-text role="ref"}.
When an `llvm.stackrestore` intrinsic is executed with a value saved
from `llvm.stacksave`, it effectively restores the state of the stack to
the state it was in when the `llvm.stacksave` intrinsic executed. In
practice, this pops any `alloca <i_alloca>`{.interpreted-text
role="ref"} blocks from the stack that were allocated after the
`llvm.stacksave` was executed.

#### \'`llvm.stackrestore`\' Intrinsic {#int_stackrestore}

##### Syntax:

    declare void @llvm.stackrestore(i8* %ptr)

##### Overview:

The \'`llvm.stackrestore`\' intrinsic is used to restore the state of
the function stack to the state it was in when the corresponding
`llvm.stacksave <int_stacksave>`{.interpreted-text role="ref"} intrinsic
executed. This is useful for implementing language features like scoped
automatic variable sized arrays in C99.

##### Semantics:

See the description for
`llvm.stacksave <int_stacksave>`{.interpreted-text role="ref"}.

#### \'`llvm.get.dynamic.area.offset`\' Intrinsic {#int_get_dynamic_area_offset}

##### Syntax:

    declare i32 @llvm.get.dynamic.area.offset.i32()
    declare i64 @llvm.get.dynamic.area.offset.i64()

##### Overview:

> The \'`llvm.get.dynamic.area.offset.*`\' intrinsic family is used to
> get the offset from native stack pointer to the address of the most
> recent dynamic alloca on the caller\'s stack. These intrinsics are
> intendend for use in combination with
> `llvm.stacksave <int_stacksave>`{.interpreted-text role="ref"} to get
> a pointer to the most recent dynamic alloca. This is useful, for
> example, for AddressSanitizer\'s stack unpoisoning routines.

##### Semantics:

> These intrinsics return a non-negative integer value that can be used
> to get the address of the most recent dynamic alloca, allocated by
> `alloca <i_alloca>`{.interpreted-text role="ref"} on the caller\'s
> stack. In particular, for targets where stack grows downwards, adding
> this offset to the native stack pointer would get the address of the
> most recent dynamic alloca. For targets where stack grows upwards, the
> situation is a bit more complicated, because subtracting this value
> from stack pointer would get the address one past the end of the most
> recent dynamic alloca.
>
> Although for most targets [llvm.get.dynamic.area.offset
> \<int\_get\_dynamic\_area\_offset\>]{.title-ref} returns just a zero,
> for others, such as PowerPC and PowerPC64, it returns a
> compile-time-known constant value.
>
> The return value type of
> `llvm.get.dynamic.area.offset <int_get_dynamic_area_offset>`{.interpreted-text
> role="ref"} must match the target\'s default address space\'s (address
> space 0) pointer type.

#### \'`llvm.prefetch`\' Intrinsic

##### Syntax:

    declare void @llvm.prefetch(i8* <address>, i32 <rw>, i32 <locality>, i32 <cache type>)

##### Overview:

The \'`llvm.prefetch`\' intrinsic is a hint to the code generator to
insert a prefetch instruction if supported; otherwise, it is a noop.
Prefetches have no effect on the behavior of the program but can change
its performance characteristics.

##### Arguments:

`address` is the address to be prefetched, `rw` is the specifier
determining if the fetch should be for a read (0) or write (1), and
`locality` is a temporal locality specifier ranging from (0) - no
locality, to (3) - extremely local keep in cache. The `cache type`
specifies whether the prefetch is performed on the data (1) or
instruction (0) cache. The `rw`, `locality` and `cache type` arguments
must be constant integers.

##### Semantics:

This intrinsic does not modify the behavior of the program. In
particular, prefetches cannot trap and do not produce a value. On
targets that support this intrinsic, the prefetch can provide hints to
the processor cache for better performance.

#### \'`llvm.pcmarker`\' Intrinsic

##### Syntax:

    declare void @llvm.pcmarker(i32 <id>)

##### Overview:

The \'`llvm.pcmarker`\' intrinsic is a method to export a Program
Counter (PC) in a region of code to simulators and other tools. The
method is target specific, but it is expected that the marker will use
exported symbols to transmit the PC of the marker. The marker makes no
guarantees that it will remain with any specific instruction after
optimizations. It is possible that the presence of a marker will inhibit
optimizations. The intended use is to be inserted after optimizations to
allow correlations of simulation runs.

##### Arguments:

`id` is a numerical id identifying the marker.

##### Semantics:

This intrinsic does not modify the behavior of the program. Backends
that do not support this intrinsic may ignore it.

#### \'`llvm.readcyclecounter`\' Intrinsic

##### Syntax:

    declare i64 @llvm.readcyclecounter()

##### Overview:

The \'`llvm.readcyclecounter`\' intrinsic provides access to the cycle
counter register (or similar low latency, high accuracy clocks) on those
targets that support it. On X86, it should map to RDTSC. On Alpha, it
should map to RPCC. As the backing counters overflow quickly (on the
order of 9 seconds on alpha), this should only be used for small
timings.

##### Semantics:

When directly supported, reading the cycle counter should not modify any
memory. Implementations are allowed to either return a application
specific value or a system wide value. On backends without support, this
is lowered to a constant 0.

Note that runtime support may be conditional on the privilege-level code
is running at and the host platform.

#### \'`llvm.clear_cache`\' Intrinsic

##### Syntax:

    declare void @llvm.clear_cache(i8*, i8*)

##### Overview:

The \'`llvm.clear_cache`\' intrinsic ensures visibility of modifications
in the specified range to the execution unit of the processor. On
targets with non-unified instruction and data cache, the implementation
flushes the instruction cache.

##### Semantics:

On platforms with coherent instruction and data caches (e.g. x86), this
intrinsic is a nop. On platforms with non-coherent instruction and data
cache (e.g. ARM, MIPS), the intrinsic is lowered either to appropriate
instructions or a system call, if cache flushing requires special
privileges.

The default behavior is to emit a call to `__clear_cache` from the run
time library.

This instrinsic does *not* empty the instruction pipeline. Modifications
of the current function are outside the scope of the intrinsic.

#### \'`llvm.instrprof.increment`\' Intrinsic

##### Syntax:

    declare void @llvm.instrprof.increment(i8* <name>, i64 <hash>,
                                           i32 <num-counters>, i32 <index>)

##### Overview:

The \'`llvm.instrprof.increment`\' intrinsic can be emitted by a
frontend for use with instrumentation based profiling. These will be
lowered by the `-instrprof` pass to generate execution counts of a
program at runtime.

##### Arguments:

The first argument is a pointer to a global variable containing the name
of the entity being instrumented. This should generally be the (mangled)
function name for a set of counters.

The second argument is a hash value that can be used by the consumer of
the profile data to detect changes to the instrumented source, and the
third is the number of counters associated with `name`. It is an error
if `hash` or `num-counters` differ between two instances of
`instrprof.increment` that refer to the same name.

The last argument refers to which of the counters for `name` should be
incremented. It should be a value between 0 and `num-counters`.

##### Semantics:

This intrinsic represents an increment of a profiling counter. It will
cause the `-instrprof` pass to generate the appropriate data structures
and the code to increment the appropriate value, in a format that can be
written out by a compiler runtime and consumed via the `llvm-profdata`
tool.

#### \'`llvm.instrprof.increment.step`\' Intrinsic

##### Syntax:

    declare void @llvm.instrprof.increment.step(i8* <name>, i64 <hash>,
                                                i32 <num-counters>,
                                                i32 <index>, i64 <step>)

##### Overview:

The \'`llvm.instrprof.increment.step`\' intrinsic is an extension to the
\'`llvm.instrprof.increment`\' intrinsic with an additional fifth
argument to specify the step of the increment.

##### Arguments:

The first four arguments are the same as \'`llvm.instrprof.increment`\'
intrinsic.

The last argument specifies the value of the increment of the counter
variable.

##### Semantics:

See description of \'`llvm.instrprof.increment`\' instrinsic.

#### \'`llvm.instrprof.value.profile`\' Intrinsic

##### Syntax:

    declare void @llvm.instrprof.value.profile(i8* <name>, i64 <hash>,
                                               i64 <value>, i32 <value_kind>,
                                               i32 <index>)

##### Overview:

The \'`llvm.instrprof.value.profile`\' intrinsic can be emitted by a
frontend for use with instrumentation based profiling. This will be
lowered by the `-instrprof` pass to find out the target values,
instrumented expressions take in a program at runtime.

##### Arguments:

The first argument is a pointer to a global variable containing the name
of the entity being instrumented. `name` should generally be the
(mangled) function name for a set of counters.

The second argument is a hash value that can be used by the consumer of
the profile data to detect changes to the instrumented source. It is an
error if `hash` differs between two instances of `llvm.instrprof.*` that
refer to the same name.

The third argument is the value of the expression being profiled. The
profiled expression\'s value should be representable as an unsigned
64-bit value. The fourth argument represents the kind of value profiling
that is being done. The supported value profiling kinds are enumerated
through the `InstrProfValueKind` type declared in the
`<include/llvm/ProfileData/InstrProf.h>` header file. The last argument
is the index of the instrumented expression within `name`. It should be
\>= 0.

##### Semantics:

This intrinsic represents the point where a call to a runtime routine
should be inserted for value profiling of target expressions.
`-instrprof` pass will generate the appropriate data structures and
replace the `llvm.instrprof.value.profile` intrinsic with the call to
the profile runtime library with proper arguments.

#### \'`llvm.thread.pointer`\' Intrinsic

##### Syntax:

    declare i8* @llvm.thread.pointer()

##### Overview:

The \'`llvm.thread.pointer`\' intrinsic returns the value of the thread
pointer.

##### Semantics:

The \'`llvm.thread.pointer`\' intrinsic returns a pointer to the TLS
area for the current thread. The exact semantics of this value are
target specific: it may point to the start of TLS area, to the end, or
somewhere in the middle. Depending on the target, this intrinsic may
read a register, call a helper function, read from an alternate memory
space, or perform other operations necessary to locate the TLS area. Not
all targets support this intrinsic.

### Standard C Library Intrinsics

LLVM provides intrinsics for a few important standard C library
functions. These intrinsics allow source-language front-ends to pass
information about the alignment of the pointer arguments to the code
generator, providing opportunity for more efficient code generation.

#### \'`llvm.memcpy`\' Intrinsic {#int_memcpy}

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.memcpy` on any
integer bit width and for different address spaces. Not all targets
support all bit widths however.

    declare void @llvm.memcpy.p0i8.p0i8.i32(i8* <dest>, i8* <src>,
                                            i32 <len>, i1 <isvolatile>)
    declare void @llvm.memcpy.p0i8.p0i8.i64(i8* <dest>, i8* <src>,
                                            i64 <len>, i1 <isvolatile>)

##### Overview:

The \'`llvm.memcpy.*`\' intrinsics copy a block of memory from the
source location to the destination location.

Note that, unlike the standard libc function, the `llvm.memcpy.*`
intrinsics do not return a value, takes extra isvolatile arguments and
the pointers can be in specified address spaces.

##### Arguments:

The first argument is a pointer to the destination, the second is a
pointer to the source. The third argument is an integer argument
specifying the number of bytes to copy, and the fourth is a boolean
indicating a volatile access.

The `align <attr_align>`{.interpreted-text role="ref"} parameter
attribute can be provided for the first and second arguments.

If the `isvolatile` parameter is `true`, the `llvm.memcpy` call is a
`volatile operation <volatile>`{.interpreted-text role="ref"}. The
detailed access behavior is not very cleanly specified and it is unwise
to depend on it.

##### Semantics:

The \'`llvm.memcpy.*`\' intrinsics copy a block of memory from the
source location to the destination location, which are not allowed to
overlap. It copies \"len\" bytes of memory over. If the argument is
known to be aligned to some boundary, this can be specified as the
fourth argument, otherwise it should be set to 0 or 1 (both meaning no
alignment).

#### \'`llvm.memmove`\' Intrinsic {#int_memmove}

##### Syntax:

This is an overloaded intrinsic. You can use llvm.memmove on any integer
bit width and for different address space. Not all targets support all
bit widths however.

    declare void @llvm.memmove.p0i8.p0i8.i32(i8* <dest>, i8* <src>,
                                             i32 <len>, i1 <isvolatile>)
    declare void @llvm.memmove.p0i8.p0i8.i64(i8* <dest>, i8* <src>,
                                             i64 <len>, i1 <isvolatile>)

##### Overview:

The \'`llvm.memmove.*`\' intrinsics move a block of memory from the
source location to the destination location. It is similar to the
\'`llvm.memcpy`\' intrinsic but allows the two memory locations to
overlap.

Note that, unlike the standard libc function, the `llvm.memmove.*`
intrinsics do not return a value, takes an extra isvolatile argument and
the pointers can be in specified address spaces.

##### Arguments:

The first argument is a pointer to the destination, the second is a
pointer to the source. The third argument is an integer argument
specifying the number of bytes to copy, and the fourth is a boolean
indicating a volatile access.

The `align <attr_align>`{.interpreted-text role="ref"} parameter
attribute can be provided for the first and second arguments.

If the `isvolatile` parameter is `true`, the `llvm.memmove` call is a
`volatile operation <volatile>`{.interpreted-text role="ref"}. The
detailed access behavior is not very cleanly specified and it is unwise
to depend on it.

##### Semantics:

The \'`llvm.memmove.*`\' intrinsics copy a block of memory from the
source location to the destination location, which may overlap. It
copies \"len\" bytes of memory over. If the argument is known to be
aligned to some boundary, this can be specified as the fourth argument,
otherwise it should be set to 0 or 1 (both meaning no alignment).

#### \'`llvm.memset.*`\' Intrinsics {#int_memset}

##### Syntax:

This is an overloaded intrinsic. You can use llvm.memset on any integer
bit width and for different address spaces. However, not all targets
support all bit widths.

    declare void @llvm.memset.p0i8.i32(i8* <dest>, i8 <val>,
                                       i32 <len>, i1 <isvolatile>)
    declare void @llvm.memset.p0i8.i64(i8* <dest>, i8 <val>,
                                       i64 <len>, i1 <isvolatile>)

##### Overview:

The \'`llvm.memset.*`\' intrinsics fill a block of memory with a
particular byte value.

Note that, unlike the standard libc function, the `llvm.memset`
intrinsic does not return a value and takes an extra volatile argument.
Also, the destination can be in an arbitrary address space.

##### Arguments:

The first argument is a pointer to the destination to fill, the second
is the byte value with which to fill it, the third argument is an
integer argument specifying the number of bytes to fill, and the fourth
is a boolean indicating a volatile access.

The `align <attr_align>`{.interpreted-text role="ref"} parameter
attribute can be provided for the first arguments.

If the `isvolatile` parameter is `true`, the `llvm.memset` call is a
`volatile operation <volatile>`{.interpreted-text role="ref"}. The
detailed access behavior is not very cleanly specified and it is unwise
to depend on it.

##### Semantics:

The \'`llvm.memset.*`\' intrinsics fill \"len\" bytes of memory starting
at the destination location.

#### \'`llvm.sqrt.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.sqrt` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.sqrt.f32(float %Val)
    declare double    @llvm.sqrt.f64(double %Val)
    declare x86_fp80  @llvm.sqrt.f80(x86_fp80 %Val)
    declare fp128     @llvm.sqrt.f128(fp128 %Val)
    declare ppc_fp128 @llvm.sqrt.ppcf128(ppc_fp128 %Val)

##### Overview:

The \'`llvm.sqrt`\' intrinsics return the square root of the specified
value.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`sqrt`\' function but
without trapping or setting `errno`. For types specified by IEEE-754,
the result matches a conforming libm implementation.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.powi.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.powi` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.powi.f32(float  %Val, i32 %power)
    declare double    @llvm.powi.f64(double %Val, i32 %power)
    declare x86_fp80  @llvm.powi.f80(x86_fp80  %Val, i32 %power)
    declare fp128     @llvm.powi.f128(fp128 %Val, i32 %power)
    declare ppc_fp128 @llvm.powi.ppcf128(ppc_fp128  %Val, i32 %power)

##### Overview:

The \'`llvm.powi.*`\' intrinsics return the first operand raised to the
specified (positive or negative) power. The order of evaluation of
multiplications is not defined. When a vector of floating-point type is
used, the second argument remains a scalar integer value.

##### Arguments:

The second argument is an integer power, and the first is a value to
raise to that power.

##### Semantics:

This function returns the first value raised to the second power with an
unspecified sequence of rounding operations.

#### \'`llvm.sin.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.sin` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.sin.f32(float  %Val)
    declare double    @llvm.sin.f64(double %Val)
    declare x86_fp80  @llvm.sin.f80(x86_fp80  %Val)
    declare fp128     @llvm.sin.f128(fp128 %Val)
    declare ppc_fp128 @llvm.sin.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.sin.*`\' intrinsics return the sine of the operand.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`sin`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.cos.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.cos` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.cos.f32(float  %Val)
    declare double    @llvm.cos.f64(double %Val)
    declare x86_fp80  @llvm.cos.f80(x86_fp80  %Val)
    declare fp128     @llvm.cos.f128(fp128 %Val)
    declare ppc_fp128 @llvm.cos.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.cos.*`\' intrinsics return the cosine of the operand.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`cos`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.pow.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.pow` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.pow.f32(float  %Val, float %Power)
    declare double    @llvm.pow.f64(double %Val, double %Power)
    declare x86_fp80  @llvm.pow.f80(x86_fp80  %Val, x86_fp80 %Power)
    declare fp128     @llvm.pow.f128(fp128 %Val, fp128 %Power)
    declare ppc_fp128 @llvm.pow.ppcf128(ppc_fp128  %Val, ppc_fp128 Power)

##### Overview:

The \'`llvm.pow.*`\' intrinsics return the first operand raised to the
specified (positive or negative) power.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`pow`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.exp.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.exp` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.exp.f32(float  %Val)
    declare double    @llvm.exp.f64(double %Val)
    declare x86_fp80  @llvm.exp.f80(x86_fp80  %Val)
    declare fp128     @llvm.exp.f128(fp128 %Val)
    declare ppc_fp128 @llvm.exp.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.exp.*`\' intrinsics compute the base-e exponential of the
specified value.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`exp`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.exp2.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.exp2` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.exp2.f32(float  %Val)
    declare double    @llvm.exp2.f64(double %Val)
    declare x86_fp80  @llvm.exp2.f80(x86_fp80  %Val)
    declare fp128     @llvm.exp2.f128(fp128 %Val)
    declare ppc_fp128 @llvm.exp2.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.exp2.*`\' intrinsics compute the base-2 exponential of the
specified value.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`exp2`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.log.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.log` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.log.f32(float  %Val)
    declare double    @llvm.log.f64(double %Val)
    declare x86_fp80  @llvm.log.f80(x86_fp80  %Val)
    declare fp128     @llvm.log.f128(fp128 %Val)
    declare ppc_fp128 @llvm.log.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.log.*`\' intrinsics compute the base-e logarithm of the
specified value.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`log`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.log10.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.log10` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.log10.f32(float  %Val)
    declare double    @llvm.log10.f64(double %Val)
    declare x86_fp80  @llvm.log10.f80(x86_fp80  %Val)
    declare fp128     @llvm.log10.f128(fp128 %Val)
    declare ppc_fp128 @llvm.log10.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.log10.*`\' intrinsics compute the base-10 logarithm of the
specified value.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`log10`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.log2.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.log2` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.log2.f32(float  %Val)
    declare double    @llvm.log2.f64(double %Val)
    declare x86_fp80  @llvm.log2.f80(x86_fp80  %Val)
    declare fp128     @llvm.log2.f128(fp128 %Val)
    declare ppc_fp128 @llvm.log2.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.log2.*`\' intrinsics compute the base-2 logarithm of the
specified value.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`log2`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.fma.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.fma` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.fma.f32(float  %a, float  %b, float  %c)
    declare double    @llvm.fma.f64(double %a, double %b, double %c)
    declare x86_fp80  @llvm.fma.f80(x86_fp80 %a, x86_fp80 %b, x86_fp80 %c)
    declare fp128     @llvm.fma.f128(fp128 %a, fp128 %b, fp128 %c)
    declare ppc_fp128 @llvm.fma.ppcf128(ppc_fp128 %a, ppc_fp128 %b, ppc_fp128 %c)

##### Overview:

The \'`llvm.fma.*`\' intrinsics perform the fused multiply-add
operation.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

Return the same value as a corresponding libm \'`fma`\' function but
without trapping or setting `errno`.

When specified with the fast-math-flag \'afn\', the result may be
approximated using a less accurate calculation.

#### \'`llvm.fabs.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.fabs` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.fabs.f32(float  %Val)
    declare double    @llvm.fabs.f64(double %Val)
    declare x86_fp80  @llvm.fabs.f80(x86_fp80 %Val)
    declare fp128     @llvm.fabs.f128(fp128 %Val)
    declare ppc_fp128 @llvm.fabs.ppcf128(ppc_fp128 %Val)

##### Overview:

The \'`llvm.fabs.*`\' intrinsics return the absolute value of the
operand.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `fabs` functions
would, and handles error conditions in the same way.

#### \'`llvm.minnum.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.minnum` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.minnum.f32(float %Val0, float %Val1)
    declare double    @llvm.minnum.f64(double %Val0, double %Val1)
    declare x86_fp80  @llvm.minnum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
    declare fp128     @llvm.minnum.f128(fp128 %Val0, fp128 %Val1)
    declare ppc_fp128 @llvm.minnum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

##### Overview:

The \'`llvm.minnum.*`\' intrinsics return the minimum of the two
arguments.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

Follows the IEEE-754 semantics for minNum, except for handling of
signaling NaNs. This match\'s the behavior of libm\'s fmin.

If either operand is a NaN, returns the other non-NaN operand. Returns
NaN only if both operands are NaN. The returned NaN is always quiet. If
the operands compare equal, returns a value that compares equal to both
operands. This means that fmin(+/-0.0, +/-0.0) could return either -0.0
or 0.0.

Unlike the IEEE-754 2008 behavior, this does not distinguish between
signaling and quiet NaN inputs. If a target\'s implementation follows
the standard and returns a quiet NaN if either input is a signaling NaN,
the intrinsic lowering is responsible for quieting the inputs to
correctly return the non-NaN input (e.g. by using the equivalent of
`llvm.canonicalize`).

#### \'`llvm.maxnum.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.maxnum` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.maxnum.f32(float  %Val0, float  %Val1l)
    declare double    @llvm.maxnum.f64(double %Val0, double %Val1)
    declare x86_fp80  @llvm.maxnum.f80(x86_fp80  %Val0, x86_fp80  %Val1)
    declare fp128     @llvm.maxnum.f128(fp128 %Val0, fp128 %Val1)
    declare ppc_fp128 @llvm.maxnum.ppcf128(ppc_fp128  %Val0, ppc_fp128  %Val1)

##### Overview:

The \'`llvm.maxnum.*`\' intrinsics return the maximum of the two
arguments.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

Follows the IEEE-754 semantics for maxNum except for the handling of
signaling NaNs. This matches the behavior of libm\'s fmax.

If either operand is a NaN, returns the other non-NaN operand. Returns
NaN only if both operands are NaN. The returned NaN is always quiet. If
the operands compare equal, returns a value that compares equal to both
operands. This means that fmax(+/-0.0, +/-0.0) could return either -0.0
or 0.0.

Unlike the IEEE-754 2008 behavior, this does not distinguish between
signaling and quiet NaN inputs. If a target\'s implementation follows
the standard and returns a quiet NaN if either input is a signaling NaN,
the intrinsic lowering is responsible for quieting the inputs to
correctly return the non-NaN input (e.g. by using the equivalent of
`llvm.canonicalize`).

#### \'`llvm.minimum.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.minimum` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.minimum.f32(float %Val0, float %Val1)
    declare double    @llvm.minimum.f64(double %Val0, double %Val1)
    declare x86_fp80  @llvm.minimum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
    declare fp128     @llvm.minimum.f128(fp128 %Val0, fp128 %Val1)
    declare ppc_fp128 @llvm.minimum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

##### Overview:

The \'`llvm.minimum.*`\' intrinsics return the minimum of the two
arguments, propagating NaNs and treating -0.0 as less than +0.0.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

If either operand is a NaN, returns NaN. Otherwise returns the lesser of
the two arguments. -0.0 is considered to be less than +0.0 for this
intrinsic. Note that these are the semantics specified in the draft of
IEEE 754-2018.

#### \'`llvm.maximum.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.maximum` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.maximum.f32(float %Val0, float %Val1)
    declare double    @llvm.maximum.f64(double %Val0, double %Val1)
    declare x86_fp80  @llvm.maximum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
    declare fp128     @llvm.maximum.f128(fp128 %Val0, fp128 %Val1)
    declare ppc_fp128 @llvm.maximum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

##### Overview:

The \'`llvm.maximum.*`\' intrinsics return the maximum of the two
arguments, propagating NaNs and treating -0.0 as less than +0.0.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

If either operand is a NaN, returns NaN. Otherwise returns the greater
of the two arguments. -0.0 is considered to be less than +0.0 for this
intrinsic. Note that these are the semantics specified in the draft of
IEEE 754-2018.

#### \'`llvm.copysign.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.copysign` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.copysign.f32(float  %Mag, float  %Sgn)
    declare double    @llvm.copysign.f64(double %Mag, double %Sgn)
    declare x86_fp80  @llvm.copysign.f80(x86_fp80  %Mag, x86_fp80  %Sgn)
    declare fp128     @llvm.copysign.f128(fp128 %Mag, fp128 %Sgn)
    declare ppc_fp128 @llvm.copysign.ppcf128(ppc_fp128  %Mag, ppc_fp128  %Sgn)

##### Overview:

The \'`llvm.copysign.*`\' intrinsics return a value with the magnitude
of the first operand and the sign of the second operand.

##### Arguments:

The arguments and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `copysign` functions
would, and handles error conditions in the same way.

#### \'`llvm.floor.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.floor` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.floor.f32(float  %Val)
    declare double    @llvm.floor.f64(double %Val)
    declare x86_fp80  @llvm.floor.f80(x86_fp80  %Val)
    declare fp128     @llvm.floor.f128(fp128 %Val)
    declare ppc_fp128 @llvm.floor.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.floor.*`\' intrinsics return the floor of the operand.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `floor` functions
would, and handles error conditions in the same way.

#### \'`llvm.ceil.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.ceil` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.ceil.f32(float  %Val)
    declare double    @llvm.ceil.f64(double %Val)
    declare x86_fp80  @llvm.ceil.f80(x86_fp80  %Val)
    declare fp128     @llvm.ceil.f128(fp128 %Val)
    declare ppc_fp128 @llvm.ceil.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.ceil.*`\' intrinsics return the ceiling of the operand.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `ceil` functions
would, and handles error conditions in the same way.

#### \'`llvm.trunc.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.trunc` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.trunc.f32(float  %Val)
    declare double    @llvm.trunc.f64(double %Val)
    declare x86_fp80  @llvm.trunc.f80(x86_fp80  %Val)
    declare fp128     @llvm.trunc.f128(fp128 %Val)
    declare ppc_fp128 @llvm.trunc.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.trunc.*`\' intrinsics returns the operand rounded to the
nearest integer not larger in magnitude than the operand.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `trunc` functions
would, and handles error conditions in the same way.

#### \'`llvm.rint.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.rint` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.rint.f32(float  %Val)
    declare double    @llvm.rint.f64(double %Val)
    declare x86_fp80  @llvm.rint.f80(x86_fp80  %Val)
    declare fp128     @llvm.rint.f128(fp128 %Val)
    declare ppc_fp128 @llvm.rint.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.rint.*`\' intrinsics returns the operand rounded to the
nearest integer. It may raise an inexact floating-point exception if the
operand isn\'t an integer.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `rint` functions
would, and handles error conditions in the same way.

#### \'`llvm.nearbyint.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.nearbyint` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.nearbyint.f32(float  %Val)
    declare double    @llvm.nearbyint.f64(double %Val)
    declare x86_fp80  @llvm.nearbyint.f80(x86_fp80  %Val)
    declare fp128     @llvm.nearbyint.f128(fp128 %Val)
    declare ppc_fp128 @llvm.nearbyint.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.nearbyint.*`\' intrinsics returns the operand rounded to the
nearest integer.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `nearbyint` functions
would, and handles error conditions in the same way.

#### \'`llvm.round.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.round` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

    declare float     @llvm.round.f32(float  %Val)
    declare double    @llvm.round.f64(double %Val)
    declare x86_fp80  @llvm.round.f80(x86_fp80  %Val)
    declare fp128     @llvm.round.f128(fp128 %Val)
    declare ppc_fp128 @llvm.round.ppcf128(ppc_fp128  %Val)

##### Overview:

The \'`llvm.round.*`\' intrinsics returns the operand rounded to the
nearest integer.

##### Arguments:

The argument and return value are floating-point numbers of the same
type.

##### Semantics:

This function returns the same values as the libm `round` functions
would, and handles error conditions in the same way.

### Bit Manipulation Intrinsics

LLVM provides intrinsics for a few important bit manipulation
operations. These allow efficient code generation for some algorithms.

#### \'`llvm.bitreverse.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic function. You can use bitreverse on any
integer type.

    declare i16 @llvm.bitreverse.i16(i16 <id>)
    declare i32 @llvm.bitreverse.i32(i32 <id>)
    declare i64 @llvm.bitreverse.i64(i64 <id>)

##### Overview:

The \'`llvm.bitreverse`\' family of intrinsics is used to reverse the
bitpattern of an integer value; for example `0b10110110` becomes
`0b01101101`.

##### Semantics:

The `llvm.bitreverse.iN` intrinsic returns an iN value that has bit `M`
in the input moved to bit `N-M` in the output.

#### \'`llvm.bswap.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic function. You can use bswap on any
integer type that is an even number of bytes (i.e. BitWidth % 16 == 0).

    declare i16 @llvm.bswap.i16(i16 <id>)
    declare i32 @llvm.bswap.i32(i32 <id>)
    declare i64 @llvm.bswap.i64(i64 <id>)

##### Overview:

The \'`llvm.bswap`\' family of intrinsics is used to byte swap integer
values with an even number of bytes (positive multiple of 16 bits).
These are useful for performing operations on data that is not in the
target\'s native byte order.

##### Semantics:

The `llvm.bswap.i16` intrinsic returns an i16 value that has the high
and low byte of the input i16 swapped. Similarly, the `llvm.bswap.i32`
intrinsic returns an i32 value that has the four bytes of the input i32
swapped, so that if the input bytes are numbered 0, 1, 2, 3 then the
returned i32 will have its bytes in 3, 2, 1, 0 order. The
`llvm.bswap.i48`, `llvm.bswap.i64` and other intrinsics extend this
concept to additional even-byte lengths (6 bytes, 8 bytes and more,
respectively).

#### \'`llvm.ctpop.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use llvm.ctpop on any integer
bit width, or on any vector with integer elements. Not all targets
support all bit widths or vector types, however.

    declare i8 @llvm.ctpop.i8(i8  <src>)
    declare i16 @llvm.ctpop.i16(i16 <src>)
    declare i32 @llvm.ctpop.i32(i32 <src>)
    declare i64 @llvm.ctpop.i64(i64 <src>)
    declare i256 @llvm.ctpop.i256(i256 <src>)
    declare <2 x i32> @llvm.ctpop.v2i32(<2 x i32> <src>)

##### Overview:

The \'`llvm.ctpop`\' family of intrinsics counts the number of bits set
in a value.

##### Arguments:

The only argument is the value to be counted. The argument may be of any
integer type, or a vector with integer elements. The return type must
match the argument type.

##### Semantics:

The \'`llvm.ctpop`\' intrinsic counts the 1\'s in a variable, or within
each element of a vector.

#### \'`llvm.ctlz.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.ctlz` on any integer
bit width, or any vector whose elements are integers. Not all targets
support all bit widths or vector types, however.

    declare i8   @llvm.ctlz.i8  (i8   <src>, i1 <is_zero_undef>)
    declare i16  @llvm.ctlz.i16 (i16  <src>, i1 <is_zero_undef>)
    declare i32  @llvm.ctlz.i32 (i32  <src>, i1 <is_zero_undef>)
    declare i64  @llvm.ctlz.i64 (i64  <src>, i1 <is_zero_undef>)
    declare i256 @llvm.ctlz.i256(i256 <src>, i1 <is_zero_undef>)
    declare <2 x i32> @llvm.ctlz.v2i32(<2 x i32> <src>, i1 <is_zero_undef>)

##### Overview:

The \'`llvm.ctlz`\' family of intrinsic functions counts the number of
leading zeros in a variable.

##### Arguments:

The first argument is the value to be counted. This argument may be of
any integer type, or a vector with integer element type. The return type
must match the first argument type.

The second argument must be a constant and is a flag to indicate whether
the intrinsic should ensure that a zero as the first argument produces a
defined result. Historically some architectures did not provide a
defined result for zero values as efficiently, and many algorithms are
now predicated on avoiding zero-value inputs.

##### Semantics:

The \'`llvm.ctlz`\' intrinsic counts the leading (most significant)
zeros in a variable, or within each element of the vector. If `src == 0`
then the result is the size in bits of the type of `src` if
`is_zero_undef == 0` and `undef` otherwise. For example,
`llvm.ctlz(i32 2) = 30`.

#### \'`llvm.cttz.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.cttz` on any integer
bit width, or any vector of integer elements. Not all targets support
all bit widths or vector types, however.

    declare i8   @llvm.cttz.i8  (i8   <src>, i1 <is_zero_undef>)
    declare i16  @llvm.cttz.i16 (i16  <src>, i1 <is_zero_undef>)
    declare i32  @llvm.cttz.i32 (i32  <src>, i1 <is_zero_undef>)
    declare i64  @llvm.cttz.i64 (i64  <src>, i1 <is_zero_undef>)
    declare i256 @llvm.cttz.i256(i256 <src>, i1 <is_zero_undef>)
    declare <2 x i32> @llvm.cttz.v2i32(<2 x i32> <src>, i1 <is_zero_undef>)

##### Overview:

The \'`llvm.cttz`\' family of intrinsic functions counts the number of
trailing zeros.

##### Arguments:

The first argument is the value to be counted. This argument may be of
any integer type, or a vector with integer element type. The return type
must match the first argument type.

The second argument must be a constant and is a flag to indicate whether
the intrinsic should ensure that a zero as the first argument produces a
defined result. Historically some architectures did not provide a
defined result for zero values as efficiently, and many algorithms are
now predicated on avoiding zero-value inputs.

##### Semantics:

The \'`llvm.cttz`\' intrinsic counts the trailing (least significant)
zeros in a variable, or within each element of a vector. If `src == 0`
then the result is the size in bits of the type of `src` if
`is_zero_undef == 0` and `undef` otherwise. For example,
`llvm.cttz(2) = 1`.

#### \'`llvm.fshl.*`\' Intrinsic {#int_overflow}

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.fshl` on any integer
bit width or any vector of integer elements. Not all targets support all
bit widths or vector types, however.

    declare i8  @llvm.fshl.i8 (i8 %a, i8 %b, i8 %c)
    declare i67 @llvm.fshl.i67(i67 %a, i67 %b, i67 %c)
    declare <2 x i32> @llvm.fshl.v2i32(<2 x i32> %a, <2 x i32> %b, <2 x i32> %c)

##### Overview:

The \'`llvm.fshl`\' family of intrinsic functions performs a funnel
shift left: the first two values are concatenated as { %a : %b } (%a is
the most significant bits of the wide value), the combined value is
shifted left, and the most significant bits are extracted to produce a
result that is the same size as the original arguments. If the first 2
arguments are identical, this is equivalent to a rotate left operation.
For vector types, the operation occurs for each element of the vector.
The shift argument is treated as an unsigned amount modulo the element
size of the arguments.

##### Arguments:

The first two arguments are the values to be concatenated. The third
argument is the shift amount. The arguments may be any integer type or a
vector with integer element type. All arguments and the return value
must have the same type.

##### Example:

``` {.text}
%r = call i8 @llvm.fshl.i8(i8 %x, i8 %y, i8 %z)  ; %r = i8: msb_extract((concat(x, y) << (z % 8)), 8)
%r = call i8 @llvm.fshl.i8(i8 255, i8 0, i8 15)  ; %r = i8: 128 (0b10000000)
%r = call i8 @llvm.fshl.i8(i8 15, i8 15, i8 11)  ; %r = i8: 120 (0b01111000)
%r = call i8 @llvm.fshl.i8(i8 0, i8 255, i8 8)   ; %r = i8: 0   (0b00000000)
```

#### \'`llvm.fshr.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.fshr` on any integer
bit width or any vector of integer elements. Not all targets support all
bit widths or vector types, however.

    declare i8  @llvm.fshr.i8 (i8 %a, i8 %b, i8 %c)
    declare i67 @llvm.fshr.i67(i67 %a, i67 %b, i67 %c)
    declare <2 x i32> @llvm.fshr.v2i32(<2 x i32> %a, <2 x i32> %b, <2 x i32> %c)

##### Overview:

The \'`llvm.fshr`\' family of intrinsic functions performs a funnel
shift right: the first two values are concatenated as { %a : %b } (%a is
the most significant bits of the wide value), the combined value is
shifted right, and the least significant bits are extracted to produce a
result that is the same size as the original arguments. If the first 2
arguments are identical, this is equivalent to a rotate right operation.
For vector types, the operation occurs for each element of the vector.
The shift argument is treated as an unsigned amount modulo the element
size of the arguments.

##### Arguments:

The first two arguments are the values to be concatenated. The third
argument is the shift amount. The arguments may be any integer type or a
vector with integer element type. All arguments and the return value
must have the same type.

##### Example:

``` {.text}
%r = call i8 @llvm.fshr.i8(i8 %x, i8 %y, i8 %z)  ; %r = i8: lsb_extract((concat(x, y) >> (z % 8)), 8)
%r = call i8 @llvm.fshr.i8(i8 255, i8 0, i8 15)  ; %r = i8: 254 (0b11111110)
%r = call i8 @llvm.fshr.i8(i8 15, i8 15, i8 11)  ; %r = i8: 225 (0b11100001)
%r = call i8 @llvm.fshr.i8(i8 0, i8 255, i8 8)   ; %r = i8: 255 (0b11111111)
```

### Arithmetic with Overflow Intrinsics

LLVM provides intrinsics for fast arithmetic overflow checking.

Each of these intrinsics returns a two-element struct. The first element
of this struct contains the result of the corresponding arithmetic
operation modulo 2^n^, where n is the bit width of the result.
Therefore, for example, the first element of the struct returned by
`llvm.sadd.with.overflow.i32` is always the same as the result of a
32-bit `add` instruction with the same operands, where the `add` is
*not* modified by an `nsw` or `nuw` flag.

The second element of the result is an `i1` that is 1 if the arithmetic
operation overflowed and 0 otherwise. An operation overflows if, for any
values of its operands `A` and `B` and for any `N` larger than the
operands\' width, `ext(A op B) to iN` is not equal to
`(ext(A) to iN) op (ext(B) to iN)` where `ext` is `sext` for signed
overflow and `zext` for unsigned overflow, and `op` is the underlying
arithmetic operation.

The behavior of these intrinsics is well-defined for all argument
values.

#### \'`llvm.sadd.with.overflow.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.sadd.with.overflow`
on any integer bit width.

    declare {i16, i1} @llvm.sadd.with.overflow.i16(i16 %a, i16 %b)
    declare {i32, i1} @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
    declare {i64, i1} @llvm.sadd.with.overflow.i64(i64 %a, i64 %b)

##### Overview:

The \'`llvm.sadd.with.overflow`\' family of intrinsic functions perform
a signed addition of the two arguments, and indicate whether an overflow
occurred during the signed summation.

##### Arguments:

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
`i1`. `%a` and `%b` are the two values that will undergo signed
addition.

##### Semantics:

The \'`llvm.sadd.with.overflow`\' family of intrinsic functions perform
a signed addition of the two variables. They return a structure \-\--
the first element of which is the signed summation, and the second
element of which is a bit specifying if the signed summation resulted in
an overflow.

##### Examples:

``` {.llvm}
%res = call {i32, i1} @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
%sum = extractvalue {i32, i1} %res, 0
%obit = extractvalue {i32, i1} %res, 1
br i1 %obit, label %overflow, label %normal
```

#### \'`llvm.uadd.with.overflow.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.uadd.with.overflow`
on any integer bit width.

    declare {i16, i1} @llvm.uadd.with.overflow.i16(i16 %a, i16 %b)
    declare {i32, i1} @llvm.uadd.with.overflow.i32(i32 %a, i32 %b)
    declare {i64, i1} @llvm.uadd.with.overflow.i64(i64 %a, i64 %b)

##### Overview:

The \'`llvm.uadd.with.overflow`\' family of intrinsic functions perform
an unsigned addition of the two arguments, and indicate whether a carry
occurred during the unsigned summation.

##### Arguments:

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
`i1`. `%a` and `%b` are the two values that will undergo unsigned
addition.

##### Semantics:

The \'`llvm.uadd.with.overflow`\' family of intrinsic functions perform
an unsigned addition of the two arguments. They return a structure \-\--
the first element of which is the sum, and the second element of which
is a bit specifying if the unsigned summation resulted in a carry.

##### Examples:

``` {.llvm}
%res = call {i32, i1} @llvm.uadd.with.overflow.i32(i32 %a, i32 %b)
%sum = extractvalue {i32, i1} %res, 0
%obit = extractvalue {i32, i1} %res, 1
br i1 %obit, label %carry, label %normal
```

#### \'`llvm.ssub.with.overflow.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.ssub.with.overflow`
on any integer bit width.

    declare {i16, i1} @llvm.ssub.with.overflow.i16(i16 %a, i16 %b)
    declare {i32, i1} @llvm.ssub.with.overflow.i32(i32 %a, i32 %b)
    declare {i64, i1} @llvm.ssub.with.overflow.i64(i64 %a, i64 %b)

##### Overview:

The \'`llvm.ssub.with.overflow`\' family of intrinsic functions perform
a signed subtraction of the two arguments, and indicate whether an
overflow occurred during the signed subtraction.

##### Arguments:

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
`i1`. `%a` and `%b` are the two values that will undergo signed
subtraction.

##### Semantics:

The \'`llvm.ssub.with.overflow`\' family of intrinsic functions perform
a signed subtraction of the two arguments. They return a structure \-\--
the first element of which is the subtraction, and the second element of
which is a bit specifying if the signed subtraction resulted in an
overflow.

##### Examples:

``` {.llvm}
%res = call {i32, i1} @llvm.ssub.with.overflow.i32(i32 %a, i32 %b)
%sum = extractvalue {i32, i1} %res, 0
%obit = extractvalue {i32, i1} %res, 1
br i1 %obit, label %overflow, label %normal
```

#### \'`llvm.usub.with.overflow.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.usub.with.overflow`
on any integer bit width.

    declare {i16, i1} @llvm.usub.with.overflow.i16(i16 %a, i16 %b)
    declare {i32, i1} @llvm.usub.with.overflow.i32(i32 %a, i32 %b)
    declare {i64, i1} @llvm.usub.with.overflow.i64(i64 %a, i64 %b)

##### Overview:

The \'`llvm.usub.with.overflow`\' family of intrinsic functions perform
an unsigned subtraction of the two arguments, and indicate whether an
overflow occurred during the unsigned subtraction.

##### Arguments:

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
`i1`. `%a` and `%b` are the two values that will undergo unsigned
subtraction.

##### Semantics:

The \'`llvm.usub.with.overflow`\' family of intrinsic functions perform
an unsigned subtraction of the two arguments. They return a structure
\-\--the first element of which is the subtraction, and the second
element of which is a bit specifying if the unsigned subtraction
resulted in an overflow.

##### Examples:

``` {.llvm}
%res = call {i32, i1} @llvm.usub.with.overflow.i32(i32 %a, i32 %b)
%sum = extractvalue {i32, i1} %res, 0
%obit = extractvalue {i32, i1} %res, 1
br i1 %obit, label %overflow, label %normal
```

#### \'`llvm.smul.with.overflow.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.smul.with.overflow`
on any integer bit width.

    declare {i16, i1} @llvm.smul.with.overflow.i16(i16 %a, i16 %b)
    declare {i32, i1} @llvm.smul.with.overflow.i32(i32 %a, i32 %b)
    declare {i64, i1} @llvm.smul.with.overflow.i64(i64 %a, i64 %b)

##### Overview:

The \'`llvm.smul.with.overflow`\' family of intrinsic functions perform
a signed multiplication of the two arguments, and indicate whether an
overflow occurred during the signed multiplication.

##### Arguments:

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
`i1`. `%a` and `%b` are the two values that will undergo signed
multiplication.

##### Semantics:

The \'`llvm.smul.with.overflow`\' family of intrinsic functions perform
a signed multiplication of the two arguments. They return a structure
\-\--the first element of which is the multiplication, and the second
element of which is a bit specifying if the signed multiplication
resulted in an overflow.

##### Examples:

``` {.llvm}
%res = call {i32, i1} @llvm.smul.with.overflow.i32(i32 %a, i32 %b)
%sum = extractvalue {i32, i1} %res, 0
%obit = extractvalue {i32, i1} %res, 1
br i1 %obit, label %overflow, label %normal
```

#### \'`llvm.umul.with.overflow.*`\' Intrinsics

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.umul.with.overflow`
on any integer bit width.

    declare {i16, i1} @llvm.umul.with.overflow.i16(i16 %a, i16 %b)
    declare {i32, i1} @llvm.umul.with.overflow.i32(i32 %a, i32 %b)
    declare {i64, i1} @llvm.umul.with.overflow.i64(i64 %a, i64 %b)

##### Overview:

The \'`llvm.umul.with.overflow`\' family of intrinsic functions perform
a unsigned multiplication of the two arguments, and indicate whether an
overflow occurred during the unsigned multiplication.

##### Arguments:

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
`i1`. `%a` and `%b` are the two values that will undergo unsigned
multiplication.

##### Semantics:

The \'`llvm.umul.with.overflow`\' family of intrinsic functions perform
an unsigned multiplication of the two arguments. They return a structure
\-\--the first element of which is the multiplication, and the second
element of which is a bit specifying if the unsigned multiplication
resulted in an overflow.

##### Examples:

``` {.llvm}
%res = call {i32, i1} @llvm.umul.with.overflow.i32(i32 %a, i32 %b)
%sum = extractvalue {i32, i1} %res, 0
%obit = extractvalue {i32, i1} %res, 1
br i1 %obit, label %overflow, label %normal
```

### Saturation Arithmetic Intrinsics

Saturation arithmetic is a version of arithmetic in which operations are
limited to a fixed range between a minimum and maximum value. If the
result of an operation is greater than the maximum value, the result is
set (or \"clamped\") to this maximum. If it is below the minimum, it is
clamped to this minimum.

#### \'`llvm.sadd.sat.*`\' Intrinsics

##### Syntax

This is an overloaded intrinsic. You can use `llvm.sadd.sat` on any
integer bit width or vectors of integers.

    declare i16 @llvm.sadd.sat.i16(i16 %a, i16 %b)
    declare i32 @llvm.sadd.sat.i32(i32 %a, i32 %b)
    declare i64 @llvm.sadd.sat.i64(i64 %a, i64 %b)
    declare <4 x i32> @llvm.sadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

##### Overview

The \'`llvm.sadd.sat`\' family of intrinsic functions perform signed
saturation addition on the 2 arguments.

##### Arguments

The arguments (%a and %b) and the result may be of integer types of any
bit width, but they must have the same bit width. `%a` and `%b` are the
two values that will undergo signed addition.

##### Semantics:

The maximum value this operation can clamp to is the largest signed
value representable by the bit width of the arguments. The minimum value
is the smallest signed value representable by this bit width.

##### Examples

``` {.llvm}
%res = call i4 @llvm.sadd.sat.i4(i4 1, i4 2)  ; %res = 3
%res = call i4 @llvm.sadd.sat.i4(i4 5, i4 6)  ; %res = 7
%res = call i4 @llvm.sadd.sat.i4(i4 -4, i4 2)  ; %res = -2
%res = call i4 @llvm.sadd.sat.i4(i4 -4, i4 -5)  ; %res = -8
```

#### \'`llvm.uadd.sat.*`\' Intrinsics

##### Syntax

This is an overloaded intrinsic. You can use `llvm.uadd.sat` on any
integer bit width or vectors of integers.

    declare i16 @llvm.uadd.sat.i16(i16 %a, i16 %b)
    declare i32 @llvm.uadd.sat.i32(i32 %a, i32 %b)
    declare i64 @llvm.uadd.sat.i64(i64 %a, i64 %b)
    declare <4 x i32> @llvm.uadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

##### Overview

The \'`llvm.uadd.sat`\' family of intrinsic functions perform unsigned
saturation addition on the 2 arguments.

##### Arguments

The arguments (%a and %b) and the result may be of integer types of any
bit width, but they must have the same bit width. `%a` and `%b` are the
two values that will undergo unsigned addition.

##### Semantics:

The maximum value this operation can clamp to is the largest unsigned
value representable by the bit width of the arguments. Because this is
an unsigned operation, the result will never saturate towards zero.

##### Examples

``` {.llvm}
%res = call i4 @llvm.uadd.sat.i4(i4 1, i4 2)  ; %res = 3
%res = call i4 @llvm.uadd.sat.i4(i4 5, i4 6)  ; %res = 11
%res = call i4 @llvm.uadd.sat.i4(i4 8, i4 8)  ; %res = 15
```

#### \'`llvm.ssub.sat.*`\' Intrinsics

##### Syntax

This is an overloaded intrinsic. You can use `llvm.ssub.sat` on any
integer bit width or vectors of integers.

    declare i16 @llvm.ssub.sat.i16(i16 %a, i16 %b)
    declare i32 @llvm.ssub.sat.i32(i32 %a, i32 %b)
    declare i64 @llvm.ssub.sat.i64(i64 %a, i64 %b)
    declare <4 x i32> @llvm.ssub.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

##### Overview

The \'`llvm.ssub.sat`\' family of intrinsic functions perform signed
saturation subtraction on the 2 arguments.

##### Arguments

The arguments (%a and %b) and the result may be of integer types of any
bit width, but they must have the same bit width. `%a` and `%b` are the
two values that will undergo signed subtraction.

##### Semantics:

The maximum value this operation can clamp to is the largest signed
value representable by the bit width of the arguments. The minimum value
is the smallest signed value representable by this bit width.

##### Examples

``` {.llvm}
%res = call i4 @llvm.ssub.sat.i4(i4 2, i4 1)  ; %res = 1
%res = call i4 @llvm.ssub.sat.i4(i4 2, i4 6)  ; %res = -4
%res = call i4 @llvm.ssub.sat.i4(i4 -4, i4 5)  ; %res = -8
%res = call i4 @llvm.ssub.sat.i4(i4 4, i4 -5)  ; %res = 7
```

#### \'`llvm.usub.sat.*`\' Intrinsics

##### Syntax

This is an overloaded intrinsic. You can use `llvm.usub.sat` on any
integer bit width or vectors of integers.

    declare i16 @llvm.usub.sat.i16(i16 %a, i16 %b)
    declare i32 @llvm.usub.sat.i32(i32 %a, i32 %b)
    declare i64 @llvm.usub.sat.i64(i64 %a, i64 %b)
    declare <4 x i32> @llvm.usub.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

##### Overview

The \'`llvm.usub.sat`\' family of intrinsic functions perform unsigned
saturation subtraction on the 2 arguments.

##### Arguments

The arguments (%a and %b) and the result may be of integer types of any
bit width, but they must have the same bit width. `%a` and `%b` are the
two values that will undergo unsigned subtraction.

##### Semantics:

The minimum value this operation can clamp to is 0, which is the
smallest unsigned value representable by the bit width of the unsigned
arguments. Because this is an unsigned operation, the result will never
saturate towards the largest possible value representable by this bit
width.

##### Examples

``` {.llvm}
%res = call i4 @llvm.usub.sat.i4(i4 2, i4 1)  ; %res = 1
%res = call i4 @llvm.usub.sat.i4(i4 2, i4 6)  ; %res = 0
```

### Fixed Point Arithmetic Intrinsics

A fixed point number represents a real data type for a number that has a
fixed number of digits after a radix point (equivalent to the decimal
point \'.\'). The number of digits after the radix point is referred as
the `scale`. These are useful for representing fractional values to a
specific precision. The following intrinsics perform fixed point
arithmetic operations on 2 operands of the same scale, specified as the
third argument.

#### \'`llvm.smul.fix.*`\' Intrinsics

##### Syntax

This is an overloaded intrinsic. You can use `llvm.smul.fix` on any
integer bit width or vectors of integers.

    declare i16 @llvm.smul.fix.i16(i16 %a, i16 %b, i32 %scale)
    declare i32 @llvm.smul.fix.i32(i32 %a, i32 %b, i32 %scale)
    declare i64 @llvm.smul.fix.i64(i64 %a, i64 %b, i32 %scale)
    declare <4 x i32> @llvm.smul.fix.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

##### Overview

The \'`llvm.smul.fix`\' family of intrinsic functions perform signed
fixed point multiplication on 2 arguments of the same scale.

##### Arguments

The arguments (%a and %b) and the result may be of integer types of any
bit width, but they must have the same bit width. `%a` and `%b` are the
two values that will undergo signed fixed point multiplication. The
argument `%scale` represents the scale of both operands, and must be a
constant integer.

##### Semantics:

This operation performs fixed point multiplication on the 2 arguments of
a specified scale. The result will also be returned in the same scale
specified in the third argument.

If the result value cannot be precisely represented in the given scale,
the value is rounded up or down to the closest representable value. The
rounding direction is unspecified.

It is undefined behavior if the source value does not fit within the
range of the fixed point type.

##### Examples

``` {.llvm}
%res = call i4 @llvm.smul.fix.i4(i4 3, i4 2, i32 0)  ; %res = 6 (2 x 3 = 6)
%res = call i4 @llvm.smul.fix.i4(i4 3, i4 2, i32 1)  ; %res = 3 (1.5 x 1 = 1.5)
%res = call i4 @llvm.smul.fix.i4(i4 3, i4 -2, i32 1)  ; %res = -3 (1.5 x -1 = -1.5)

; The result in the following could be rounded up to -2 or down to -2.5
%res = call i4 @llvm.smul.fix.i4(i4 3, i4 -3, i32 1)  ; %res = -5 (or -4) (1.5 x -1.5 = -2.25)
```

### Specialised Arithmetic Intrinsics

#### \'`llvm.canonicalize.*`\' Intrinsic

##### Syntax:

    declare float @llvm.canonicalize.f32(float %a)
    declare double @llvm.canonicalize.f64(double %b)

##### Overview:

The \'`llvm.canonicalize.*`\' intrinsic returns the platform specific
canonical encoding of a floating-point number. This canonicalization is
useful for implementing certain numeric primitives such as frexp. The
canonical encoding is defined by IEEE-754-2008 to be:

    2.1.8 canonical encoding: The preferred encoding of a floating-point
    representation in a format. Applied to declets, significands of finite
    numbers, infinities, and NaNs, especially in decimal formats.

This operation can also be considered equivalent to the IEEE-754-2008
conversion of a floating-point value to the same format. NaNs are
handled according to section 6.2.

Examples of non-canonical encodings:

-   x87 pseudo denormals, pseudo NaNs, pseudo Infinity, Unnormals. These
    are converted to a canonical representation per hardware-specific
    protocol.
-   Many normal decimal floating-point numbers have non-canonical
    alternative encodings.
-   Some machines, like GPUs or ARMv7 NEON, do not support subnormal
    values. These are treated as non-canonical encodings of zero and
    will be flushed to a zero of the same sign by this operation.

Note that per IEEE-754-2008 6.2, systems that support signaling NaNs
with default exception handling must signal an invalid exception, and
produce a quiet NaN result.

This function should always be implementable as multiplication by 1.0,
provided that the compiler does not constant fold the operation.
Likewise, division by 1.0 and `llvm.minnum(x, x)` are possible
implementations. Addition with -0.0 is also sufficient provided that the
rounding mode is not -Infinity.

`@llvm.canonicalize` must preserve the equality relation. That is:

-   `(@llvm.canonicalize(x) == x)` is equivalent to `(x == x)`
-   `(@llvm.canonicalize(x) == @llvm.canonicalize(y))` is equivalent to
    to `(x == y)`

Additionally, the sign of zero must be conserved:
`@llvm.canonicalize(-0.0) = -0.0` and `@llvm.canonicalize(+0.0) = +0.0`

The payload bits of a NaN must be conserved, with two exceptions. First,
environments which use only a single canonical representation of NaN
must perform said canonicalization. Second, SNaNs must be quieted per
the usual methods.

The canonicalization operation may be optimized away if:

-   The input is known to be canonical. For example, it was produced by
    a floating-point operation that is required by the standard to be
    canonical.
-   The result is consumed only by (or fused with) other floating-point
    operations. That is, the bits of the floating-point value are not
    examined.

#### \'`llvm.fmuladd.*`\' Intrinsic

##### Syntax:

    declare float @llvm.fmuladd.f32(float %a, float %b, float %c)
    declare double @llvm.fmuladd.f64(double %a, double %b, double %c)

##### Overview:

The \'`llvm.fmuladd.*`\' intrinsic functions represent multiply-add
expressions that can be fused if the code generator determines that (a)
the target instruction set has support for a fused operation, and (b)
that the fused operation is more efficient than the equivalent, separate
pair of mul and add instructions.

##### Arguments:

The \'`llvm.fmuladd.*`\' intrinsics each take three arguments: two
multiplicands, a and b, and an addend c.

##### Semantics:

The expression:

    %0 = call float @llvm.fmuladd.f32(%a, %b, %c)

is equivalent to the expression a \* b + c, except that rounding will
not be performed between the multiplication and addition steps if the
code generator fuses the operations. Fusion is not guaranteed, even if
the target platform supports it. If a fused multiply-add is required the
corresponding llvm.fma.\* intrinsic function should be used instead.
This never sets errno, just as \'`llvm.fma.*`\'.

##### Examples:

``` {.llvm}
%r2 = call float @llvm.fmuladd.f32(float %a, float %b, float %c) ; yields float:r2 = (a * b) + c
```

### Experimental Vector Reduction Intrinsics

Horizontal reductions of vectors can be expressed using the following
intrinsics. Each one takes a vector operand as an input and applies its
respective operation across all elements of the vector, returning a
single scalar result of the same element type.

#### \'`llvm.experimental.vector.reduce.add.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.add.i32.v4i32(<4 x i32> %a)
    declare i64 @llvm.experimental.vector.reduce.add.i64.v2i64(<2 x i64> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.add.*`\' intrinsics do an integer
`ADD` reduction of a vector, returning the result as a scalar. The
return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.fadd.*`\' Intrinsic

##### Syntax:

    declare float @llvm.experimental.vector.reduce.fadd.f32.v4f32(float %acc, <4 x float> %a)
    declare double @llvm.experimental.vector.reduce.fadd.f64.v2f64(double %acc, <2 x double> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.fadd.*`\' intrinsics do a
floating-point `ADD` reduction of a vector, returning the result as a
scalar. The return type matches the element-type of the vector input.

If the intrinsic call has fast-math flags, then the reduction will not
preserve the associativity of an equivalent scalarized counterpart. If
it does not have fast-math flags, then the reduction will be *ordered*,
implying that the operation respects the associativity of a scalarized
reduction.

##### Arguments:

The first argument to this intrinsic is a scalar accumulator value,
which is only used when there are no fast-math flags attached. This
argument may be undef when fast-math flags are used.

The second argument must be a vector of floating-point values.

##### Examples:

``` {.llvm}
%fast = call fast float @llvm.experimental.vector.reduce.fadd.f32.v4f32(float undef, <4 x float> %input) ; fast reduction
%ord = call float @llvm.experimental.vector.reduce.fadd.f32.v4f32(float %acc, <4 x float> %input) ; ordered reduction
```

#### \'`llvm.experimental.vector.reduce.mul.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.mul.i32.v4i32(<4 x i32> %a)
    declare i64 @llvm.experimental.vector.reduce.mul.i64.v2i64(<2 x i64> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.mul.*`\' intrinsics do an integer
`MUL` reduction of a vector, returning the result as a scalar. The
return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.fmul.*`\' Intrinsic

##### Syntax:

    declare float @llvm.experimental.vector.reduce.fmul.f32.v4f32(float %acc, <4 x float> %a)
    declare double @llvm.experimental.vector.reduce.fmul.f64.v2f64(double %acc, <2 x double> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.fmul.*`\' intrinsics do a
floating-point `MUL` reduction of a vector, returning the result as a
scalar. The return type matches the element-type of the vector input.

If the intrinsic call has fast-math flags, then the reduction will not
preserve the associativity of an equivalent scalarized counterpart. If
it does not have fast-math flags, then the reduction will be *ordered*,
implying that the operation respects the associativity of a scalarized
reduction.

##### Arguments:

The first argument to this intrinsic is a scalar accumulator value,
which is only used when there are no fast-math flags attached. This
argument may be undef when fast-math flags are used.

The second argument must be a vector of floating-point values.

##### Examples:

``` {.llvm}
%fast = call fast float @llvm.experimental.vector.reduce.fmul.f32.v4f32(float undef, <4 x float> %input) ; fast reduction
%ord = call float @llvm.experimental.vector.reduce.fmul.f32.v4f32(float %acc, <4 x float> %input) ; ordered reduction
```

#### \'`llvm.experimental.vector.reduce.and.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.and.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.and.*`\' intrinsics do a bitwise
`AND` reduction of a vector, returning the result as a scalar. The
return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.or.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.or.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.or.*`\' intrinsics do a bitwise
`OR` reduction of a vector, returning the result as a scalar. The return
type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.xor.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.xor.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.xor.*`\' intrinsics do a bitwise
`XOR` reduction of a vector, returning the result as a scalar. The
return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.smax.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.smax.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.smax.*`\' intrinsics do a signed
integer `MAX` reduction of a vector, returning the result as a scalar.
The return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.smin.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.smin.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.smin.*`\' intrinsics do a signed
integer `MIN` reduction of a vector, returning the result as a scalar.
The return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.umax.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.umax.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.umax.*`\' intrinsics do an
unsigned integer `MAX` reduction of a vector, returning the result as a
scalar. The return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.umin.*`\' Intrinsic

##### Syntax:

    declare i32 @llvm.experimental.vector.reduce.umin.i32.v4i32(<4 x i32> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.umin.*`\' intrinsics do an
unsigned integer `MIN` reduction of a vector, returning the result as a
scalar. The return type matches the element-type of the vector input.

##### Arguments:

The argument to this intrinsic must be a vector of integer values.

#### \'`llvm.experimental.vector.reduce.fmax.*`\' Intrinsic

##### Syntax:

    declare float @llvm.experimental.vector.reduce.fmax.f32.v4f32(<4 x float> %a)
    declare double @llvm.experimental.vector.reduce.fmax.f64.v2f64(<2 x double> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.fmax.*`\' intrinsics do a
floating-point `MAX` reduction of a vector, returning the result as a
scalar. The return type matches the element-type of the vector input.

If the intrinsic call has the `nnan` fast-math flag then the operation
can assume that NaNs are not present in the input vector.

##### Arguments:

The argument to this intrinsic must be a vector of floating-point
values.

#### \'`llvm.experimental.vector.reduce.fmin.*`\' Intrinsic

##### Syntax:

    declare float @llvm.experimental.vector.reduce.fmin.f32.v4f32(<4 x float> %a)
    declare double @llvm.experimental.vector.reduce.fmin.f64.v2f64(<2 x double> %a)

##### Overview:

The \'`llvm.experimental.vector.reduce.fmin.*`\' intrinsics do a
floating-point `MIN` reduction of a vector, returning the result as a
scalar. The return type matches the element-type of the vector input.

If the intrinsic call has the `nnan` fast-math flag then the operation
can assume that NaNs are not present in the input vector.

##### Arguments:

The argument to this intrinsic must be a vector of floating-point
values.

### Half Precision Floating-Point Intrinsics

For most target platforms, half precision floating-point is a
storage-only format. This means that it is a dense encoding (in memory)
but does not support computation in the format.

This means that code must first load the half-precision floating-point
value as an i16, then convert it to float with
`llvm.convert.from.fp16 <int_convert_from_fp16>`{.interpreted-text
role="ref"}. Computation can then be performed on the float value
(including extending to double etc). To store the value back to memory,
it is first converted to float if needed, then converted to i16 with
`llvm.convert.to.fp16 <int_convert_to_fp16>`{.interpreted-text
role="ref"}, then storing as an i16 value.

#### \'`llvm.convert.to.fp16`\' Intrinsic {#int_convert_to_fp16}

##### Syntax:

    declare i16 @llvm.convert.to.fp16.f32(float %a)
    declare i16 @llvm.convert.to.fp16.f64(double %a)

##### Overview:

The \'`llvm.convert.to.fp16`\' intrinsic function performs a conversion
from a conventional floating-point type to half precision floating-point
format.

##### Arguments:

The intrinsic function contains single argument - the value to be
converted.

##### Semantics:

The \'`llvm.convert.to.fp16`\' intrinsic function performs a conversion
from a conventional floating-point format to half precision
floating-point format. The return value is an `i16` which contains the
converted number.

##### Examples:

``` {.llvm}
%res = call i16 @llvm.convert.to.fp16.f32(float %a)
store i16 %res, i16* @x, align 2
```

#### \'`llvm.convert.from.fp16`\' Intrinsic {#int_convert_from_fp16}

##### Syntax:

    declare float @llvm.convert.from.fp16.f32(i16 %a)
    declare double @llvm.convert.from.fp16.f64(i16 %a)

##### Overview:

The \'`llvm.convert.from.fp16`\' intrinsic function performs a
conversion from half precision floating-point format to single precision
floating-point format.

##### Arguments:

The intrinsic function contains single argument - the value to be
converted.

##### Semantics:

The \'`llvm.convert.from.fp16`\' intrinsic function performs a
conversion from half single precision floating-point format to single
precision floating-point format. The input half-float value is
represented by an `i16` value.

##### Examples:

``` {.llvm}
%a = load i16, i16* @x, align 2
%res = call float @llvm.convert.from.fp16(i16 %a)
```

### Debugger Intrinsics {#dbg_intrinsics}

The LLVM debugger intrinsics (which all start with `llvm.dbg.` prefix),
are described in the [LLVM Source Level
Debugging](SourceLevelDebugging.html#format-common-intrinsics) document.

### Exception Handling Intrinsics

The LLVM exception handling intrinsics (which all start with `llvm.eh.`
prefix), are described in the [LLVM Exception
Handling](ExceptionHandling.html#format-common-intrinsics) document.

### Trampoline Intrinsics {#int_trampoline}

These intrinsics make it possible to excise one parameter, marked with
the `nest <nest>`{.interpreted-text role="ref"} attribute, from a
function. The result is a callable function pointer lacking the nest
parameter - the caller does not need to provide a value for it. Instead,
the value to use is stored in advance in a \"trampoline\", a block of
memory usually allocated on the stack, which also contains code to
splice the nest value into the argument list. This is used to implement
the GCC nested function address extension.

For example, if the function is `i32 f(i8* nest %c, i32 %x, i32 %y)`
then the resulting function pointer has signature `i32 (i32, i32)*`. It
can be created as follows:

``` {.llvm}
%tramp = alloca [10 x i8], align 4 ; size and alignment only correct for X86
%tramp1 = getelementptr [10 x i8], [10 x i8]* %tramp, i32 0, i32 0
call i8* @llvm.init.trampoline(i8* %tramp1, i8* bitcast (i32 (i8*, i32, i32)* @f to i8*), i8* %nval)
%p = call i8* @llvm.adjust.trampoline(i8* %tramp1)
%fp = bitcast i8* %p to i32 (i32, i32)*
```

The call `%val = call i32 %fp(i32 %x, i32 %y)` is then equivalent to
`%val = call i32 %f(i8* %nval, i32 %x, i32 %y)`.

#### \'`llvm.init.trampoline`\' Intrinsic {#int_it}

##### Syntax:

    declare void @llvm.init.trampoline(i8* <tramp>, i8* <func>, i8* <nval>)

##### Overview:

This fills the memory pointed to by `tramp` with executable code,
turning it into a trampoline.

##### Arguments:

The `llvm.init.trampoline` intrinsic takes three arguments, all
pointers. The `tramp` argument must point to a sufficiently large and
sufficiently aligned block of memory; this memory is written to by the
intrinsic. Note that the size and the alignment are target-specific
-LLVM currently provides no portable way of determining them, so a
front-end that generates this intrinsic needs to have some
target-specific knowledge. The `func` argument must hold a function
bitcast to an `i8*`.

##### Semantics:

The block of memory pointed to by `tramp` is filled with target
dependent code, turning it into a function. Then `tramp` needs to be
passed to `llvm.adjust.trampoline <int_at>`{.interpreted-text
role="ref"} to get a pointer which can be
`bitcast (to a new function) and called <int_trampoline>`{.interpreted-text
role="ref"}. The new function\'s signature is the same as that of `func`
with any arguments marked with the `nest` attribute removed. At most one
such `nest` argument is allowed, and it must be of pointer type. Calling
the new function is equivalent to calling `func` with the same argument
list, but with `nval` used for the missing `nest` argument. If, after
calling `llvm.init.trampoline`, the memory pointed to by `tramp` is
modified, then the effect of any later call to the returned function
pointer is undefined.

#### \'`llvm.adjust.trampoline`\' Intrinsic {#int_at}

##### Syntax:

    declare i8* @llvm.adjust.trampoline(i8* <tramp>)

##### Overview:

This performs any required machine-specific adjustment to the address of
a trampoline (passed as `tramp`).

##### Arguments:

`tramp` must point to a block of memory which already has trampoline
code filled in by a previous call to
`llvm.init.trampoline <int_it>`{.interpreted-text role="ref"}.

##### Semantics:

On some architectures the address of the code to be executed needs to be
different than the address where the trampoline is actually stored. This
intrinsic returns the executable address corresponding to `tramp` after
performing the required machine specific adjustments. The pointer
returned can then be
`bitcast and executed <int_trampoline>`{.interpreted-text role="ref"}.

### Masked Vector Load and Store Intrinsics {#int_mload_mstore}

LLVM provides intrinsics for predicated vector load and store
operations. The predicate is specified by a mask operand, which holds
one bit per vector element, switching the associated vector lane on or
off. The memory addresses corresponding to the \"off\" lanes are not
accessed. When all bits of the mask are on, the intrinsic is identical
to a regular vector load or store. When all bits are off, no memory is
accessed.

#### \'`llvm.masked.load.*`\' Intrinsics {#int_mload}

##### Syntax:

This is an overloaded intrinsic. The loaded data is a vector of any
integer, floating-point or pointer data type.

    declare <16 x float>  @llvm.masked.load.v16f32.p0v16f32 (<16 x float>* <ptr>, i32 <alignment>, <16 x i1> <mask>, <16 x float> <passthru>)
    declare <2 x double>  @llvm.masked.load.v2f64.p0v2f64  (<2 x double>* <ptr>, i32 <alignment>, <2 x i1>  <mask>, <2 x double> <passthru>)
    ;; The data is a vector of pointers to double
    declare <8 x double*> @llvm.masked.load.v8p0f64.p0v8p0f64    (<8 x double*>* <ptr>, i32 <alignment>, <8 x i1> <mask>, <8 x double*> <passthru>)
    ;; The data is a vector of function pointers
    declare <8 x i32 ()*> @llvm.masked.load.v8p0f_i32f.p0v8p0f_i32f (<8 x i32 ()*>* <ptr>, i32 <alignment>, <8 x i1> <mask>, <8 x i32 ()*> <passthru>)

##### Overview:

Reads a vector from memory according to the provided mask. The mask
holds a bit for each vector lane, and is used to prevent memory accesses
to the masked-off lanes. The masked-off lanes in the result vector are
taken from the corresponding lanes of the \'`passthru`\' operand.

##### Arguments:

The first operand is the base pointer for the load. The second operand
is the alignment of the source location. It must be a constant integer
value. The third operand, mask, is a vector of boolean values with the
same number of elements as the return type. The fourth is a pass-through
value that is used to fill the masked-off lanes of the result. The
return type, underlying type of the base pointer and the type of the
\'`passthru`\' operand are the same vector types.

##### Semantics:

The \'`llvm.masked.load`\' intrinsic is designed for conditional reading
of selected vector elements in a single IR operation. It is useful for
targets that support vector masked loads and allows vectorizing
predicated basic blocks on these targets. Other targets may support this
intrinsic differently, for example by lowering it into a sequence of
branches that guard scalar load operations. The result of this operation
is equivalent to a regular vector load instruction followed by a
\'select\' between the loaded and the passthru values, predicated on the
same mask. However, using this intrinsic prevents exceptions on memory
access to masked-off lanes.

    %res = call <16 x float> @llvm.masked.load.v16f32.p0v16f32 (<16 x float>* %ptr, i32 4, <16 x i1>%mask, <16 x float> %passthru)

    ;; The result of the two following instructions is identical aside from potential memory access exception
    %loadlal = load <16 x float>, <16 x float>* %ptr, align 4
    %res = select <16 x i1> %mask, <16 x float> %loadlal, <16 x float> %passthru

#### \'`llvm.masked.store.*`\' Intrinsics {#int_mstore}

##### Syntax:

This is an overloaded intrinsic. The data stored in memory is a vector
of any integer, floating-point or pointer data type.

    declare void @llvm.masked.store.v8i32.p0v8i32  (<8  x i32>   <value>, <8  x i32>*   <ptr>, i32 <alignment>,  <8  x i1> <mask>)
    declare void @llvm.masked.store.v16f32.p0v16f32 (<16 x float> <value>, <16 x float>* <ptr>, i32 <alignment>,  <16 x i1> <mask>)
    ;; The data is a vector of pointers to double
    declare void @llvm.masked.store.v8p0f64.p0v8p0f64    (<8 x double*> <value>, <8 x double*>* <ptr>, i32 <alignment>, <8 x i1> <mask>)
    ;; The data is a vector of function pointers
    declare void @llvm.masked.store.v4p0f_i32f.p0v4p0f_i32f (<4 x i32 ()*> <value>, <4 x i32 ()*>* <ptr>, i32 <alignment>, <4 x i1> <mask>)

##### Overview:

Writes a vector to memory according to the provided mask. The mask holds
a bit for each vector lane, and is used to prevent memory accesses to
the masked-off lanes.

##### Arguments:

The first operand is the vector value to be written to memory. The
second operand is the base pointer for the store, it has the same
underlying type as the value operand. The third operand is the alignment
of the destination location. The fourth operand, mask, is a vector of
boolean values. The types of the mask and the value operand must have
the same number of vector elements.

##### Semantics:

The \'`llvm.masked.store`\' intrinsics is designed for conditional
writing of selected vector elements in a single IR operation. It is
useful for targets that support vector masked store and allows
vectorizing predicated basic blocks on these targets. Other targets may
support this intrinsic differently, for example by lowering it into a
sequence of branches that guard scalar store operations. The result of
this operation is equivalent to a load-modify-store sequence. However,
using this intrinsic prevents exceptions and data races on memory access
to masked-off lanes.

    call void @llvm.masked.store.v16f32.p0v16f32(<16 x float> %value, <16 x float>* %ptr, i32 4,  <16 x i1> %mask)

    ;; The result of the following instructions is identical aside from potential data races and memory access exceptions
    %oldval = load <16 x float>, <16 x float>* %ptr, align 4
    %res = select <16 x i1> %mask, <16 x float> %value, <16 x float> %oldval
    store <16 x float> %res, <16 x float>* %ptr, align 4

### Masked Vector Gather and Scatter Intrinsics

LLVM provides intrinsics for vector gather and scatter operations. They
are similar to
`Masked Vector Load and Store <int_mload_mstore>`{.interpreted-text
role="ref"}, except they are designed for arbitrary memory accesses,
rather than sequential memory accesses. Gather and scatter also employ a
mask operand, which holds one bit per vector element, switching the
associated vector lane on or off. The memory addresses corresponding to
the \"off\" lanes are not accessed. When all bits are off, no memory is
accessed.

#### \'`llvm.masked.gather.*`\' Intrinsics {#int_mgather}

##### Syntax:

This is an overloaded intrinsic. The loaded data are multiple scalar
values of any integer, floating-point or pointer data type gathered
together into one vector.

    declare <16 x float> @llvm.masked.gather.v16f32.v16p0f32   (<16 x float*> <ptrs>, i32 <alignment>, <16 x i1> <mask>, <16 x float> <passthru>)
    declare <2 x double> @llvm.masked.gather.v2f64.v2p1f64     (<2 x double addrspace(1)*> <ptrs>, i32 <alignment>, <2 x i1>  <mask>, <2 x double> <passthru>)
    declare <8 x float*> @llvm.masked.gather.v8p0f32.v8p0p0f32 (<8 x float**> <ptrs>, i32 <alignment>, <8 x i1>  <mask>, <8 x float*> <passthru>)

##### Overview:

Reads scalar values from arbitrary memory locations and gathers them
into one vector. The memory locations are provided in the vector of
pointers \'`ptrs`\'. The memory is accessed according to the provided
mask. The mask holds a bit for each vector lane, and is used to prevent
memory accesses to the masked-off lanes. The masked-off lanes in the
result vector are taken from the corresponding lanes of the
\'`passthru`\' operand.

##### Arguments:

The first operand is a vector of pointers which holds all memory
addresses to read. The second operand is an alignment of the source
addresses. It must be a constant integer value. The third operand, mask,
is a vector of boolean values with the same number of elements as the
return type. The fourth is a pass-through value that is used to fill the
masked-off lanes of the result. The return type, underlying type of the
vector of pointers and the type of the \'`passthru`\' operand are the
same vector types.

##### Semantics:

The \'`llvm.masked.gather`\' intrinsic is designed for conditional
reading of multiple scalar values from arbitrary memory locations in a
single IR operation. It is useful for targets that support vector masked
gathers and allows vectorizing basic blocks with data and control
divergence. Other targets may support this intrinsic differently, for
example by lowering it into a sequence of scalar load operations. The
semantics of this operation are equivalent to a sequence of conditional
scalar loads with subsequent gathering all loaded values into a single
vector. The mask restricts memory access to certain lanes and
facilitates vectorization of predicated basic blocks.

    %res = call <4 x double> @llvm.masked.gather.v4f64.v4p0f64 (<4 x double*> %ptrs, i32 8, <4 x i1> <i1 true, i1 true, i1 true, i1 true>, <4 x double> undef)

    ;; The gather with all-true mask is equivalent to the following instruction sequence
    %ptr0 = extractelement <4 x double*> %ptrs, i32 0
    %ptr1 = extractelement <4 x double*> %ptrs, i32 1
    %ptr2 = extractelement <4 x double*> %ptrs, i32 2
    %ptr3 = extractelement <4 x double*> %ptrs, i32 3

    %val0 = load double, double* %ptr0, align 8
    %val1 = load double, double* %ptr1, align 8
    %val2 = load double, double* %ptr2, align 8
    %val3 = load double, double* %ptr3, align 8

    %vec0    = insertelement <4 x double>undef, %val0, 0
    %vec01   = insertelement <4 x double>%vec0, %val1, 1
    %vec012  = insertelement <4 x double>%vec01, %val2, 2
    %vec0123 = insertelement <4 x double>%vec012, %val3, 3

#### \'`llvm.masked.scatter.*`\' Intrinsics {#int_mscatter}

##### Syntax:

This is an overloaded intrinsic. The data stored in memory is a vector
of any integer, floating-point or pointer data type. Each vector element
is stored in an arbitrary memory address. Scatter with overlapping
addresses is guaranteed to be ordered from least-significant to
most-significant element.

    declare void @llvm.masked.scatter.v8i32.v8p0i32     (<8 x i32>     <value>, <8 x i32*>     <ptrs>, i32 <alignment>, <8 x i1>  <mask>)
    declare void @llvm.masked.scatter.v16f32.v16p1f32   (<16 x float>  <value>, <16 x float addrspace(1)*>  <ptrs>, i32 <alignment>, <16 x i1> <mask>)
    declare void @llvm.masked.scatter.v4p0f64.v4p0p0f64 (<4 x double*> <value>, <4 x double**> <ptrs>, i32 <alignment>, <4 x i1>  <mask>)

##### Overview:

Writes each element from the value vector to the corresponding memory
address. The memory addresses are represented as a vector of pointers.
Writing is done according to the provided mask. The mask holds a bit for
each vector lane, and is used to prevent memory accesses to the
masked-off lanes.

##### Arguments:

The first operand is a vector value to be written to memory. The second
operand is a vector of pointers, pointing to where the value elements
should be stored. It has the same underlying type as the value operand.
The third operand is an alignment of the destination addresses. The
fourth operand, mask, is a vector of boolean values. The types of the
mask and the value operand must have the same number of vector elements.

##### Semantics:

The \'`llvm.masked.scatter`\' intrinsics is designed for writing
selected vector elements to arbitrary memory addresses in a single IR
operation. The operation may be conditional, when not all bits in the
mask are switched on. It is useful for targets that support vector
masked scatter and allows vectorizing basic blocks with data and control
divergence. Other targets may support this intrinsic differently, for
example by lowering it into a sequence of branches that guard scalar
store operations.

    ;; This instruction unconditionally stores data vector in multiple addresses
    call @llvm.masked.scatter.v8i32.v8p0i32 (<8 x i32> %value, <8 x i32*> %ptrs, i32 4,  <8 x i1>  <true, true, .. true>)

    ;; It is equivalent to a list of scalar stores
    %val0 = extractelement <8 x i32> %value, i32 0
    %val1 = extractelement <8 x i32> %value, i32 1
    ..
    %val7 = extractelement <8 x i32> %value, i32 7
    %ptr0 = extractelement <8 x i32*> %ptrs, i32 0
    %ptr1 = extractelement <8 x i32*> %ptrs, i32 1
    ..
    %ptr7 = extractelement <8 x i32*> %ptrs, i32 7
    ;; Note: the order of the following stores is important when they overlap:
    store i32 %val0, i32* %ptr0, align 4
    store i32 %val1, i32* %ptr1, align 4
    ..
    store i32 %val7, i32* %ptr7, align 4

### Masked Vector Expanding Load and Compressing Store Intrinsics

LLVM provides intrinsics for expanding load and compressing store
operations. Data selected from a vector according to a mask is stored in
consecutive memory addresses (compressed store), and vice-versa
(expanding load). These operations effective map to \"if (cond.i)
a\[j++\] = v.i\" and \"if (cond.i) v.i = a\[j++\]\" patterns,
respectively. Note that when the mask starts with \'1\' bits followed by
\'0\' bits, these operations are identical to
`llvm.masked.store <int_mstore>`{.interpreted-text role="ref"} and
`llvm.masked.load <int_mload>`{.interpreted-text role="ref"}.

#### \'`llvm.masked.expandload.*`\' Intrinsics {#int_expandload}

##### Syntax:

This is an overloaded intrinsic. Several values of integer, floating
point or pointer data type are loaded from consecutive memory addresses
and stored into the elements of a vector according to the mask.

    declare <16 x float>  @llvm.masked.expandload.v16f32 (float* <ptr>, <16 x i1> <mask>, <16 x float> <passthru>)
    declare <2 x i64>     @llvm.masked.expandload.v2i64 (i64* <ptr>, <2 x i1>  <mask>, <2 x i64> <passthru>)

##### Overview:

Reads a number of scalar values sequentially from memory location
provided in \'`ptr`\' and spreads them in a vector. The \'`mask`\' holds
a bit for each vector lane. The number of elements read from memory is
equal to the number of \'1\' bits in the mask. The loaded elements are
positioned in the destination vector according to the sequence of \'1\'
and \'0\' bits in the mask. E.g., if the mask vector is \'10010001\',
\"explandload\" reads 3 values from memory addresses ptr, ptr+1, ptr+2
and places them in lanes 0, 3 and 7 accordingly. The masked-off lanes
are filled by elements from the corresponding lanes of the
\'`passthru`\' operand.

##### Arguments:

The first operand is the base pointer for the load. It has the same
underlying type as the element of the returned vector. The second
operand, mask, is a vector of boolean values with the same number of
elements as the return type. The third is a pass-through value that is
used to fill the masked-off lanes of the result. The return type and the
type of the \'`passthru`\' operand have the same vector type.

##### Semantics:

The \'`llvm.masked.expandload`\' intrinsic is designed for reading
multiple scalar values from adjacent memory addresses into possibly
non-adjacent vector lanes. It is useful for targets that support vector
expanding loads and allows vectorizing loop with cross-iteration
dependency like in the following example:

``` {.c}
// In this loop we load from B and spread the elements into array A.
double *A, B; int *C;
for (int i = 0; i < size; ++i) {
  if (C[i] != 0)
    A[i] = B[j++];
}
```

``` {.llvm}
; Load several elements from array B and expand them in a vector.
; The number of loaded elements is equal to the number of '1' elements in the Mask.
%Tmp = call <8 x double> @llvm.masked.expandload.v8f64(double* %Bptr, <8 x i1> %Mask, <8 x double> undef)
; Store the result in A
call void @llvm.masked.store.v8f64.p0v8f64(<8 x double> %Tmp, <8 x double>* %Aptr, i32 8, <8 x i1> %Mask)

; %Bptr should be increased on each iteration according to the number of '1' elements in the Mask.
%MaskI = bitcast <8 x i1> %Mask to i8
%MaskIPopcnt = call i8 @llvm.ctpop.i8(i8 %MaskI)
%MaskI64 = zext i8 %MaskIPopcnt to i64
%BNextInd = add i64 %BInd, %MaskI64
```

Other targets may support this intrinsic differently, for example, by
lowering it into a sequence of conditional scalar load operations and
shuffles. If all mask elements are \'1\', the intrinsic behavior is
equivalent to the regular unmasked vector load.

#### \'`llvm.masked.compressstore.*`\' Intrinsics {#int_compressstore}

##### Syntax:

This is an overloaded intrinsic. A number of scalar values of integer,
floating point or pointer data type are collected from an input vector
and stored into adjacent memory addresses. A mask defines which elements
to collect from the vector.

    declare void @llvm.masked.compressstore.v8i32  (<8  x i32>   <value>, i32*   <ptr>, <8  x i1> <mask>)
    declare void @llvm.masked.compressstore.v16f32 (<16 x float> <value>, float* <ptr>, <16 x i1> <mask>)

##### Overview:

Selects elements from input vector \'`value`\' according to the
\'`mask`\'. All selected elements are written into adjacent memory
addresses starting at address \'[ptr]{.title-ref}\', from lower to
higher. The mask holds a bit for each vector lane, and is used to select
elements to be stored. The number of elements to be stored is equal to
the number of active bits in the mask.

##### Arguments:

The first operand is the input vector, from which elements are collected
and written to memory. The second operand is the base pointer for the
store, it has the same underlying type as the element of the input
vector operand. The third operand is the mask, a vector of boolean
values. The mask and the input vector must have the same number of
vector elements.

##### Semantics:

The \'`llvm.masked.compressstore`\' intrinsic is designed for
compressing data in memory. It allows to collect elements from possibly
non-adjacent lanes of a vector and store them contiguously in memory in
one IR operation. It is useful for targets that support compressing
store operations and allows vectorizing loops with cross-iteration
dependences like in the following example:

``` {.c}
// In this loop we load elements from A and store them consecutively in B
double *A, B; int *C;
for (int i = 0; i < size; ++i) {
  if (C[i] != 0)
    B[j++] = A[i]
}
```

``` {.llvm}
; Load elements from A.
%Tmp = call <8 x double> @llvm.masked.load.v8f64.p0v8f64(<8 x double>* %Aptr, i32 8, <8 x i1> %Mask, <8 x double> undef)
; Store all selected elements consecutively in array B
call <void> @llvm.masked.compressstore.v8f64(<8 x double> %Tmp, double* %Bptr, <8 x i1> %Mask)

; %Bptr should be increased on each iteration according to the number of '1' elements in the Mask.
%MaskI = bitcast <8 x i1> %Mask to i8
%MaskIPopcnt = call i8 @llvm.ctpop.i8(i8 %MaskI)
%MaskI64 = zext i8 %MaskIPopcnt to i64
%BNextInd = add i64 %BInd, %MaskI64
```

Other targets may support this intrinsic differently, for example, by
lowering it into a sequence of branches that guard scalar store
operations.

### Memory Use Markers

This class of intrinsics provides information about the lifetime of
memory objects and ranges where variables are immutable.

#### \'`llvm.lifetime.start`\' Intrinsic {#int_lifestart}

##### Syntax:

    declare void @llvm.lifetime.start(i64 <size>, i8* nocapture <ptr>)

##### Overview:

The \'`llvm.lifetime.start`\' intrinsic specifies the start of a memory
object\'s lifetime.

##### Arguments:

The first argument is a constant integer representing the size of the
object, or -1 if it is variable sized. The second argument is a pointer
to the object.

##### Semantics:

This intrinsic indicates that before this point in the code, the value
of the memory pointed to by `ptr` is dead. This means that it is known
to never be used and has an undefined value. A load from the pointer
that precedes this intrinsic can be replaced with `'undef'`.

#### \'`llvm.lifetime.end`\' Intrinsic {#int_lifeend}

##### Syntax:

    declare void @llvm.lifetime.end(i64 <size>, i8* nocapture <ptr>)

##### Overview:

The \'`llvm.lifetime.end`\' intrinsic specifies the end of a memory
object\'s lifetime.

##### Arguments:

The first argument is a constant integer representing the size of the
object, or -1 if it is variable sized. The second argument is a pointer
to the object.

##### Semantics:

This intrinsic indicates that after this point in the code, the value of
the memory pointed to by `ptr` is dead. This means that it is known to
never be used and has an undefined value. Any stores into the memory
object following this intrinsic may be removed as dead.

#### \'`llvm.invariant.start`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. The memory object can belong to any
address space.

    declare {}* @llvm.invariant.start.p0i8(i64 <size>, i8* nocapture <ptr>)

##### Overview:

The \'`llvm.invariant.start`\' intrinsic specifies that the contents of
a memory object will not change.

##### Arguments:

The first argument is a constant integer representing the size of the
object, or -1 if it is variable sized. The second argument is a pointer
to the object.

##### Semantics:

This intrinsic indicates that until an `llvm.invariant.end` that uses
the return value, the referenced memory location is constant and
unchanging.

#### \'`llvm.invariant.end`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. The memory object can belong to any
address space.

    declare void @llvm.invariant.end.p0i8({}* <start>, i64 <size>, i8* nocapture <ptr>)

##### Overview:

The \'`llvm.invariant.end`\' intrinsic specifies that the contents of a
memory object are mutable.

##### Arguments:

The first argument is the matching `llvm.invariant.start` intrinsic. The
second argument is a constant integer representing the size of the
object, or -1 if it is variable sized and the third argument is a
pointer to the object.

##### Semantics:

This intrinsic indicates that the memory is mutable again.

#### \'`llvm.launder.invariant.group`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. The memory object can belong to any
address space. The returned pointer must belong to the same address
space as the argument.

    declare i8* @llvm.launder.invariant.group.p0i8(i8* <ptr>)

##### Overview:

The \'`llvm.launder.invariant.group`\' intrinsic can be used when an
invariant established by `invariant.group` metadata no longer holds, to
obtain a new pointer value that carries fresh invariant group
information. It is an experimental intrinsic, which means that its
semantics might change in the future.

##### Arguments:

The `llvm.launder.invariant.group` takes only one argument, which is a
pointer to the memory.

##### Semantics:

Returns another pointer that aliases its argument but which is
considered different for the purposes of `load`/`store`
`invariant.group` metadata. It does not read any accessible memory and
the execution can be speculated.

#### \'`llvm.strip.invariant.group`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. The memory object can belong to any
address space. The returned pointer must belong to the same address
space as the argument.

    declare i8* @llvm.strip.invariant.group.p0i8(i8* <ptr>)

##### Overview:

The \'`llvm.strip.invariant.group`\' intrinsic can be used when an
invariant established by `invariant.group` metadata no longer holds, to
obtain a new pointer value that does not carry the invariant
information. It is an experimental intrinsic, which means that its
semantics might change in the future.

##### Arguments:

The `llvm.strip.invariant.group` takes only one argument, which is a
pointer to the memory.

##### Semantics:

Returns another pointer that aliases its argument but which has no
associated `invariant.group` metadata. It does not read any memory and
can be speculated.

### Constrained Floating-Point Intrinsics {#constrainedfp}

These intrinsics are used to provide special handling of floating-point
operations when specific rounding mode or floating-point exception
behavior is required. By default, LLVM optimization passes assume that
the rounding mode is round-to-nearest and that floating-point exceptions
will not be monitored. Constrained FP intrinsics are used to support
non-default rounding modes and accurately preserve exception behavior
without compromising LLVM\'s ability to optimize FP code when the
default behavior is used.

Each of these intrinsics corresponds to a normal floating-point
operation. The first two arguments and the return value are the same as
the corresponding FP operation.

The third argument is a metadata argument specifying the rounding mode
to be assumed. This argument must be one of the following strings:

    "round.dynamic"
    "round.tonearest"
    "round.downward"
    "round.upward"
    "round.towardzero"

If this argument is \"round.dynamic\" optimization passes must assume
that the rounding mode is unknown and may change at runtime. No
transformations that depend on rounding mode may be performed in this
case.

The other possible values for the rounding mode argument correspond to
the similarly named IEEE rounding modes. If the argument is any of these
values optimization passes may perform transformations as long as they
are consistent with the specified rounding mode.

For example, \'x-0\'-\>\'x\' is not a valid transformation if the
rounding mode is \"round.downward\" or \"round.dynamic\" because if the
value of \'x\' is +0 then \'x-0\' should evaluate to \'-0\' when
rounding downward. However, this transformation is legal for all other
rounding modes.

For values other than \"round.dynamic\" optimization passes may assume
that the actual runtime rounding mode (as defined in a target-specific
manner) matches the specified rounding mode, but this is not guaranteed.
Using a specific non-dynamic rounding mode which does not match the
actual rounding mode at runtime results in undefined behavior.

The fourth argument to the constrained floating-point intrinsics
specifies the required exception behavior. This argument must be one of
the following strings:

    "fpexcept.ignore"
    "fpexcept.maytrap"
    "fpexcept.strict"

If this argument is \"fpexcept.ignore\" optimization passes may assume
that the exception status flags will not be read and that floating-point
exceptions will be masked. This allows transformations to be performed
that may change the exception semantics of the original code. For
example, FP operations may be speculatively executed in this case
whereas they must not be for either of the other possible values of this
argument.

If the exception behavior argument is \"fpexcept.maytrap\" optimization
passes must avoid transformations that may raise exceptions that would
not have been raised by the original code (such as speculatively
executing FP operations), but passes are not required to preserve all
exceptions that are implied by the original code. For example,
exceptions may be potentially hidden by constant folding.

If the exception behavior argument is \"fpexcept.strict\" all
transformations must strictly preserve the floating-point exception
semantics of the original code. Any FP exception that would have been
raised by the original code must be raised by the transformed code, and
the transformed code must not raise any FP exceptions that would not
have been raised by the original code. This is the exception behavior
argument that will be used if the code being compiled reads the FP
exception status flags, but this mode can also be used with code that
unmasks FP exceptions.

The number and order of floating-point exceptions is NOT guaranteed. For
example, a series of FP operations that each may raise exceptions may be
vectorized into a single instruction that raises each unique exception a
single time.

#### \'`llvm.experimental.constrained.fadd`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.fadd(<type> <op1>, <type> <op2>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.fadd`\' intrinsic returns the sum
of its two operands.

##### Arguments:

The first two arguments to the \'`llvm.experimental.constrained.fadd`\'
intrinsic must be `floating-point <t_floating>`{.interpreted-text
role="ref"} or `vector <t_vector>`{.interpreted-text role="ref"} of
floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

The value produced is the floating-point sum of the two value operands
and has the same type as the operands.

#### \'`llvm.experimental.constrained.fsub`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.fsub(<type> <op1>, <type> <op2>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.fsub`\' intrinsic returns the
difference of its two operands.

##### Arguments:

The first two arguments to the \'`llvm.experimental.constrained.fsub`\'
intrinsic must be `floating-point <t_floating>`{.interpreted-text
role="ref"} or `vector <t_vector>`{.interpreted-text role="ref"} of
floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

The value produced is the floating-point difference of the two value
operands and has the same type as the operands.

#### \'`llvm.experimental.constrained.fmul`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.fmul(<type> <op1>, <type> <op2>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.fmul`\' intrinsic returns the
product of its two operands.

##### Arguments:

The first two arguments to the \'`llvm.experimental.constrained.fmul`\'
intrinsic must be `floating-point <t_floating>`{.interpreted-text
role="ref"} or `vector <t_vector>`{.interpreted-text role="ref"} of
floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

The value produced is the floating-point product of the two value
operands and has the same type as the operands.

#### \'`llvm.experimental.constrained.fdiv`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.fdiv(<type> <op1>, <type> <op2>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.fdiv`\' intrinsic returns the
quotient of its two operands.

##### Arguments:

The first two arguments to the \'`llvm.experimental.constrained.fdiv`\'
intrinsic must be `floating-point <t_floating>`{.interpreted-text
role="ref"} or `vector <t_vector>`{.interpreted-text role="ref"} of
floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

The value produced is the floating-point quotient of the two value
operands and has the same type as the operands.

#### \'`llvm.experimental.constrained.frem`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.frem(<type> <op1>, <type> <op2>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.frem`\' intrinsic returns the
remainder from the division of its two operands.

##### Arguments:

The first two arguments to the \'`llvm.experimental.constrained.frem`\'
intrinsic must be `floating-point <t_floating>`{.interpreted-text
role="ref"} or `vector <t_vector>`{.interpreted-text role="ref"} of
floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above. The rounding mode argument has no effect,
since the result of frem is never rounded, but the argument is included
for consistency with the other constrained floating-point intrinsics.

##### Semantics:

The value produced is the floating-point remainder from the division of
the two value operands and has the same type as the operands. The
remainder has the same sign as the dividend.

#### \'`llvm.experimental.constrained.fma`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.fma(<type> <op1>, <type> <op2>, <type> <op3>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.fma`\' intrinsic returns the result
of a fused-multiply-add operation on its operands.

##### Arguments:

The first three arguments to the \'`llvm.experimental.constrained.fma`\'
intrinsic must be `floating-point <t_floating>`{.interpreted-text
role="ref"} or `vector
<t_vector>`{.interpreted-text role="ref"} of floating-point values. All
arguments must have identical types.

The fourth and fifth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

The result produced is the product of the first two operands added to
the third operand computed with infinite precision, and then rounded to
the target precision.

### Constrained libm-equivalent Intrinsics

In addition to the basic floating-point operations for which constrained
intrinsics are described above, there are constrained versions of
various operations which provide equivalent behavior to a corresponding
libm function. These intrinsics allow the precise behavior of these
operations with respect to rounding mode and exception behavior to be
controlled.

As with the basic constrained floating-point intrinsics, the rounding
mode and exception behavior arguments only control the behavior of the
optimizer. They do not change the runtime floating-point environment.

#### \'`llvm.experimental.constrained.sqrt`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.sqrt(<type> <op1>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.sqrt`\' intrinsic returns the
square root of the specified value, returning the same value as the libm
\'`sqrt`\' functions would, but without setting `errno`.

##### Arguments:

The first argument and the return type are floating-point numbers of the
same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the nonnegative square root of the specified
value. If the value is less than negative zero, a floating-point
exception occurs and the return value is architecture specific.

#### \'`llvm.experimental.constrained.pow`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.pow(<type> <op1>, <type> <op2>,
                                       metadata <rounding mode>,
                                       metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.pow`\' intrinsic returns the first
operand raised to the (positive or negative) power specified by the
second operand.

##### Arguments:

The first two arguments and the return value are floating-point numbers
of the same type. The second argument specifies the power to which the
first argument should be raised.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the first value raised to the second power,
returning the same values as the libm `pow` functions would, and handles
error conditions in the same way.

#### \'`llvm.experimental.constrained.powi`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.powi(<type> <op1>, i32 <op2>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.powi`\' intrinsic returns the first
operand raised to the (positive or negative) power specified by the
second operand. The order of evaluation of multiplications is not
defined. When a vector of floating-point type is used, the second
argument remains a scalar integer value.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type. The second argument is a 32-bit signed integer specifying
the power to which the first argument should be raised.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the first value raised to the second power with an
unspecified sequence of rounding operations.

#### \'`llvm.experimental.constrained.sin`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.sin(<type> <op1>,
                                       metadata <rounding mode>,
                                       metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.sin`\' intrinsic returns the sine
of the first operand.

##### Arguments:

The first argument and the return type are floating-point numbers of the
same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the sine of the specified operand, returning the
same values as the libm `sin` functions would, and handles error
conditions in the same way.

#### \'`llvm.experimental.constrained.cos`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.cos(<type> <op1>,
                                       metadata <rounding mode>,
                                       metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.cos`\' intrinsic returns the cosine
of the first operand.

##### Arguments:

The first argument and the return type are floating-point numbers of the
same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the cosine of the specified operand, returning the
same values as the libm `cos` functions would, and handles error
conditions in the same way.

#### \'`llvm.experimental.constrained.exp`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.exp(<type> <op1>,
                                       metadata <rounding mode>,
                                       metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.exp`\' intrinsic computes the
base-e exponential of the specified value.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `exp` functions would,
and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.exp2`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.exp2(<type> <op1>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.exp2`\' intrinsic computes the
base-2 exponential of the specified value.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `exp2` functions
would, and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.log`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.log(<type> <op1>,
                                       metadata <rounding mode>,
                                       metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.log`\' intrinsic computes the
base-e logarithm of the specified value.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `log` functions would,
and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.log10`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.log10(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.log10`\' intrinsic computes the
base-10 logarithm of the specified value.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `log10` functions
would, and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.log2`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.log2(<type> <op1>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.log2`\' intrinsic computes the
base-2 logarithm of the specified value.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `log2` functions
would, and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.rint`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.rint(<type> <op1>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.rint`\' intrinsic returns the first
operand rounded to the nearest integer. It may raise an inexact
floating-point exception if the operand is not an integer.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `rint` functions
would, and handles error conditions in the same way. The rounding mode
is described, not determined, by the rounding mode argument. The actual
rounding mode is determined by the runtime floating-point environment.
The rounding mode argument is only intended as information to the
compiler.

#### \'`llvm.experimental.constrained.nearbyint`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.nearbyint(<type> <op1>,
                                             metadata <rounding mode>,
                                             metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.nearbyint`\' intrinsic returns the
first operand rounded to the nearest integer. It will not raise an
inexact floating-point exception if the operand is not an integer.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function returns the same values as the libm `nearbyint` functions
would, and handles error conditions in the same way. The rounding mode
is described, not determined, by the rounding mode argument. The actual
rounding mode is determined by the runtime floating-point environment.
The rounding mode argument is only intended as information to the
compiler.

#### \'`llvm.experimental.constrained.maxnum`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.maxnum(<type> <op1>, <type> <op2>
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.maxnum`\' intrinsic returns the
maximum of the two arguments.

##### Arguments:

The first two arguments and the return value are floating-point numbers
of the same type.

The third and forth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function follows the IEEE-754 semantics for maxNum. The rounding
mode is described, not determined, by the rounding mode argument. The
actual rounding mode is determined by the runtime floating-point
environment. The rounding mode argument is only intended as information
to the compiler.

#### \'`llvm.experimental.constrained.minnum`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.minnum(<type> <op1>, <type> <op2>
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.minnum`\' intrinsic returns the
minimum of the two arguments.

##### Arguments:

The first two arguments and the return value are floating-point numbers
of the same type.

The third and forth arguments specify the rounding mode and exception
behavior as described above.

##### Semantics:

This function follows the IEEE-754 semantics for minNum. The rounding
mode is described, not determined, by the rounding mode argument. The
actual rounding mode is determined by the runtime floating-point
environment. The rounding mode argument is only intended as information
to the compiler.

#### \'`llvm.experimental.constrained.ceil`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.ceil(<type> <op1>,
                                        metadata <rounding mode>,
                                        metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.ceil`\' intrinsic returns the
ceiling of the first operand.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above. The rounding mode is currently unused for
this intrinsic.

##### Semantics:

This function returns the same values as the libm `ceil` functions would
and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.floor`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.floor(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.floor`\' intrinsic returns the
floor of the first operand.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above. The rounding mode is currently unused for
this intrinsic.

##### Semantics:

This function returns the same values as the libm `floor` functions
would and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.round`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.round(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.round`\' intrinsic returns the
first operand rounded to the nearest integer.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the rounding mode and exception
behavior as described above. The rounding mode is currently unused for
this intrinsic.

##### Semantics:

This function returns the same values as the libm `round` functions
would and handles error conditions in the same way.

#### \'`llvm.experimental.constrained.trunc`\' Intrinsic

##### Syntax:

    declare <type>
    @llvm.experimental.constrained.trunc(<type> <op1>,
                                         metadata <truncing mode>,
                                         metadata <exception behavior>)

##### Overview:

The \'`llvm.experimental.constrained.trunc`\' intrinsic returns the
first operand rounded to the nearest integer not larger in magnitude
than the operand.

##### Arguments:

The first argument and the return value are floating-point numbers of
the same type.

The second and third arguments specify the truncing mode and exception
behavior as described above. The truncing mode is currently unused for
this intrinsic.

##### Semantics:

This function returns the same values as the libm `trunc` functions
would and handles error conditions in the same way.

### General Intrinsics

This class of intrinsics is designed to be generic and has no specific
purpose.

#### \'`llvm.var.annotation`\' Intrinsic

##### Syntax:

    declare void @llvm.var.annotation(i8* <val>, i8* <str>, i8* <str>, i32  <int>)

##### Overview:

The \'`llvm.var.annotation`\' intrinsic.

##### Arguments:

The first argument is a pointer to a value, the second is a pointer to a
global string, the third is a pointer to a global string which is the
source file name, and the last argument is the line number.

##### Semantics:

This intrinsic allows annotation of local variables with arbitrary
strings. This can be useful for special purpose optimizations that want
to look for these annotations. These have no other defined use; they are
ignored by code generation and optimization.

#### \'`llvm.ptr.annotation.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use \'`llvm.ptr.annotation`\'
on a pointer to an integer of any width. *NOTE* you must specify an
address space for the pointer. The identifier for the default address
space is the integer \'`0`\'.

    declare i8*   @llvm.ptr.annotation.p<address space>i8(i8* <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i16*  @llvm.ptr.annotation.p<address space>i16(i16* <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i32*  @llvm.ptr.annotation.p<address space>i32(i32* <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i64*  @llvm.ptr.annotation.p<address space>i64(i64* <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i256* @llvm.ptr.annotation.p<address space>i256(i256* <val>, i8* <str>, i8* <str>, i32  <int>)

##### Overview:

The \'`llvm.ptr.annotation`\' intrinsic.

##### Arguments:

The first argument is a pointer to an integer value of arbitrary
bitwidth (result of some expression), the second is a pointer to a
global string, the third is a pointer to a global string which is the
source file name, and the last argument is the line number. It returns
the value of the first argument.

##### Semantics:

This intrinsic allows annotation of a pointer to an integer with
arbitrary strings. This can be useful for special purpose optimizations
that want to look for these annotations. These have no other defined
use; they are ignored by code generation and optimization.

#### \'`llvm.annotation.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use \'`llvm.annotation`\' on
any integer bit width.

    declare i8 @llvm.annotation.i8(i8 <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i16 @llvm.annotation.i16(i16 <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i32 @llvm.annotation.i32(i32 <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i64 @llvm.annotation.i64(i64 <val>, i8* <str>, i8* <str>, i32  <int>)
    declare i256 @llvm.annotation.i256(i256 <val>, i8* <str>, i8* <str>, i32  <int>)

##### Overview:

The \'`llvm.annotation`\' intrinsic.

##### Arguments:

The first argument is an integer value (result of some expression), the
second is a pointer to a global string, the third is a pointer to a
global string which is the source file name, and the last argument is
the line number. It returns the value of the first argument.

##### Semantics:

This intrinsic allows annotations to be put on arbitrary expressions
with arbitrary strings. This can be useful for special purpose
optimizations that want to look for these annotations. These have no
other defined use; they are ignored by code generation and optimization.

#### \'`llvm.codeview.annotation`\' Intrinsic

##### Syntax:

This annotation emits a label at its program point and an associated
`S_ANNOTATION` codeview record with some additional string metadata.
This is used to implement MSVC\'s `__annotation` intrinsic. It is marked
`noduplicate`, so calls to this intrinsic prevent inlining and should be
considered expensive.

    declare void @llvm.codeview.annotation(metadata)

##### Arguments:

The argument should be an MDTuple containing any number of MDStrings.

#### \'`llvm.trap`\' Intrinsic

##### Syntax:

    declare void @llvm.trap() cold noreturn nounwind

##### Overview:

The \'`llvm.trap`\' intrinsic.

##### Arguments:

None.

##### Semantics:

This intrinsic is lowered to the target dependent trap instruction. If
the target does not have a trap instruction, this intrinsic will be
lowered to a call of the `abort()` function.

#### \'`llvm.debugtrap`\' Intrinsic

##### Syntax:

    declare void @llvm.debugtrap() nounwind

##### Overview:

The \'`llvm.debugtrap`\' intrinsic.

##### Arguments:

None.

##### Semantics:

This intrinsic is lowered to code which is intended to cause an
execution trap with the intention of requesting the attention of a
debugger.

#### \'`llvm.stackprotector`\' Intrinsic

##### Syntax:

    declare void @llvm.stackprotector(i8* <guard>, i8** <slot>)

##### Overview:

The `llvm.stackprotector` intrinsic takes the `guard` and stores it onto
the stack at `slot`. The stack slot is adjusted to ensure that it is
placed on the stack before local variables.

##### Arguments:

The `llvm.stackprotector` intrinsic requires two pointer arguments. The
first argument is the value loaded from the stack guard
`@__stack_chk_guard`. The second variable is an `alloca` that has enough
space to hold the value of the guard.

##### Semantics:

This intrinsic causes the prologue/epilogue inserter to force the
position of the `AllocaInst` stack slot to be before local variables on
the stack. This is to ensure that if a local variable on the stack is
overwritten, it will destroy the value of the guard. When the function
exits, the guard on the stack is checked against the original guard by
`llvm.stackprotectorcheck`. If they are different, then
`llvm.stackprotectorcheck` causes the program to abort by calling the
`__stack_chk_fail()` function.

#### \'`llvm.stackguard`\' Intrinsic

##### Syntax:

    declare i8* @llvm.stackguard()

##### Overview:

The `llvm.stackguard` intrinsic returns the system stack guard value.

It should not be generated by frontends, since it is only for internal
usage. The reason why we create this intrinsic is that we still support
IR form Stack Protector in FastISel.

##### Arguments:

None.

##### Semantics:

On some platforms, the value returned by this intrinsic remains
unchanged between loads in the same thread. On other platforms, it
returns the same global variable value, if any, e.g.
`@__stack_chk_guard`.

Currently some platforms have IR-level customized stack guard loading
(e.g. X86 Linux) that is not handled by `llvm.stackguard()`, while they
should be in the future.

#### \'`llvm.objectsize`\' Intrinsic

##### Syntax:

    declare i32 @llvm.objectsize.i32(i8* <object>, i1 <min>, i1 <nullunknown>)
    declare i64 @llvm.objectsize.i64(i8* <object>, i1 <min>, i1 <nullunknown>)

##### Overview:

The `llvm.objectsize` intrinsic is designed to provide information to
the optimizers to determine at compile time whether a) an operation
(like memcpy) will overflow a buffer that corresponds to an object, or
b) that a runtime check for overflow isn\'t necessary. An object in this
context means an allocation of a specific class, structure, array, or
other object.

##### Arguments:

The `llvm.objectsize` intrinsic takes three arguments. The first
argument is a pointer to or into the `object`. The second argument
determines whether `llvm.objectsize` returns 0 (if true) or -1 (if
false) when the object size is unknown. The third argument controls how
`llvm.objectsize` acts when `null` in address space 0 is used as its
pointer argument. If it\'s `false`, `llvm.objectsize` reports 0 bytes
available when given `null`. Otherwise, if the `null` is in a non-zero
address space or if `true` is given for the third argument of
`llvm.objectsize`, we assume its size is unknown.

The second and third arguments only accept constants.

##### Semantics:

The `llvm.objectsize` intrinsic is lowered to a constant representing
the size of the object concerned. If the size cannot be determined at
compile time, `llvm.objectsize` returns `i32/i64 -1 or 0` (depending on
the `min` argument).

#### \'`llvm.expect`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use `llvm.expect` on any
integer bit width.

    declare i1 @llvm.expect.i1(i1 <val>, i1 <expected_val>)
    declare i32 @llvm.expect.i32(i32 <val>, i32 <expected_val>)
    declare i64 @llvm.expect.i64(i64 <val>, i64 <expected_val>)

##### Overview:

The `llvm.expect` intrinsic provides information about expected (the
most probable) value of `val`, which can be used by optimizers.

##### Arguments:

The `llvm.expect` intrinsic takes two arguments. The first argument is a
value. The second argument is an expected value, this needs to be a
constant value, variables are not allowed.

##### Semantics:

This intrinsic is lowered to the `val`.

#### \'`llvm.assume`\' Intrinsic {#int_assume}

##### Syntax:

    declare void @llvm.assume(i1 %cond)

##### Overview:

The `llvm.assume` allows the optimizer to assume that the provided
condition is true. This information can then be used in simplifying
other parts of the code.

##### Arguments:

The condition which the optimizer may assume is always true.

##### Semantics:

The intrinsic allows the optimizer to assume that the provided condition
is always true whenever the control flow reaches the intrinsic call. No
code is generated for this intrinsic, and instructions that contribute
only to the provided condition are not used for code generation. If the
condition is violated during execution, the behavior is undefined.

Note that the optimizer might limit the transformations performed on
values used by the `llvm.assume` intrinsic in order to preserve the
instructions only used to form the intrinsic\'s input argument. This
might prove undesirable if the extra information provided by the
`llvm.assume` intrinsic does not cause sufficient overall improvement in
code quality. For this reason, `llvm.assume` should not be used to
document basic mathematical invariants that the optimizer can otherwise
deduce or facts that are of little use to the optimizer.

#### \'`llvm.ssa_copy`\' Intrinsic {#int_ssa_copy}

##### Syntax:

    declare type @llvm.ssa_copy(type %operand) returned(1) readnone

##### Arguments:

The first argument is an operand which is used as the returned value.

##### Overview:

The `llvm.ssa_copy` intrinsic can be used to attach information to
operations by copying them and giving them new names. For example, the
PredicateInfo utility uses it to build Extended SSA form, and attach
various forms of information to operands that dominate specific uses. It
is not meant for general use, only for building temporary renaming forms
that require value splits at certain points.

#### \'`llvm.type.test`\' Intrinsic {#type.test}

##### Syntax:

    declare i1 @llvm.type.test(i8* %ptr, metadata %type) nounwind readnone

##### Arguments:

The first argument is a pointer to be tested. The second argument is a
metadata object representing a
`type identifier <TypeMetadata>`{.interpreted-text role="doc"}.

##### Overview:

The `llvm.type.test` intrinsic tests whether the given pointer is
associated with the given type identifier.

#### \'`llvm.type.checked.load`\' Intrinsic

##### Syntax:

    declare {i8*, i1} @llvm.type.checked.load(i8* %ptr, i32 %offset, metadata %type) argmemonly nounwind readonly

##### Arguments:

The first argument is a pointer from which to load a function pointer.
The second argument is the byte offset from which to load the function
pointer. The third argument is a metadata object representing a
`type identifier
<TypeMetadata>`{.interpreted-text role="doc"}.

##### Overview:

The `llvm.type.checked.load` intrinsic safely loads a function pointer
from a virtual table pointer using type metadata. This intrinsic is used
to implement control flow integrity in conjunction with virtual call
optimization. The virtual call optimization pass will optimize away
`llvm.type.checked.load` intrinsics associated with devirtualized calls,
thereby removing the type check in cases where it is not needed to
enforce the control flow integrity constraint.

If the given pointer is associated with a type metadata identifier, this
function returns true as the second element of its return value. (Note
that the function may also return true if the given pointer is not
associated with a type metadata identifier.) If the function\'s return
value\'s second element is true, the following rules apply to the first
element:

-   If the given pointer is associated with the given type metadata
    identifier, it is the function pointer loaded from the given byte
    offset from the given pointer.
-   If the given pointer is not associated with the given type metadata
    identifier, it is one of the following (the choice of which is
    unspecified):
    1.  The function pointer that would have been loaded from an
        arbitrarily chosen (through an unspecified mechanism) pointer
        associated with the type metadata.
    2.  If the function has a non-void return type, a pointer to a
        function that returns an unspecified value without causing side
        effects.

If the function\'s return value\'s second element is false, the value of
the first element is undefined.

#### \'`llvm.donothing`\' Intrinsic

##### Syntax:

    declare void @llvm.donothing() nounwind readnone

##### Overview:

The `llvm.donothing` intrinsic doesn\'t perform any operation. It\'s one
of only three intrinsics (besides `llvm.experimental.patchpoint` and
`llvm.experimental.gc.statepoint`) that can be called with an invoke
instruction.

##### Arguments:

None.

##### Semantics:

This intrinsic does nothing, and it\'s removed by optimizers and ignored
by codegen.

#### \'`llvm.experimental.deoptimize`\' Intrinsic

##### Syntax:

    declare type @llvm.experimental.deoptimize(...) [ "deopt"(...) ]

##### Overview:

This intrinsic, together with `deoptimization operand bundles
<deopt_opbundles>`{.interpreted-text role="ref"}, allow frontends to
express transfer of control and frame-local state from the currently
executing (typically more specialized, hence faster) version of a
function into another (typically more generic, hence slower) version.

In languages with a fully integrated managed runtime like Java and
JavaScript this intrinsic can be used to implement \"uncommon trap\" or
\"side exit\" like functionality. In unmanaged languages like C and C++,
this intrinsic can be used to represent the slow paths of specialized
functions.

##### Arguments:

The intrinsic takes an arbitrary number of arguments, whose meaning is
decided by the
`lowering strategy<deoptimize_lowering>`{.interpreted-text role="ref"}.

##### Semantics:

The `@llvm.experimental.deoptimize` intrinsic executes an attached
deoptimization continuation (denoted using a `deoptimization
operand bundle <deopt_opbundles>`{.interpreted-text role="ref"}) and
returns the value returned by the deoptimization continuation. Defining
the semantic properties of the continuation itself is out of scope of
the language reference \--as far as LLVM is concerned, the
deoptimization continuation can invoke arbitrary side effects, including
reading from and writing to the entire heap.

Deoptimization continuations expressed using `"deopt"` operand bundles
always continue execution to the end of the physical frame containing
them, so all calls to `@llvm.experimental.deoptimize` must be in \"tail
position\":

> -   `@llvm.experimental.deoptimize` cannot be invoked.
> -   The call must immediately precede a
>     `ret <i_ret>`{.interpreted-text role="ref"} instruction.
> -   The `ret` instruction must return the value produced by the
>     `@llvm.experimental.deoptimize` call if there is one, or void.

Note that the above restrictions imply that the return type for a call
to `@llvm.experimental.deoptimize` will match the return type of its
immediate caller.

The inliner composes the `"deopt"` continuations of the caller into the
`"deopt"` continuations present in the inlinee, and also updates calls
to this intrinsic to return directly from the frame of the function it
inlined into.

All declarations of `@llvm.experimental.deoptimize` must share the same
calling convention.

##### Lowering: {#deoptimize_lowering}

Calls to `@llvm.experimental.deoptimize` are lowered to calls to the
symbol `__llvm_deoptimize` (it is the frontend\'s responsibility to
ensure that this symbol is defined). The call arguments to
`@llvm.experimental.deoptimize` are lowered as if they were formal
arguments of the specified types, and not as varargs.

#### \'`llvm.experimental.guard`\' Intrinsic

##### Syntax:

    declare void @llvm.experimental.guard(i1, ...) [ "deopt"(...) ]

##### Overview:

This intrinsic, together with `deoptimization operand bundles
<deopt_opbundles>`{.interpreted-text role="ref"}, allows frontends to
express guards or checks on optimistic assumptions made during
compilation. The semantics of `@llvm.experimental.guard` is defined in
terms of `@llvm.experimental.deoptimize` \-- its body is defined to be
equivalent to:

``` {.text}
define void @llvm.experimental.guard(i1 %pred, <args...>) {
  %realPred = and i1 %pred, undef
  br i1 %realPred, label %continue, label %leave [, !make.implicit !{}]

leave:
  call void @llvm.experimental.deoptimize(<args...>) [ "deopt"() ]
  ret void

continue:
  ret void
}
```

with the optional `[, !make.implicit !{}]` present if and only if it is
present on the call site. For more details on `!make.implicit`, see
`FaultMaps`{.interpreted-text role="doc"}.

In words, `@llvm.experimental.guard` executes the attached `"deopt"`
continuation if (but **not** only if) its first argument is `false`.
Since the optimizer is allowed to replace the `undef` with an arbitrary
value, it can optimize guard to fail \"spuriously\", i.e. without the
original condition being false (hence the \"not only if\"); and this
allows for \"check widening\" type optimizations.

`@llvm.experimental.guard` cannot be invoked.

#### \'`llvm.experimental.widenable.condition`\' Intrinsic

##### Syntax:

    declare i1 @llvm.experimental.widenable.condition()

##### Overview:

This intrinsic represents a \"widenable condition\" which is boolean
expressions with the following property: whether this expression is
[true]{.title-ref} or [false]{.title-ref}, the program is correct and
well-defined.

Together with
`deoptimization operand bundles <deopt_opbundles>`{.interpreted-text
role="ref"}, `@llvm.experimental.widenable.condition` allows frontends
to express guards or checks on optimistic assumptions made during
compilation and represent them as branch instructions on special
conditions.

While this may appear similar in semantics to [undef]{.title-ref}, it is
very different in that an invocation produces a particular, singular
value. It is also intended to be lowered late, and remain available for
specific optimizations and transforms that can benefit from its special
properties.

##### Arguments:

None.

##### Semantics:

The intrinsic `@llvm.experimental.widenable.condition()` returns either
[true]{.title-ref} or [false]{.title-ref}. For each evaluation of a call
to this intrinsic, the program must be valid and correct both if it
returns [true]{.title-ref} and if it returns [false]{.title-ref}. This
allows transformation passes to replace evaluations of this intrinsic
with either value whenever one is beneficial.

When used in a branch condition, it allows us to choose between two
alternative correct solutions for the same problem, like in example
below:

``` {.text}
%cond = call i1 @llvm.experimental.widenable.condition()
br i1 %cond, label %solution_1, label %solution_2
```

> label %fast\_path:
>
> :   ; Apply memory-consuming but fast solution for a task.
>
> label %slow\_path:
>
> :   ; Cheap in memory but slow solution.
>
Whether the result of intrinsic\'s call is [true]{.title-ref} or
[false]{.title-ref}, it should be correct to pick either solution. We
can switch between them by replacing the result of
`@llvm.experimental.widenable.condition` with different [i1]{.title-ref}
expressions.

This is how it can be used to represent guards as widenable branches:

``` {.text}
block:
  ; Unguarded instructions
  call void @llvm.experimental.guard(i1 %cond, <args...>) ["deopt"(<deopt_args...>)]
  ; Guarded instructions
```

Can be expressed in an alternative equivalent form of explicit branch
using `@llvm.experimental.widenable.condition`:

``` {.text}
block:
  ; Unguarded instructions
  %widenable_condition = call i1 @llvm.experimental.widenable.condition()
  %guard_condition = and i1 %cond, %widenable_condition
  br i1 %guard_condition, label %guarded, label %deopt

guarded:
  ; Guarded instructions

deopt:
  call type @llvm.experimental.deoptimize(<args...>) [ "deopt"(<deopt_args...>) ]
```

So the block [guarded]{.title-ref} is only reachable when
[%cond]{.title-ref} is [true]{.title-ref}, and it should be valid to go
to the block [deopt]{.title-ref} whenever [%cond]{.title-ref} is
[true]{.title-ref} or [false]{.title-ref}.

`@llvm.experimental.widenable.condition` will never throw, thus it
cannot be invoked.

##### Guard widening:

When `@llvm.experimental.widenable.condition()` is used in condition of
a guard represented as explicit branch, it is legal to widen the
guard\'s condition with any additional conditions.

Guard widening looks like replacement of

``` {.text}
%widenable_cond = call i1 @llvm.experimental.widenable.condition()
%guard_cond = and i1 %cond, %widenable_cond
br i1 %guard_cond, label %guarded, label %deopt
```

with

``` {.text}
%widenable_cond = call i1 @llvm.experimental.widenable.condition()
%new_cond = and i1 %any_other_cond, %widenable_cond
%new_guard_cond = and i1 %cond, %new_cond
br i1 %new_guard_cond, label %guarded, label %deopt
```

for this branch. Here [%any\_other\_cond]{.title-ref} is an arbitrarily
chosen well-defined [i1]{.title-ref} value. By making guard widening, we
may impose stricter conditions on [guarded]{.title-ref} block and bail
to the deopt when the new condition is not met.

##### Lowering:

Default lowering strategy is replacing the result of call of
`@llvm.experimental.widenable.condition` with constant
[true]{.title-ref}. However it is always correct to replace it with any
other [i1]{.title-ref} value. Any pass can freely do it if it can
benefit from non-default lowering.

#### \'`llvm.load.relative`\' Intrinsic

##### Syntax:

    declare i8* @llvm.load.relative.iN(i8* %ptr, iN %offset) argmemonly nounwind readonly

##### Overview:

This intrinsic loads a 32-bit value from the address `%ptr + %offset`,
adds `%ptr` to that value and returns it. The constant folder
specifically recognizes the form of this intrinsic and the constant
initializers it may load from; if a loaded constant initializer is known
to have the form `i32 trunc(x - %ptr)`, the intrinsic call is folded to
`x`.

LLVM provides that the calculation of such a constant initializer will
not overflow at link time under the medium code model if `x` is an
`unnamed_addr` function. However, it does not provide this guarantee for
a constant initializer folded into a function body. This intrinsic can
be used to avoid the possibility of overflows when loading from such a
constant.

#### \'`llvm.sideeffect`\' Intrinsic

##### Syntax:

    declare void @llvm.sideeffect() inaccessiblememonly nounwind

##### Overview:

The `llvm.sideeffect` intrinsic doesn\'t perform any operation.
Optimizers treat it as having side effects, so it can be inserted into a
loop to indicate that the loop shouldn\'t be assumed to terminate (which
could potentially lead to the loop being optimized away entirely), even
if it\'s an infinite loop with no other side effects.

##### Arguments:

None.

##### Semantics:

This intrinsic actually does nothing, but optimizers must assume that it
has externally observable side effects.

#### \'`llvm.is.constant.*`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use llvm.is.constant with any
argument type.

    declare i1 @llvm.is.constant.i32(i32 %operand) nounwind readnone
    declare i1 @llvm.is.constant.f32(float %operand) nounwind readnone
    declare i1 @llvm.is.constant.TYPENAME(TYPE %operand) nounwind readnone

##### Overview:

The \'`llvm.is.constant`\' intrinsic will return true if the argument is
known to be a manifest compile-time constant. It is guaranteed to fold
to either true or false before generating machine code.

##### Semantics:

This intrinsic generates no code. If its argument is known to be a
manifest compile-time constant value, then the intrinsic will be
converted to a constant true value. Otherwise, it will be converted to a
constant false value.

In particular, note that if the argument is a constant expression which
refers to a global (the address of which *is* a constant, but not
manifest during the compile), then the intrinsic evaluates to false.

The result also intentionally depends on the result of optimization
passes \-- e.g., the result can change depending on whether a function
gets inlined or not. A function\'s parameters are obviously not
constant. However, a call like `llvm.is.constant.i32(i32 %param)` *can*
return true after the function is inlined, if the value passed to the
function parameter was a constant.

On the other hand, if constant folding is not run, it will never
evaluate to true, even in simple cases.

### Stack Map Intrinsics

LLVM provides experimental intrinsics to support runtime patching
mechanisms commonly desired in dynamic language JITs. These intrinsics
are described in `StackMaps`{.interpreted-text role="doc"}.

### Element Wise Atomic Memory Intrinsics

These intrinsics are similar to the standard library memory intrinsics
except that they perform memory transfer as a sequence of atomic memory
accesses.

#### \'`llvm.memcpy.element.unordered.atomic`\' Intrinsic {#int_memcpy_element_unordered_atomic}

##### Syntax:

This is an overloaded intrinsic. You can use
`llvm.memcpy.element.unordered.atomic` on any integer bit width and for
different address spaces. Not all targets support all bit widths
however.

    declare void @llvm.memcpy.element.unordered.atomic.p0i8.p0i8.i32(i8* <dest>,
                                                                     i8* <src>,
                                                                     i32 <len>,
                                                                     i32 <element_size>)
    declare void @llvm.memcpy.element.unordered.atomic.p0i8.p0i8.i64(i8* <dest>,
                                                                     i8* <src>,
                                                                     i64 <len>,
                                                                     i32 <element_size>)

##### Overview:

The \'`llvm.memcpy.element.unordered.atomic.*`\' intrinsic is a
specialization of the \'`llvm.memcpy.*`\' intrinsic. It differs in that
the `dest` and `src` are treated as arrays with elements that are
exactly `element_size` bytes, and the copy between buffers uses a
sequence of `unordered atomic <ordering>`{.interpreted-text role="ref"}
load/store operations that are a positive integer multiple of the
`element_size` in size.

##### Arguments:

The first three arguments are the same as they are in the
`@llvm.memcpy <int_memcpy>`{.interpreted-text role="ref"} intrinsic,
with the added constraint that `len` is required to be a positive
integer multiple of the `element_size`. If `len` is not a positive
integer multiple of `element_size`, then the behaviour of the intrinsic
is undefined.

`element_size` must be a compile-time constant positive power of two no
greater than target-specific atomic access size limit.

For each of the input pointers `align` parameter attribute must be
specified. It must be a power of two no less than the `element_size`.
Caller guarantees that both the source and destination pointers are
aligned to that boundary.

##### Semantics:

The \'`llvm.memcpy.element.unordered.atomic.*`\' intrinsic copies `len`
bytes of memory from the source location to the destination location.
These locations are not allowed to overlap. The memory copy is performed
as a sequence of load/store operations where each access is guaranteed
to be a multiple of `element_size` bytes wide and aligned at an
`element_size` boundary.

The order of the copy is unspecified. The same value may be read from
the source buffer many times, but only one write is issued to the
destination buffer per element. It is well defined to have concurrent
reads and writes to both source and destination provided those reads and
writes are unordered atomic when specified.

This intrinsic does not provide any additional ordering guarantees over
those provided by a set of unordered loads from the source location and
stores to the destination.

##### Lowering:

In the most general case call to the
\'`llvm.memcpy.element.unordered.atomic.*`\' is lowered to a call to the
symbol `__llvm_memcpy_element_unordered_atomic_*`. Where \'\*\' is
replaced with an actual element size.

Optimizer is allowed to inline memory copy when it\'s profitable to do
so.

#### \'`llvm.memmove.element.unordered.atomic`\' Intrinsic

##### Syntax:

This is an overloaded intrinsic. You can use
`llvm.memmove.element.unordered.atomic` on any integer bit width and for
different address spaces. Not all targets support all bit widths
however.

    declare void @llvm.memmove.element.unordered.atomic.p0i8.p0i8.i32(i8* <dest>,
                                                                      i8* <src>,
                                                                      i32 <len>,
                                                                      i32 <element_size>)
    declare void @llvm.memmove.element.unordered.atomic.p0i8.p0i8.i64(i8* <dest>,
                                                                      i8* <src>,
                                                                      i64 <len>,
                                                                      i32 <element_size>)

##### Overview:

The \'`llvm.memmove.element.unordered.atomic.*`\' intrinsic is a
specialization of the \'`llvm.memmove.*`\' intrinsic. It differs in that
the `dest` and `src` are treated as arrays with elements that are
exactly `element_size` bytes, and the copy between buffers uses a
sequence of `unordered atomic <ordering>`{.interpreted-text role="ref"}
load/store operations that are a positive integer multiple of the
`element_size` in size.

##### Arguments:

The first three arguments are the same as they are in the
`@llvm.memmove <int_memmove>`{.interpreted-text role="ref"} intrinsic,
with the added constraint that `len` is required to be a positive
integer multiple of the `element_size`. If `len` is not a positive
integer multiple of `element_size`, then the behaviour of the intrinsic
is undefined.

`element_size` must be a compile-time constant positive power of two no
greater than a target-specific atomic access size limit.

For each of the input pointers the `align` parameter attribute must be
specified. It must be a power of two no less than the `element_size`.
Caller guarantees that both the source and destination pointers are
aligned to that boundary.

##### Semantics:

The \'`llvm.memmove.element.unordered.atomic.*`\' intrinsic copies `len`
bytes of memory from the source location to the destination location.
These locations are allowed to overlap. The memory copy is performed as
a sequence of load/store operations where each access is guaranteed to
be a multiple of `element_size` bytes wide and aligned at an
`element_size` boundary.

The order of the copy is unspecified. The same value may be read from
the source buffer many times, but only one write is issued to the
destination buffer per element. It is well defined to have concurrent
reads and writes to both source and destination provided those reads and
writes are unordered atomic when specified.

This intrinsic does not provide any additional ordering guarantees over
those provided by a set of unordered loads from the source location and
stores to the destination.

##### Lowering:

In the most general case call to the
\'`llvm.memmove.element.unordered.atomic.*`\' is lowered to a call to
the symbol `__llvm_memmove_element_unordered_atomic_*`. Where \'\*\' is
replaced with an actual element size.

The optimizer is allowed to inline the memory copy when it\'s profitable
to do so.

#### \'`llvm.memset.element.unordered.atomic`\' Intrinsic {#int_memset_element_unordered_atomic}

##### Syntax:

This is an overloaded intrinsic. You can use
`llvm.memset.element.unordered.atomic` on any integer bit width and for
different address spaces. Not all targets support all bit widths
however.

    declare void @llvm.memset.element.unordered.atomic.p0i8.i32(i8* <dest>,
                                                                i8 <value>,
                                                                i32 <len>,
                                                                i32 <element_size>)
    declare void @llvm.memset.element.unordered.atomic.p0i8.i64(i8* <dest>,
                                                                i8 <value>,
                                                                i64 <len>,
                                                                i32 <element_size>)

##### Overview:

The \'`llvm.memset.element.unordered.atomic.*`\' intrinsic is a
specialization of the \'`llvm.memset.*`\' intrinsic. It differs in that
the `dest` is treated as an array with elements that are exactly
`element_size` bytes, and the assignment to that array uses uses a
sequence of `unordered atomic <ordering>`{.interpreted-text role="ref"}
store operations that are a positive integer multiple of the
`element_size` in size.

##### Arguments:

The first three arguments are the same as they are in the
`@llvm.memset <int_memset>`{.interpreted-text role="ref"} intrinsic,
with the added constraint that `len` is required to be a positive
integer multiple of the `element_size`. If `len` is not a positive
integer multiple of `element_size`, then the behaviour of the intrinsic
is undefined.

`element_size` must be a compile-time constant positive power of two no
greater than target-specific atomic access size limit.

The `dest` input pointer must have the `align` parameter attribute
specified. It must be a power of two no less than the `element_size`.
Caller guarantees that the destination pointer is aligned to that
boundary.

##### Semantics:

The \'`llvm.memset.element.unordered.atomic.*`\' intrinsic sets the
`len` bytes of memory starting at the destination location to the given
`value`. The memory is set with a sequence of store operations where
each access is guaranteed to be a multiple of `element_size` bytes wide
and aligned at an `element_size` boundary.

The order of the assignment is unspecified. Only one write is issued to
the destination buffer per element. It is well defined to have
concurrent reads and writes to the destination provided those reads and
writes are unordered atomic when specified.

This intrinsic does not provide any additional ordering guarantees over
those provided by a set of unordered stores to the destination.

##### Lowering:

In the most general case call to the
\'`llvm.memset.element.unordered.atomic.*`\' is lowered to a call to the
symbol `__llvm_memset_element_unordered_atomic_*`. Where \'\*\' is
replaced with an actual element size.

The optimizer is allowed to inline the memory assignment when it\'s
profitable to do so.

### Objective-C ARC Runtime Intrinsics

LLVM provides intrinsics that lower to Objective-C ARC runtime entry
points. LLVM is aware of the semantics of these functions, and optimizes
based on that knowledge. You can read more about the details of
Objective-C ARC
[here](https://clang.llvm.org/docs/AutomaticReferenceCounting.html).

#### \'`llvm.objc.autorelease`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.autorelease(i8*)

##### Lowering:

Lowers to a call to
[objc\_autorelease](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autorelease).

#### \'`llvm.objc.autoreleasePoolPop`\' Intrinsic

##### Syntax:

    declare void @llvm.objc.autoreleasePoolPop(i8*)

##### Lowering:

Lowers to a call to
[objc\_autoreleasePoolPop](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-autoreleasepoolpop-void-pool).

#### \'`llvm.objc.autoreleasePoolPush`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.autoreleasePoolPush()

##### Lowering:

Lowers to a call to
[objc\_autoreleasePoolPush](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-autoreleasepoolpush-void).

#### \'`llvm.objc.autoreleaseReturnValue`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.autoreleaseReturnValue(i8*)

##### Lowering:

Lowers to a call to
[objc\_autoreleaseReturnValue](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autoreleasereturnvalue).

#### \'`llvm.objc.copyWeak`\' Intrinsic

##### Syntax:

    declare void @llvm.objc.copyWeak(i8**, i8**)

##### Lowering:

Lowers to a call to
[objc\_copyWeak](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-copyweak-id-dest-id-src).

#### \'`llvm.objc.destroyWeak`\' Intrinsic

##### Syntax:

    declare void @llvm.objc.destroyWeak(i8**)

##### Lowering:

Lowers to a call to
[objc\_destroyWeak](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-destroyweak-id-object).

#### \'`llvm.objc.initWeak`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.initWeak(i8**, i8*)

##### Lowering:

Lowers to a call to
[objc\_initWeak](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-initweak).

#### \'`llvm.objc.loadWeak`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.loadWeak(i8**)

##### Lowering:

Lowers to a call to
[objc\_loadWeak](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-loadweak).

#### \'`llvm.objc.loadWeakRetained`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.loadWeakRetained(i8**)

##### Lowering:

Lowers to a call to
[objc\_loadWeakRetained](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-loadweakretained).

#### \'`llvm.objc.moveWeak`\' Intrinsic

##### Syntax:

    declare void @llvm.objc.moveWeak(i8**, i8**)

##### Lowering:

Lowers to a call to
[objc\_moveWeak](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-moveweak-id-dest-id-src).

#### \'`llvm.objc.release`\' Intrinsic

##### Syntax:

    declare void @llvm.objc.release(i8*)

##### Lowering:

Lowers to a call to
[objc\_release](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-release-id-value).

#### \'`llvm.objc.retain`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.retain(i8*)

##### Lowering:

Lowers to a call to
[objc\_retain](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retain).

#### \'`llvm.objc.retainAutorelease`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.retainAutorelease(i8*)

##### Lowering:

Lowers to a call to
[objc\_retainAutorelease](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautorelease).

#### \'`llvm.objc.retainAutoreleaseReturnValue`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.retainAutoreleaseReturnValue(i8*)

##### Lowering:

Lowers to a call to
[objc\_retainAutoreleaseReturnValue](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautoreleasereturnvalue).

#### \'`llvm.objc.retainAutoreleasedReturnValue`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.retainAutoreleasedReturnValue(i8*)

##### Lowering:

Lowers to a call to
[objc\_retainAutoreleasedReturnValue](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautoreleasedreturnvalue).

#### \'`llvm.objc.retainBlock`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.retainBlock(i8*)

##### Lowering:

Lowers to a call to
[objc\_retainBlock](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainblock).

#### \'`llvm.objc.storeStrong`\' Intrinsic

##### Syntax:

    declare void @llvm.objc.storeStrong(i8**, i8*)

##### Lowering:

Lowers to a call to
[objc\_storeStrong](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-storestrong-id-object-id-value).

#### \'`llvm.objc.storeWeak`\' Intrinsic

##### Syntax:

    declare i8* @llvm.objc.storeWeak(i8**, i8*)

##### Lowering:

Lowers to a call to
[objc\_storeWeak](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-storeweak).
