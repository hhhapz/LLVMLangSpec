
ThinLTO Summary
---------------

Compiling with [ThinLTO](https://clang.llvm.org/docs/ThinLTO.html)
causes the building of a compact summary of the module that is emitted
into the bitcode. The summary is emitted into the LLVM assembly and
identified in syntax by a caret (\'`^`\').

The summary is parsed into a bitcode output, along with the Module IR,
via the \"`llvm-as`\" tool. Tools that parse the Module IR for the
purposes of optimization (e.g. \"`clang -x ir`\" and \"`opt`\"), will
ignore the summary entries (just as they currently ignore summary
entries in a bitcode input file).

Eventually, the summary will be parsed into a ModuleSummaryIndex object
under the same conditions where summary index is currently built from
bitcode. Specifically, tools that test the Thin Link portion of a
ThinLTO compile (i.e. llvm-lto and llvm-lto2), or when parsing a
combined index for a distributed ThinLTO backend via clang\'s
\"`-fthinlto-index=<>`\" flag (this part is not yet implemented, use
llvm-as to create a bitcode object before feeding into thin link tools
for now).

There are currently 3 types of summary entries in the LLVM assembly:
`module paths<module_path_summary>`{.interpreted-text role="ref"},
`global values<gv_summary>`{.interpreted-text role="ref"}, and
`type identifiers<typeid_summary>`{.interpreted-text role="ref"}.

### Module Path Summary Entry

Each module path summary entry lists a module containing global values
included in the summary. For a single IR module there will be one such
entry, but in a combined summary index produced during the thin link,
there will be one module path entry per linked module with summary.

Example:

``` {.text}
^0 = module: (path: "/path/to/file.o", hash: (2468601609, 1329373163, 1565878005, 638838075, 3148790418))
```

The `path` field is a string path to the bitcode file, and the `hash`
field is the 160-bit SHA-1 hash of the IR bitcode contents, used for
incremental builds and caching.

### Global Value Summary Entry

Each global value summary entry corresponds to a global value defined or
referenced by a summarized module.

Example:

``` {.text}
^4 = gv: (name: "f"[, summaries: (Summary)[, (Summary)]*]?) ; guid = 14740650423002898831
```

For declarations, there will not be a summary list. For definitions, a
global value will contain a list of summaries, one per module containing
a definition. There can be multiple entries in a combined summary index
for symbols with weak linkage.

Each `Summary` format will depend on whether the global value is a
`function<function_summary>`{.interpreted-text role="ref"},
`variable<variable_summary>`{.interpreted-text role="ref"}, or
`alias<alias_summary>`{.interpreted-text role="ref"}.

#### Function Summary

If the global value is a function, the `Summary` entry will look like:

``` {.text}
function: (module: ^0, flags: (linkage: external, notEligibleToImport: 0, live: 0, dsoLocal: 0), insts: 2[, FuncFlags]?[, Calls]?[, TypeIdInfo]?[, Refs]?
```

The `module` field includes the summary entry id for the module
containing this definition, and the `flags` field contains information
such as the linkage type, a flag indicating whether it is legal to
import the definition, whether it is globally live and whether the
linker resolved it to a local definition (the latter two are populated
during the thin link). The `insts` field contains the number of IR
instructions in the function. Finally, there are several optional
fields: `FuncFlags<funcflags_summary>`{.interpreted-text role="ref"},
`Calls<calls_summary>`{.interpreted-text role="ref"},
`TypeIdInfo<typeidinfo_summary>`{.interpreted-text role="ref"},
`Refs<refs_summary>`{.interpreted-text role="ref"}.

#### Global Variable Summary

If the global value is a variable, the `Summary` entry will look like:

``` {.text}
variable: (module: ^0, flags: (linkage: external, notEligibleToImport: 0, live: 0, dsoLocal: 0)[, Refs]?
```

The variable entry contains a subset of the fields in a
`function summary <function_summary>`{.interpreted-text role="ref"}, see
the descriptions there.

#### Alias Summary

If the global value is an alias, the `Summary` entry will look like:

``` {.text}
alias: (module: ^0, flags: (linkage: external, notEligibleToImport: 0, live: 0, dsoLocal: 0), aliasee: ^2)
```

The `module` and `flags` fields are as described for a
`function summary <function_summary>`{.interpreted-text role="ref"}. The
`aliasee` field contains a reference to the global value summary entry
of the aliasee.

#### Function Flags

The optional `FuncFlags` field looks like:

``` {.text}
funcFlags: (readNone: 0, readOnly: 0, noRecurse: 0, returnDoesNotAlias: 0)
```

If unspecified, flags are assumed to hold the conservative `false` value
of `0`.

#### Calls

The optional `Calls` field looks like:

``` {.text}
calls: ((Callee)[, (Callee)]*)
```

where each `Callee` looks like:

``` {.text}
callee: ^1[, hotness: None]?[, relbf: 0]?
```

The `callee` refers to the summary entry id of the callee. At most one
of `hotness` (which can take the values `Unknown`, `Cold`, `None`,
`Hot`, and `Critical`), and `relbf` (which holds the integer branch
frequency relative to the entry frequency, scaled down by 2\^8) may be
specified. The defaults are `Unknown` and `0`, respectively.

#### Refs

The optional `Refs` field looks like:

``` {.text}
refs: ((Ref)[, (Ref)]*)
```

where each `Ref` contains a reference to the summary id of the
referenced value (e.g. `^1`).

#### TypeIdInfo

The optional `TypeIdInfo` field, used for [Control Flow
Integrity](http://clang.llvm.org/docs/ControlFlowIntegrity.html), looks
like:

``` {.text}
typeIdInfo: [(TypeTests)]?[, (TypeTestAssumeVCalls)]?[, (TypeCheckedLoadVCalls)]?[, (TypeTestAssumeConstVCalls)]?[, (TypeCheckedLoadConstVCalls)]?
```

These optional fields have the following forms:

##### TypeTests

``` {.text}
typeTests: (TypeIdRef[, TypeIdRef]*)
```

Where each `TypeIdRef` refers to a
`type id<typeid_summary>`{.interpreted-text role="ref"} by summary id or
`GUID`.

##### TypeTestAssumeVCalls

``` {.text}
typeTestAssumeVCalls: (VFuncId[, VFuncId]*)
```

Where each VFuncId has the format:

``` {.text}
vFuncId: (TypeIdRef, offset: 16)
```

Where each `TypeIdRef` refers to a
`type id<typeid_summary>`{.interpreted-text role="ref"} by summary id or
`GUID` preceeded by a `guid:` tag.

##### TypeCheckedLoadVCalls

``` {.text}
typeCheckedLoadVCalls: (VFuncId[, VFuncId]*)
```

Where each VFuncId has the format described for `TypeTestAssumeVCalls`.

##### TypeTestAssumeConstVCalls

``` {.text}
typeTestAssumeConstVCalls: (ConstVCall[, ConstVCall]*)
```

Where each ConstVCall has the format:

``` {.text}
(VFuncId, args: (Arg[, Arg]*))
```

and where each VFuncId has the format described for
`TypeTestAssumeVCalls`, and each Arg is an integer argument number.

##### TypeCheckedLoadConstVCalls

``` {.text}
typeCheckedLoadConstVCalls: (ConstVCall[, ConstVCall]*)
```

Where each ConstVCall has the format described for
`TypeTestAssumeConstVCalls`.

### Type ID Summary Entry

Each type id summary entry corresponds to a type identifier resolution
which is generated during the LTO link portion of the compile when
building with [Control Flow
Integrity](http://clang.llvm.org/docs/ControlFlowIntegrity.html), so
these are only present in a combined summary index.

Example:

``` {.text}
^4 = typeid: (name: "_ZTS1A", summary: (typeTestRes: (kind: allOnes, sizeM1BitWidth: 7[, alignLog2: 0]?[, sizeM1: 0]?[, bitMask: 0]?[, inlineBits: 0]?)[, WpdResolutions]?)) ; guid = 7004155349499253778
```

The `typeTestRes` gives the type test resolution `kind` (which may be
`unsat`, `byteArray`, `inline`, `single`, or `allOnes`), and the
`size-1` bit width. It is followed by optional flags, which default to
0, and an optional WpdResolutions (whole program devirtualization
resolution) field that looks like:

``` {.text}
wpdResolutions: ((offset: 0, WpdRes)[, (offset: 1, WpdRes)]*
```

where each entry is a mapping from the given byte offset to the
whole-program devirtualization resolution WpdRes, that has one of the
following formats:

``` {.text}
wpdRes: (kind: branchFunnel)
wpdRes: (kind: singleImpl, singleImplName: "_ZN1A1nEi")
wpdRes: (kind: indir)
```

Additionally, each wpdRes has an optional `resByArg` field, which
describes the resolutions for calls with all constant integer arguments:

``` {.text}
resByArg: (ResByArg[, ResByArg]*)
```

where ResByArg is:

``` {.text}
args: (Arg[, Arg]*), byArg: (kind: UniformRetVal[, info: 0][, byte: 0][, bit: 0])
```

Where the `kind` can be `Indir`, `UniformRetVal`, `UniqueRetVal` or
`VirtualConstProp`. The `info` field is only used if the kind is
`UniformRetVal` (indicates the uniform return value), or `UniqueRetVal`
(holds the return value associated with the unique vtable (0 or 1)). The
`byte` and `bit` fields are only used if the target does not support the
use of absolute symbols to store constants.

