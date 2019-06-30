Module Flags Metadata
---------------------

Information about the module as a whole is difficult to convey to
LLVM\'s subsystems. The LLVM IR isn\'t sufficient to transmit this
information. The `llvm.module.flags` named metadata exists in order to
facilitate this. These flags are in the form of key / value pairs \-\--
much like a dictionary \-\-- making it easy for any subsystem who cares
about a flag to look it up.

The `llvm.module.flags` metadata contains a list of metadata triplets.
Each triplet has the following form:

-   The first element is a *behavior* flag, which specifies the behavior
    when two (or more) modules are merged together, and it encounters
    two (or more) metadata with the same ID. The supported behaviors are
    described below.
-   The second element is a metadata string that is a unique ID for the
    metadata. Each module may only have one flag entry for each unique
    ID (not including entries with the **Require** behavior).
-   The third element is the value of the flag.

When two (or more) modules are merged together, the resulting
`llvm.module.flags` metadata is the union of the modules\' flags. That
is, for each unique metadata ID string, there will be exactly one entry
in the merged modules `llvm.module.flags` metadata table, and the value
for that entry will be determined by the merge behavior flag, as
described below. The only exception is that entries with the *Require*
behavior are always preserved.

The following behaviors are supported:

| Value | Behavior                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|-------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1     |  **Error**  Emits an error if two values disagree, otherwise the resulting value is that of the operands.                                               
| 2     |   **Warning** Emits a warning if two values disagree. The result value will be the operand for the flag from the first module being linked.             
| 3     |  **Require** Adds a requirement that another module flag be present and have a specified value after linking is performed. The value must be a metadata pair, where the first element of the pair is the ID of the module flag to be restricted, and the second element of the pair is the value the module flag should be restricted to. This behavior can be used to restrict the allowable results (via triggering of an error) of linking IDs with the **Override** behavior. |
| 4     |  **Override**  Uses the specified value, regardless of the behavior or value of the other module. If both modules specify Override, but the values differ, an error will be emitted.                                                                                                                                                                                                                                                                                              |
| 5     |  **Append**  Appends the two values, which are required to be metadata nodes.                                                                                                                                                                                                                                                                                                                                                                                                     |
| 6     |  **AppendUnique**   Appends the two values, which are required to be metadata nodes. However, duplicate entries in the second list are dropped during the append operation.                                                                                                                                                                                                                                                                                                       |
| 7     |  **Max**  Takes the max of the two values, which are required to be integers.                                                                                                                                                                                                                                                                                                                                                                                                     |

It is an error for a particular unique flag ID to have multiple
behaviors, except in the case of **Require** (which adds restrictions on
another metadata value) or **Override**.

An example of module flags:

``` {.llvm}
!0 = !{ i32 1, !"foo", i32 1 }
!1 = !{ i32 4, !"bar", i32 37 }
!2 = !{ i32 2, !"qux", i32 42 }
!3 = !{ i32 3, !"qux",
  !{
    !"foo", i32 1
  }
}
!llvm.module.flags = !{ !0, !1, !2, !3 }
```

-   Metadata `!0` has the ID `!"foo"` and the value \'1\'. The behavior
    if two or more `!"foo"` flags are seen is to emit an error if their
    values are not equal.

-   Metadata `!1` has the ID `!"bar"` and the value \'37\'. The behavior
    if two or more `!"bar"` flags are seen is to use the value \'37\'.

-   Metadata `!2` has the ID `!"qux"` and the value \'42\'. The behavior
    if two or more `!"qux"` flags are seen is to emit a warning if their
    values are not equal.

-   Metadata `!3` has the ID `!"qux"` and the value:

        !{ !"foo", i32 1 }

    The behavior is to emit an error if the `llvm.module.flags` does not
    contain a flag with the ID `!"foo"` that has the value \'1\' after
    linking is performed.

### Objective-C Garbage Collection Module Flags Metadata

On the Mach-O platform, Objective-C stores metadata about garbage
collection in a special section called \"image info\". The metadata
consists of a version number and a bitmask specifying what types of
garbage collection are supported (if any) by the file. If two or more
modules are linked together their garbage collection metadata needs to
be merged rather than appended together.

The Objective-C garbage collection module flags metadata consists of the
following key-value pairs:

| Key                               | Value                                                   |
|-----------------------------------|---------------------------------------------------------|
| `Objective-C Version`             | **\[Required\]** \-\-- The Objective-C ABI version.     |
|                                   |   Valid values are 1 and 2.                             |
|                                   |                                                         |    
| `Objective-C Image Info Version`  | **\[Required\]** \-\-- The version of the image info    |
|                                   |   section. Currently always 0.                          |
|                                   |                                                         |
| `Objective-C Image Info Section`  | **\[Required\]** \-\-- The section to place the         |
|                                   |   metadata. Valid values are                            |    
|                                   |   `"__OBJC, __image_info, regular"` for Objective-C ABI |    
|                                   |   version 1, and                                        |        
|                                   |   `"__DATA,__objc_imageinfo, regular, no_dead_strip"`   |
|                                   |   for Objective-C ABI version 2.                        |    
|                                   |                                                         |        
| `Objective-C Garbage Collection`  | **\[Required\]** \-\-- Specifies whether garbage        |    
|                                   |   collection is supported or not. Valid values are 0,   |        
|                                   |   for no garbage collection, and 2, for garbage         |            
|                                   |   collection supported.                                 |                
|                                   |                                                         |                
| `Objective-C GC Only`             | **\[Optional\]** \-\-- Specifies that only garbage      |            
|                                   |   collection is supported. If present, its value must   |                
|                                   |   be 6. This flag requires that the                     |                    
|                                   |   `Objective-C Garbage Collection` flag have the value  |                

Some important flag interactions:

-   If a module with `Objective-C Garbage Collection` set to 0 is merged
    with a module with `Objective-C Garbage Collection` set to 2, then
    the resulting module has the `Objective-C Garbage Collection` flag
    set to 0.
-   A module with `Objective-C Garbage Collection` set to 0 cannot be
    merged with a module with `Objective-C GC Only` set to 6.

### C type width Module Flags Metadata

The ARM backend emits a section into each generated object file
describing the options that it was compiled with (in a
compiler-independent way) to prevent linking incompatible objects, and
to allow automatic library selection. Some of these options are not
visible at the IR level, namely wchar\_t width and enum width.

To pass this information to the backend, these options are encoded in
module flags metadata, using the following key-value pairs:

| Key                | Value                                           |
|--------------------|-------------------------------------------------|
| short\_wchar       | -   0 \-\-- sizeof(wchar\_t) == 4               |
|                    | -   1 \-\-- sizeof(wchar\_t) == 2               |
|                    |                                                 |
|                    |                                                 |
| short\_enum        | -   0 \-\-- Enums are at least as large as an   |
|                    |     `int`.                                      |
|                    | -   1 \-\-- Enums are stored in the smallest    |
|                    |     integer type which can represent all of its |
|                    |     values.                                     |

For example, the following metadata section specifies that the module
was compiled with a `wchar_t` width of 4 bytes, and the underlying type
of an enum is the smallest type which can represent all of its values:

    !llvm.module.flags = !{!0, !1}
    !0 = !{i32 1, !"short_wchar", i32 1}
    !1 = !{i32 1, !"short_enum", i32 0}

