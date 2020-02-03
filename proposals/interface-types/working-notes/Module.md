# Interface module

This proposal defines an `#interface-module` which is composed of `#interface-func`, `#interface-import`, and `#interface-export`.

An interface module is given by the following syntax:

```
#interface-module ::= {
  imports #interface-import*,
  funcs #interface-func*,
  exports #interface-export*,
}
```

## Interface func

An `interface-func` is set of `interface-instr*` that map a sequence of `interface-valtype` to another sequence of `interface-valtype`. This is expressed with the `interface-functype` `[#interface-valtype*] -> [#interface-valtype*]`.

An `interface-func` is executed with an evaluation stack and set of locals.

```
#interface-funcidx ::= #u32

#interace-functype ::= {
  params #interface-valtype*,
  results #interface-valtype*,
}

#interface-func ::= {
  type #interface-functype,
  locals #u32,
  body #interface-instr*,
}
```

## Interface import

An `interface import` is a description of an interface func that will be provided by instantiation of the interface module.

```
#interface-import ::= import #name? #name #interface-functype
```

## Interface export

An `interface export` is a description of an interface func that is exposed from this interface module.

```
#interface-export ::= export #name #interface-funcidx
```

## Nested interface modules

This proposal extends the declarative linking proposal to allow definition of an interface module as a nested module.

```
#nested-module ::=
  | module #module
  | interface #interface-module

#module ::= {
  ..
  nested-modules #nested-module*
}
```

The `moduleidx` space may now refer to a `#module` or an `#interface-module`. `#interface-module`'s are not allowed to be imported or exported, so they must always be defined within a `#module` as a leaf node.

## Linking interface modules

As a `#moduleidx` may now refer to a `#module` or `#interface-module` the `#link` function must be updated to support instantiating interface modules. This is done by replacing `#externval` and `#externtype` with a version that supports interface funcs

```
#linktype ::=
  | func #functype
  | global #globaltype
  | mem #memtype
  | table #tabletype
  | interface-func #interface-functype

#linkval ::=
  | func #funcaddr
  | global #globaladdr
  | mem #memaddr
  | table #tableaddr
  | interface-func #interface-funcaddr

#link-import ::= import #importname #linktype
#link-export ::= export #name #linktype

#link-val ::=
  | link #linkval
  | instance #moduleinst

#link-valtype ::=
  | link #linktype
  | instance #instancetype
```

A subtyping relationship is established between `#functype` and `#interface-functype` so that funcs and interface funcs may be used interchangeably for compatible imports and exports.

```
func x y <: interface-func z w
  iff x <: z && y <: w

interface-func x y <: func z w
  iff x <: z && y <: w

(param x) <: (param z)
  iff x <: z
(param x) y <: (param z) w
  iff x <: z && y <: w

(result x) <: (result z)
  iff x <: z
(result x) y <: (result z) w
  iff x <: z && y <: w
```
