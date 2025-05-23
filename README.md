ppx_bin_prot
============

Generation of binary serialization and deserialization functions from type definitions.
There's more information about:

- The [bin-prot format](https://github.com/janestreet/bin_prot/blob/master/README.md)
- [bin-prot-shape](https://github.com/janestreet/bin_prot/blob/master/shape/README.md), which is useful for checking
  compatibility of the `bin_prot` representations of different types.

## A note about signatures

In signatures, `ppx_bin_prot` tries to generate an include of a named
interface, instead of a list of value bindings.
That is:

```ocaml
type 'a t [@@deriving bin_io]
```

will generate:

```ocaml
include Binable.S1 with type 'a t := 'a t
```

instead of:

```ocaml
val bin_t : 'a Bin_prot.Type_class.t ‑> 'a t Bin_prot.Type_class.t
val bin_read_t : 'a Bin_prot.Read.reader ‑> 'a t Bin_prot.Read.reader
val __bin_read_t__ : 'a Bin_prot.Read.reader ‑> (int ‑> 'a t) Bin_prot.Read.reader
val bin_reader_t : 'a Bin_prot.Type_class.reader ‑> 'a t Bin_prot.Type_class.reader
val bin_size_t : 'a Bin_prot.Size.sizer ‑> 'a t Bin_prot.Size.sizer
val bin_write_t : 'a Bin_prot.Write.writer ‑> 'a t Bin_prot.Write.writer
val bin_writer_t : 'a Bin_prot.Type_class.writer ‑> 'a t Bin_prot.Type_class.writer
val bin_shape_t : Bin_prot.Shape.t ‑> Bin_prot.Shape.t
```

There are however a number of limitations:
- the type has to be named t
- the type can only have up to 3 parameters
- there shouldn't be any constraint on the type parameters

If these aren't met, then `ppx_bin_prot` will simply generate a list of value
bindings.

## Expression extensions

Several new expression forms are supported to derive the individual
values defined by `ppx_bin_prot`. Each of the following extensions can
be used, with arbitrary type expressions in place of `t`, to produce
values of the corresponding types on the right:

```ocaml
[%bin_shape: t]      : Bin_prot.Shape.t
[%bin_digest: t]     : string
[%bin_size: t]       : t Bin_prot.Size.sizer
[%bin_write: t]      : t Bin_prot.Write.writer
[%bin_read: t]       : t Bin_prot.Read.reader
[%bin_writer: t]     : t Bin_prot.Type_class.writer
[%bin_reader: t]     : t Bin_prot.Type_class.reader
[%bin_type_class: t] : t Bin_prot.Type_class.t
```

### Weird looking type errors

In some cases, a type can meet all the conditions listed above, in which case the
rewriting will apply, but lead to a type error. This happens when the type `t`
is an alias to a type which does have constraints on the parameters, for
instance:

```ocaml
type 'a s constraint 'a = [> `read ]
val bin_s : ([> `read ] as 'a) Type_class.t0 -> 'a s Type_class.t0
val bin_read_s : ([> `read ] as 'a) Read.reader -> 'a s Read.reader
val bin_reader_s : ([> `read ] as 'a) Type_class.reader0 -> 'a s Type_class.reader0
val bin_size_s : ([> `read ] as 'a) Size.sizer -> 'a s Size.sizer
val bin_write_s : ([> `read ] as 'a) Write.writer -> 'a s Write.writer
val bin_writer_s : ([> `read ] as 'a) Type_class.writer0 -> 'a s Type_class.writer0
val bin_shape_s : Shape.t -> Shape.t

type 'a t = 'a s [@@deriving_inline bin_io]
include Binable.S1 with type 'a t := 'a t
[@@@end]
```

will give an error looking like:

```
Error: In this `with' constraint, the new definition of t
       does not match its original definition in the constrained signature:
       Type declarations do not match:
         type 'a t = 'a t constraint 'a = [> `read ]
       is not included in
         type 'a t
       File "binable.ml", line 34, characters 2-11: Expected declaration
       Their constraints differ.
```

To workaround that error, simply copy the constraint on the type which has the
`[@@deriving]` annotation. This will force generating a list of value bindings.

## Local-accepting bin_io functions

This ppx includes the option to support local allocation, a nonstandard OCaml
extension available at: https://github.com/ocaml-flambda/ocaml-jst

In both structures and signatures, `[@@deriving bin_io ~localize]` (and similarly
for `bin_write` and `bin_size`) additionally generates definitions for
`bin_write_t__local` and `bin_size_t__local` which accept locally allocated `t`s.

As well, the following extension points are available:

```ocaml
[%bin_size_local: t]       : t Bin_prot.Size.sizer__local
[%bin_write_local: t]      : t Bin_prot.Write.writer__local
```

No other values from this ppx currently support local allocations.
