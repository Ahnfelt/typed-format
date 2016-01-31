# typed-format
Experimental: A minimal and complete schema language and a compact binary format for data interchange.

## Schema language

The schema language defines two kinds of types: The built-in type `bytes` (the only, built-in type) which represents a sequence of bytes; and user-defined types. The user defined types are algebraic datatypes, letting you express record types, eg. "this field OR this field OR this field..." as well as sum types, eg. "this sort of value OR this sort of value OR this sort of value...".

As an example, the following user-defined type `Person` has a text field `name` and an integer field `age`:

    type Person(String name, Int age)

Values of the following type `Bool` is either `False` or `True`:

    type Bool {
        False
        True
    }

The following type describes an optional value, the type safe alternative to `null`. For example, an optional integer could be `None`, indicating no value, or `Some(42)`, indicating a value of 42:

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

The schema language is encoded in ASCII and has the following grammar, where `UPPER` is an upper case letter (A-Z) followed by zero or more letters and digits (0-9), and where `LOWER` is as `UPPER`, except that `LOWER` starts with a lower case letter (a-z). Whitespace can be used to separate tokens, but is otherwise ignored.

    definition ::= "type" UPPER generics? body
    body       ::= fields | "{" (UPPER fields?)* "}"
    fields     ::= "(" (type LOWER ("," type LOWER)*)? ")"
    generics   ::= "<" UPPER ("," UPPER)* ">"
    type       ::= "bytes" | custom
    custom     ::= UPPER ("<" custom ("," custom)* ">")?

