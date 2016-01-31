# typed-format
Experimental: A simple and complete schema language and a compact binary format for data interchange.


## Schema language

The schema language defines two kinds of types: The built-in type `bytes` (the only, built-in type) which represents a sequence of bytes; and user-defined types. The user defined types are algebraic datatypes, letting you express record types, eg. "this field AND this field AND this field..." as well as sum types, eg. "this sort of value OR this sort of value OR this sort of value...".

As an example, the following user-defined type `Person` has a text field `name` and an integer field `age`:

    type Person(String name, Int age)

All type definitions follow the structure "there is a type of this name, with these generics, that can be constructed by one of the following constructors, which has the following fields". The `Person` definition is simply syntactic sugar for a type with exacly one constructor:

    type Person {
        Person(String name, Int age)
    }

Values of the following type `Bool` is either `False` or `True`:

    type Bool {
        False
        True
    }

The following type describes an optional value. For example, an optional integer could be `None`, indicating no value, or `Some(42)`, indicating a value of 42:

    type Option<T> {
        None
        Some(T value)
    }

As is evident from the example above, generic types are supported. Recursive types are also supported, and this can be used to define a singly-linked list, which is a great fit for data interchange:

    type List<T> {
        Link(T head, List<T> tail)
        Empty
    }

In the above example, a list of three numbers 1-3 could be written in pseudo-code as `Link(1, Link(2, Link(3, Empty))`. In practice, the native representation of any given user-defined type can be customized for each target language. In many languages, the elements of such a list would be parsed into an array, rather than a linked list.

All the types that are called "primitive" in other languages can be defined as a user-defined type wrapping a byte sequence. For example, the optional standard library `common.ti` contains a `String` type with a field `utf8` that, as the name implies, contains UTF-8-encoded text:

    type String(bytes utf8)

To ease interopability, the optional standard library `common.ti` only defines two numeric types: A 64 bit integer and a 64 bit floating point type.

    type Int(bytes i64)
    type Float(bytes f64)

The binary format will make sure small values are represented in a compact fashion, and big values with minimal overhead. The in-memory representation is as always up to the target language. It's trivially easy to define other numeric types.

It's possible to import schemas from URLs, for use in the local schema, either qualified or unqualified (eg. leave out the `C` and `C.`):

    import C "https://raw.githubusercontent.com/Ahnfelt/typed-format/master/common.ti"
    type Author(C.List<C.String> bookTitles)

The schema language is encoded in ASCII and has the following grammar, where `UPPER` is an upper case letter (A-Z) followed by zero or more letters and digits (0-9), and where `LOWER` is as `UPPER`, except that `LOWER` starts with a lower case letter (a-z). The `URL` begins with a double quote (") and continues until the next double quote. Whitespace can be used to separate tokens, but is otherwise ignored. Comments begin with a // and continues to the end of the line, and are ignored.

    schema     ::= import* definition*
    import     ::= "import" UPPER? URL
    definition ::= "type" UPPER generics? body
    body       ::= fields | "{" (UPPER fields?)* "}"
    fields     ::= "(" (type LOWER ("," type LOWER)*)? ")"
    generics   ::= "<" UPPER ("," UPPER)* ">"
    type       ::= "bytes" | custom
    custom     ::= (UPPER ".")? UPPER ("<" custom ("," custom)* ">")?


## Binary format

The binary format is a compact representation of values, and requires a schema to encode and decode. The format is designed to allow for tiny implementations and efficient encoding and decoding. Many common values fit into a single byte. The longest byte sequence that can be represented is 4.294.967.295 bytes long (~ 4 GB). It's possible to have "unknown length" of unlimited size, which can be useful for streaming data.

### Definitions

The following sections contain the logic for encoding values into the binary format. Pick the first row in the table that is applicable. The syntax `a[i]` means the element `i` of array `a`, where `i` is the zero based array index. The syntax `length(a)` means the number of elements in array `a`. The syntax `B(x)` is the literal value of `x` as a byte in the resulting encoding. The syntax `U(x)` is the literal value of `x` as a 32 bit unsigned big-endian integer in the resulting encoding. The syntax `E(x)` is the encoding of `x` as specified by this section. The syntax `f(a[...])` means "apply `f` to all elements of `a` in order".

### Byte sequences

The rules in the following table is used to encode byte sequences. The byte sequence is an array of bytes called `bs`.

| Condition | Encoding |
|-----------|---------:|
| `length(bs) = 1` and `bs[0] < 128` | ```                                  B(bs[0])``` |
| `length(bs) < 120`                 | ```B(128 + length(bs))               B(bs[...])``` |
| otherwise                          | ```B(255)              U(length(bs)) B(bs[...])``` |

### Constructors

The rules in the following table is used to encode constructors. The constructor number is called `c` and the fields of the constructor is an array called `fs`. The first row in the table is applicable when the type has exactly one constructor.

| Condition | Encoding |
|-----------|---------:|
| exactly one constructor | ```            E(fs[...])``` |
| `c < 128`               | ```B(c)        E(fs[...])``` |
| otherwise               | ```B(254) U(c) E(fs[...])``` |

### Arrays

If a type is defined exactly like the `List<T>` type mentioned earlier, modulo renaming of identifiers and constructor order, the following table MAY be used instead of the table in the previous section. The elements of the list is called `es`.

| Condition | Encoding |
|-----------|---------:|
| `length(es) < 120` | ```B(128 + length(es))               E(es[...])``` |
| otherwise          | ```B(255)              U(length(es)) E(es[...])``` |
