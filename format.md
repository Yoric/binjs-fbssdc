# About this document

This document attempts to document the format currently implemented by binjs-fbssdc, also known as "context 0.1".

This format is parameterized by the AST of the host language (as of this writing, the JavaScript ES6 AST, as
defined [here](https://github.com/binast/binjs-ref/blob/master/spec/es6.webidl).

# Compressed files

## Global structure

```
Stream ::= MagicHeader BrotliBody
```

### Note for future versions

Future versions may add footers containing data that is not necessary for compilation/interpretation
of the program, e.g. license, documentation, sourcemaps.

## Magic Header

The magic header serves to identify the format.

```
MagicHeader ::= "\x89BJS\r\n\0\n" FormatVersion
FormatVersion ::= varnum(2)
```

## Brotli content

With the exception of the header, the entire file is brotli-compressed.

```
BrotliBody ::= Brotli(Body)
Body ::= Prelude AST
```

Where `Brotli(...)` represents data that may be uncompressed by the
`brotli` command-line tool or any compatible library.

## Prelude

The prelude defines a dictionary of strings and a dictionary of huffman tables. The order of both is meaningful.

```
Prelude ::= StringPrelude HuffmanPrelude
```

### String dictionary

The Prelude string dictionary extends the Shared string dictionary with addition strings used in the file.
These strings may be identifier names, property keys or literal strings. String enums are **not** part of
the Prelude string dictionary, nor of the Shared string dictionar.

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

#### Note for future versions

Experience has shown that namespacing differently between literal strings, identifier names and property keys
could seriously improve compression. To be tested with this format.

### Huffman dictionaries

The Prelude Huffman dictionaries encodes a set of Huffman tables *for the host language
grammar*. Each possible combination of `(parent_tag, child_type)` is assigned one
Huffman table, which may be used to decode sequences of bits into a value of type
`child_type` for a field that is part of an interface with tag `parent_tag`.

The tables are written down in an order extracted from the grammar and define a model
`huffman_at: (parent tag, my type) -> HuffmanTable`.

The tables are written down in an order extracted from the grammar and define a model
`huffman_at: (parent tag, my type) -> HuffmanTable`.

```
HuffmanPrelude ::= HuffmanTable{N}           # The number of tables is extracted from the grammar.
HuffmanTable ::= HuffmanTableUnreachable     # Compression artifact. A table that needs to appear here but is never used.
             |   HuffmanTableOptimizedOne    # Optimization: A table with a single symbol.
             |   HuffmanTableExplicitSymbols # Used for strings, numbers.
             |   HuffmanTableIndexedSymbols  # Used for enums, booleans, sums of interfaces.
```

As all tables need to be expressed, regardless of whether they are used, a number of tables may
contain `HuffmanTableUnreachable`. If only one value is possible at a given `(parent tag, my type)`,
we collapse the Huffman table into a single symbol definition, and its encoding in the AST takes
0 bits.

Otherwise, we differentiate between tables of Indexed Symbols (used whenever all possible values
at this point form a simple, finite set, specified in the grammar as a sum of constants) and
tables of Explicit Symbols (used whenever the set of values at this point is extensible by
any file).

```
HuffmanTableUnreachable ::= 0x02
HuffmanTableOptimizedOne ::= 0x00 ExplicitSymbolData # Used for strings, numbers.
                         |   0x00 Index              # Used for enums, booleans, sums of interfaces.
HuffmanTableExplicitSymbols ::= 0x01 n=HuffmanTableLen BitLength{n} ExplicitSymbolData{n} # Only list the symbols actually used.
HuffmanTableIndexedSymbols ::= 0x01 BitLength{N}     # List all possible symbols. Number and order are extracted from the grammar.
HuffmanTableLen ::= varnum
Index ::= varnum
BitLength ::= u8   # Number of bits needed to decode this symbol.
```

An `Index` is an index in a list of well-known symbols (enums, booleans, sums of interfaces). The list is
extracted statically from the grammar.

FIXME: Specify the order of well-known symbols.

Strings are always represented as indices into the string dictionary.

FIXME: Specify how we interpret an index when there is both a Shared string dictionary and a Prelude string
dictionary.

```
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

Both `ExplicitSymbolStringIndex` and `ExplicitSymbolOptionalStringIndex` are indices in the list of strings.
The list is in the order specified by `StringPrelude`. In `ExplicitSymbolOptionalStringIndex`, if the result
is the non-0 value `n`, the actual index is `n - 1`.

#### Note for future versions

Experience has shown that nearly all instances of f64 are actually short
integers and may therefore be represented efficiently as `varnum`. To
be experimented.

## AST

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

### Note for future versions

We intend to experiment on splitting the String prelude to ensure that
we do not include in `RootNode` fragments that are only useful in lazy
parts, and which would therefore generally slow startup.


## Nodes

Nodes are stored as sequences of Huffman-encoded values. Note that the encoding uses
numerous distinct Huffman tables, rather than a single one. Each `(parent tag, value type)`
pair determines the Huffman table to be used to decode the next few bits in the sequence.

At each node, the `parent tag` is determined by decoding the parent node (see `Tag` below),
while the `value type` is specified by the language grammar.

```
RootNode ::= Value(ε)*
Node(parent) ::= t=Tag(parent) Value(t)*  # Number of values is specified by the grammar
Tag(parent) ::= Primitive(parent, TAG)
Value(parent) ::= ""                      # If grammar specifies that field is lazy
              |   Node(parent)            # If grammar specifies that field is an interface or sum of interfaces
              |   List(parent)            # If grammar specifies that field is a list
              |   Primitive(parent, U32)  # If grammar specifies that field is a u32
              |   Primitive(parent, I32)  # If grammar specifies that field is a i32
              |   Primitive(parent, F64)  # ...
              |   Primitive(parent, StringIndex)
              |   Primitive(parent, OptionalStringIndex)
List(parent) ::= n=ListLength(parent) Value(parent){n}
ListLength(parent) ::= Primitive(ListLength<parent>, U32) # List lengths are u32 values with a special parent
Primitive(parent, type) ::= bit*
```

In every instance of `Primitive(parent, type)`, we use the Huffman table defined as `huffman_at` (see above)
to both determine the number of bits to read and interpret these bits as a value of the corresponding `type`.

# Shared dictionaries

TBD