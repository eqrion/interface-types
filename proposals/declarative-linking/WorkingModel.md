# Working model

Declarative linking is an extension to WebAssembly in order to allow modules to express some forms of composition in environments that don't support garbage collection.

This document describes a working model of the semantics of declarative linking. This is meant to be useful for developing the specification, but is not intended to be a formal description or an explainer.

## Module type

Declarative linking depends on the [module types](https://github.com/WebAssembly/module-types/) proposal.

## Nested modules

This proposal introduces the concept of a `nested module`, allowing a module to contain modules inside of it. A nested module has no access to the containing module, and is not instantiated by default. Its only purpose is to be instantiated with the new `link` function.

A nested module can be imported or defined directly in the module. If the module is imported, then its module type must be specified as with other imports. If a module is defined directly in a module, then it must be a valid module.

Regardless of if a nested module is imported or defined, it can be exported from its containing module.

The core structure is extended with the following additions.

```
#moduleidx ::= #u32

#exportdesc ::=
  | ..
  | module $moduleidx

#importdesc ::=
  | ..
  | module #moduletype

#externtype ::=
  | ..
  | module #moduletype

#externval ::=
  | ..
  | module #module

#module ::= {
  ..
  nested-modules #module*
}
```

The module index space is formed from the containing module, imported modules, and nested modules.

So there is always a `moduleidx = 0` (refering to the containing module), followed by the imported modules, then by the directly defined modules.

The text format is extended with the following additions.

```
#moduleidx ::= #u32 | #id

#modulefield ::=
  | ..
  | #module

#importdesc ::=
  | ..
  | #moduletype

#exportdesc ::=
  | ..
  | #moduleidx
```

The binary format is extended with the following additions.

```
#moduleidx ::= #u32

#module ::=
  ..
  #datasec∗
  #customsec*
  #modulesec*
  #customsec*
#modulesec ::= section<0x12>(#module)

#importdesc ::=
  ..
  | 0x4 #moduletype

#exportdesc ::=
  ..
  | 0x4 #moduleidx
```

Note: the changes to the text and binary formats make parsing and decoding a module a recursive activity.

## Single level import

This proposal relaxes the structure of `import` so that only a single level of `name` is required.

```
#import ::= { module #name?, name #name, desc #importdesc }
```

An `import` is now referred to as single level if it does not contain a `module` name.

The js-api will support single level imports by performing only one property level lookup for `name` if an `import` is single level.

## Definite/indefinite imports

This proposal adds the concept of `definite` and `indefinite` imports. A `definite` import is any `import` that is single level and whose `name` is a valid [URL string](https://url.spec.whatwg.org/#valid-url-string). Every other `import` is `indefinite`.

As single level imports are added by this proposal, this is backwards compatible.

The [`module_instantiate`](https://webassembly.github.io/spec/core/appendix/embedding.html#embed-module-instantiate) function of the core spec is modified to have an optional modulemap argument:

```
module_instantiate(store, module, externval*, modulemap?) : (store, moduleinst | error)
```

Where `modulemap` is a map from URLs to modules. As a precondition, the host must ensure that:
 * there are no cycles in `modulemap` and
 * all determinate import strings (URLs) in module and the module values of `modulemap` are present as keys of `modulemap`.

A new "determinate module linking" phase is then added to the beginning of `module_instantiate` that transforms the incoming module as defined by a new `module_link` function with signature:

```
module_link(module, modulemap) : (module | error)
```

The `module_link` function recursively replaces determinate module imports in module with nested modules according to `modulemap` until there are no more determinate module imports. Before each substitution, the new nested module is checked to match the module import's declared type, ensuring that if `module_link` does not fail, the resulting module is valid.

Note: the construction of `modulemap` (and hence URL resolution) is host defined. The spec is only concerned with what happens after resolution.

Note: `definite` imports do not break the capability based security model of Wasm, as the imported modules have no capabilities of their own. They must be instantiated and provided with capabilities which go through the normal import process.

## Link function

Finally, this proposals adds the concept of a `link` function. A module may contain a single `link` function which transforms the type of the module by specifying how itself and nested modules are to be instantiated.

This is done with a small stack based language that supports single pass validation and has no control flow. The goal of this language is to compactly express any static [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) where nodes are `instances` and edges are the pairing of an `export` of an `instance` to the `import` of another `instance`.

The core structure is modified with the following definitions:

```
#importname ::= #name? #name

#link-import ::= import #importname #externtype
#link-export ::= export #name #externtype

#link-instr ::=
  | module.instantiate #importname* #moduleidx
  | instance.export #name
  | pick #u32
  | dup
  | drop

#link-val ::=
  | extern #externval
  | instance #moduleinst

#link-valtype ::=
  | extern #externtype
  | instance #instancetype

#link-func ::= link #link-import* #link-export* #link-instr*

#module ::= { .. link #link-func? .. }
```

The `link-func`, if present, overrides the type of the module and specifies how to instantiate the contained modules to satisfy this type.

The module type is created from the following steps:

```
1. Generate an import with #importname and externtype for each #link-import in #link-func
2. Generate an export with #name and externtype for each #link-export in #link-func
3. Generate a module type with the imports and exports from steps 1, 2
```

Instantiation is performed by executing the following steps:

```
1. Create an evaluation stack of #link-val that is initially empty
2. For each #link-import, push the 'extern #externval' provided from the host
3. Execute the body of #link-func
4. For each #link-export in reverse order, pop an 'extern #externval' and bind it with the #name provided from the #link-export
5. Return an instance with exports generated from step 4
```

The behavior and validation of link instructions is given below.

### `module.instantiate #importname* #moduleidx`

The `module.instantiate` instruction instantiates a module given its index and a list of `externval`'s to satisfy the `imports`s.

The type of this instruction is:
```
[X*] -> [(instance T)]

where:
  imports-satisfied(#importname*, modules[#moduleidx].imports)
  X* are the corresponding #externtype's given from matching #importname* against the #moduleidx' module
  T is the #instancetype given from instantiating the #moduleidx' module
```

### `instance.export #name`

The `instance.export` instruction gets the `externval` associated with an `export` from an `instance`.

The type of this instruction is:
```
[(instance T)] -> [X]

where:
  T is an #instancetype that contains #name as an export
  X is the corresponding #externtype given from matching #name against T's exports
```

### `pick #u32`

The `pick` instruction moves the `#u32`'th value on the stack to the top.

The type of this instruction is:
```
[X Y*] -> [Y* X]

where:
  X is the #u32'th value
  Y* is the values before X
```

### `dup`

The `dup` instruction duplicates the value on the top of the stack.

The type of this instruction is:
```
[X] -> [X X]
```

### `drop`

The `drop` instruction removes the value on the top of the stack.

The type of this instruction is:
```
[X] -> []
```
