# typed-format
Experimental: A minimal and complete schema language and binary format for data interchange.

## Schema language

The schema language is encoded in ASCII and has the following grammar, where `UPPER` is an upper case letter (A-Z) followed by zero or more letters and digits (0-9), and where `LOWER` is as `UPPER`, except that `LOWER` starts with a lower case letter (a-z). Whitespace can be used to separate tokens, but is otherwise ignored.

    definition ::= "type" UPPER generics? body
    body ::= fields | "{" (UPPER fields?)* "}"
    fields ::= "(" (type LOWER ("," type LOWER)*)? ")"
    generics ::= "<" UPPER ("," UPPER)* ">"
    type ::= "bytes" | custom
    custom ::= UPPER ("<" custom ("," custom)* ">")?
