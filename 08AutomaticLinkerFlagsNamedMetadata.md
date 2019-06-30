Automatic Linker Flags Named Metadata
-------------------------------------

Some targets support embedding flags to the linker inside individual
object files. Typically this is used in conjunction with language
extensions which allow source files to explicitly declare the libraries
they depend on, and have these automatically be transmitted to the
linker via object files.

These flags are encoded in the IR using named metadata with the name
`!llvm.linker.options`. Each operand is expected to be a metadata node
which should be a list of other metadata nodes, each of which should be
a list of metadata strings defining linker options.

For example, the following metadata section specifies two separate sets
of linker options, presumably to link against `libz` and the `Cocoa`
framework:

    !0 = !{ !"-lz" },
    !1 = !{ !"-framework", !"Cocoa" } } }
    !llvm.linker.options = !{ !0, !1 }

The metadata encoding as lists of lists of options, as opposed to a
collapsed list of options, is chosen so that the IR encoding can use
multiple option strings to specify e.g., a single library, while still
having that specifier be preserved as an atomic element that can be
recognized by a target specific assembly writer or object file emitter.

Each individual option is required to be either a valid option for the
target\'s linker, or an option that is reserved by the target specific
assembly writer or object file emitter. No other aspect of these options
is defined by the IR.

