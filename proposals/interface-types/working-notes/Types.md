# Interface value types

This proposal defines new interface values, which have corresponding interface value types. Interface value types are a superset of the [core value types](https://webassembly.github.io/spec/core/syntax/types.html#value-types)

Some interface value types are compound by being constructed using one or more interface value types. The abstract syntax is designed to prevent recursive types by not allowing a compound type definition to refer to itself or other definitions. This is so that the set of control flow needed to generate all interface values is simple enough to allow for optimized lazy semantics.

The interface value type syntax is summarized below. Following after that is a section for each type that gives the full syntax, possible values, validation rules, and subtyping relationships.

```
#interface-valtype ::=
  | #valtype
  | bool
  | s8
  | s16
  | s32
  | s64
  | u8
  | u16
  | u32
  | u64
  | string
  | record ..
  | variant ..
  | array ..
  | callback ..
```

## `bool` 

A boolean value. The set of `{true, false}`.

## `s8`, `s16`, `s32`, `s64`, `u8`, `u16`, `u32`, `u64`

Integer values.

| Type | Values            |
|------|-------------------|
| s8   | [-(2^7), 2^7-1]   |
| s16  | [-(2^15), 2^15-1] |
| s32  | [-(2^31), 2^31-1] |
| s64  | [-(2^63), 2^63-1] |
| u8   | [0, 2^8-1]        |
| u16  | [0, 2^16-1]       |
| u32  | [0, 2^32-1]       |
| u64  | [0, 2^64-1]       |

These types differ from the core value types in that they represent a range of integers instead of a set of bits.

## `string`

A string value is a homogenous sequence of [unicode code points](https://www.unicode.org/glossary/#code_point).

## `record`

The `record` interface value type is a compound type defined by the following abstract syntax:

```
#record ::= record $fields: (field #interface-valtype)*
```

A `record` value is an ordered set of `|$fields|` interface values, where each interface value is of the corresponding type in `$fields`. This is sometimes known as a struct or product type.

A `record` type is valid iff:
 * `|$fields| > 0`
 * For every `field` in `$fields`
  * `#interface-valtype` is valid

A `record` type may be a subtype of another `record` by the following rules:

```
record (field x) <: record (field y)
  iff x <: y

record (field x) y <: record (field z)
  iff x <: z

record (field x) y <: record (field z) w
  iff x <: z && (record y <: record w)
```

Subtyping is performed in order of fields and allows a subtype to have more fields than its super types.

## `variant`

The `variant` interface value type is a compound type defined by the following abstract syntax:

```
#variant ::= variant $options: (option #interface-valtype?)*
```

A `variant` value is the pair of `(#u32, #interface-value?)`, with the `#u32` being the index of an `(option)` and the `#interface-value` having the corresponding type, if it exists, of the `(option)`. This is sometimes known as a tagged union or sum type.

A `variant` type is valid iff:
 * `|$options| > 0`
 * For every `option` in `$options`
  * `#interface-valtype` is valid

A `variant` type may be a subtype of another `variant` by the following rules:

```
variant (option x) <: variant (option y)
  iff x <: y

variant (option x) <: variant (option y) z
  iff x <: y

variant (option x) y <: variant (option z) w
  iff x <: z && (variant y <: variant w)
```

Subtyping is performed in order of options and allows a subtype to have less options than its super types.

## `array`

The `array` interface value type is a compound type defined by the following abstract syntax:

```
#array ::= array #interface-valtype
```

An `array` value is a homogenous sequence of interface values with type `$interface-value-type`.

An `array` type is valid iff:
 * `#interface-valtype` is valid

An `array` type may be a subtype of another `array` by the following:

```
array x <: array y
  iff x <: y
```

## `callback`

The callback interface value type is defined by the following abstract syntax:

```
#callback ::= callback
  $params: (param #interface-valtype)*
  $results: (result #interface-valtype)*
```

A callback value is a pair of `(exec, dtor)`
 * `exec` is a function of type `[$params] -> [$results]`
 * `dtor` is a function of type `[] -> []`

The `dtor` may only be called once to signal that resources may be freed. If `exec` or `dtor` is called after this, they will trap.

A `callback` type is valid iff:
 * For each `param` in `$params`
  * `#interface-valtype` is valid
 * For each `result` in `$results`
  * `#interface-valtype` is valid

A `callback` type may be a subtype of another `callback` by the following rules:

```
callback x y <: callback z w
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

Subtyping is performed independently for params and results. Params and results can be subtypes but must have the same amount of types.
