Intrinsic Global Variables
--------------------------

LLVM has a number of \"magic\" global variables that contain data that
affect code generation or other IR semantics. These are documented here.
All globals of this sort should have a section specified as
\"`llvm.metadata`\". This section and all globals that start with
\"`llvm.`\" are reserved for use by LLVM.

### The \'`llvm.used`\' Global Variable

The `@llvm.used` global is an array which has
`appending linkage <linkage_appending>`{.interpreted-text role="ref"}.
This array contains a list of pointers to named global variables,
functions and aliases which may optionally have a pointer cast formed of
bitcast or getelementptr. For example, a legal use of it is:

``` {.llvm}
@X = global i8 4
@Y = global i32 123

@llvm.used = appending global [2 x i8*] [
   i8* @X,
   i8* bitcast (i32* @Y to i8*)
], section "llvm.metadata"
```

If a symbol appears in the `@llvm.used` list, then the compiler,
assembler, and linker are required to treat the symbol as if there is a
reference to the symbol that it cannot see (which is why they have to be
named). For example, if a variable has internal linkage and no
references other than that from the `@llvm.used` list, it cannot be
deleted. This is commonly used to represent references from inline asms
and other things the compiler cannot \"see\", and corresponds to
\"`attribute((used))`\" in GNU C.

On some targets, the code generator must emit a directive to the
assembler or object file to prevent the assembler and linker from
molesting the symbol.

### The \'`llvm.compiler.used`\' Global Variable

The `@llvm.compiler.used` directive is the same as the `@llvm.used`
directive, except that it only prevents the compiler from touching the
symbol. On targets that support it, this allows an intelligent linker to
optimize references to the symbol without being impeded as it would be
by `@llvm.used`.

This is a rare construct that should only be used in rare circumstances,
and should not be exposed to source languages.

### The \'`llvm.global_ctors`\' Global Variable

``` {.llvm}
%0 = type { i32, void ()*, i8* }
@llvm.global_ctors = appending global [1 x %0] [%0 { i32 65535, void ()* @ctor, i8* @data }]
```

The `@llvm.global_ctors` array contains a list of constructor functions,
priorities, and an optional associated global or function. The functions
referenced by this array will be called in ascending order of priority
(i.e. lowest first) when the module is loaded. The order of functions
with the same priority is not defined.

If the third field is present, non-null, and points to a global variable
or function, the initializer function will only run if the associated
data from the current module is not discarded.

### The \'`llvm.global_dtors`\' Global Variable

``` {.llvm}
%0 = type { i32, void ()*, i8* }
@llvm.global_dtors = appending global [1 x %0] [%0 { i32 65535, void ()* @dtor, i8* @data }]
```

The `@llvm.global_dtors` array contains a list of destructor functions,
priorities, and an optional associated global or function. The functions
referenced by this array will be called in descending order of priority
(i.e. highest first) when the module is unloaded. The order of functions
with the same priority is not defined.

If the third field is present, non-null, and points to a global variable
or function, the destructor function will only run if the associated
data from the current module is not discarded.

