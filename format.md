# About this document

This document attempts to document the format currently implemented by binjs-fbssdc, also known as "context 0.1".

# Global structure

```
Stream ::= MagicHeader BrotliBody
```

# Magic Header

The magic header serves to identify the format.

```
MagicHeader ::= "\x89BJS\r\n\0\n" FormatVersion
FormatVersion ::= 2 as varnum
```

# Brotli content

With the exception of the header, the entire file is brotli-compressed.

```
BrotliBody ::= Brotli(Body)
Body ::= Prelude AST
```

Where `Brotli(...)` represents data that may be uncompressed by the
`brotli` command-line tool or any compatible library.

# Prelude

The prelude defines a dictionary of strings and a dictionary of huffman tables. The order of both is meaningful.

```
Prelude ::= StringPrelude HuffmanPrelude
```

## String dictionary

```
StringPrelude ::= n=NumberOfStrings StringDefinition{n}
NumberOfStrings ::= varnum
StringDefinition ::= NonZeroByte* ZeroByte
NonZeroByte ::= 0x02-0xFF
            |   0x01 0x00 # Interpreted as escaped 0x00
            |   0x01 0x01 # Interpreted as escaped 0x01
ZeroByte ::= 0x00
```

Strings are utf-8 encoded and any instance of `0x00` is replaced with `0x01 0x00`, any instance of `0x01` is replaced with `0x01 0x01`.



## Huffman dictionaries

```
HuffmanPrelude ::= HuffmanTable* # FIXME: How do we determine the number of tables?
HuffmanTable ::= HuffmanTableUnreachable     # Compression artifact. A table that needs to appear here but is never used.
             |   HuffmanTableOptimizedOne    # Optimization: A table with a single symbol.
             |   HuffmanTableExplicitSymbols # Used for strings, numbers.
             |   HuffmanTableIndexedSymbols  # Used for enums, booleans, sums of interfaces.
```

The tables are written down in an order extracted from the grammar and define a model
`huffman_at: (parent type, my type) -> HuffmanTable`.

FIXME: Specify how the order is extracted from the grammar.

```
HuffmanTableUnreachable ::= 0x02
HuffmanTableOptimizedOne ::= 0x00 ExplicitSymbolData # Used for strings, numbers.
                         |   0x00 Index              # Used for enums, booleans, sums of interfaces.
HuffmanTableExplicitSymbols ::= 0x01 n=HuffmanTableLen BitLength{n} ExplicitSymbolData{n} # Only list the symbols actually used.
HuffmanTableIndexedSymbols  ::= 0x01 BitLength* # List all symbols, in the order extracted from the grammar.
HuffmanTableLen ::= varnum
Index ::= varnum
BitLength ::= u8   # Number of bits needed to decode this symbol.
ExplicitSymbolData ::= ExplicitSymbolStringIndex
                   |   ExplicitSymbolOptionalStringIndex
                   |   ExplicitSymbolF64
                   |   ExplicitSymbolU32
                   |   ExplicitSymbolI32
ExplicitSymbolStringIndex ::= varnum
ExplicitSymbolOptionalStringIndex ::= 0x00
                                   |  n=varnum
                                       where n > 0
ExplicitSymbolF64 ::= f64 (IEEE 754, big endian)
ExplicitSymbolU32 ::= u32 (big endian)
ExplicitSymbolI32 ::= i32 (big endian)
```

An `Index` is an index in a list of well-known symbols (enums, booleans, sums of interfaces). The list is
extracted statically from the grammar.

FIXME: Specify the order of well-known symbols.

Both `ExplicitSymbolStringIndex` and `ExplicitSymbolOptionalStringIndex` are indices in the list of strings.
The list is in the order specified by `StringPrelude`. In `ExplicitSymbolOptionalStringIndex`, if the result
is the non-0 value `n`, the actual index is `n - 1`.

# AST

AST definitions are recursive. Any AST definition may itself contain further definitions,
used to represent lazy functions.

```
AST ::= RootNode n=NumberOfLazyParts LazyPartByteLen{n} LazyAST{n}
NumerOfLazyParts ::= varnum
LazyPartByteLen ::= varnum
LazyAST ::= Node
```

In the definition of `AST`, for each `i`, `LazyPartByteLen[i]` represents the number
of bytes used to store the item of the sub-ast `LazyAST[i]`.

# Nodes

Nodes are stored as sequences of Huffman-encoded values. Note that the encoding uses
numerous distinct Huffman tables. Each `(parent tag, value type)` pair determines the
Huffman table to be used to decode the next few bits in the sequence.

```
RootNode ::= Value(ε)*
Node(parent) ::= t=Tag(parent) Field(t)*
Tag(parent) ::= Primitive(parent, TAG)
Value(parent) ::= ""                      # If field is lazy
              |   Node(parent)            # If field is an interface or sum of interfaces
              |   List(parent)            # If field is a list
              |   Primitive(parent, U32)  # If field is a u32
              |   Primitive(parent, I32)  # If field is a i32
              |   Primitive(parent, F64)  # ...
              |   Primitive(parent, StringIndex)
              |   Primitive(parent, OptionalStringIndex)
List(parent) ::= ListLength(parent) Value(parent)*
ListLength(parent) ::= Primitive(ListLength<parent>, U32) # List lengths are u32 values with a special parent
Primitive(parent, type) ::= bit*
```

In every instance of `Primitive(parent, type)`, we use the Huffman table defined as `huffman_at` (see above)
to both determine the number of bits to read and interpret these bits as a value of the corresponding `type`.