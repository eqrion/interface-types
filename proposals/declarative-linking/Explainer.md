# Declarative Linking proposal

Declarative linking is an extension to WebAssembly in order to allow modules to express some forms of composition in environments that don't support garbage collection.

`NOTE: This explainer is in-progress and not completed yet`

## Example

```
(module $self
  (memory (export "memory") 1 1)

  (type $alloc-module (module
      (import "memory" (memory 1 1))
      (func (export "alloc") (param i32) (result i32))
      (func (export "dealloc") (param i32))
    )
  )

  (* definite import, host will provide it during instantation *)
  (import $nested-import "./modules/alloc.wasm" $alloc-module)

  (* direct nested module *)
  (module $nested-direct
    (import "alloc" (param i32) (result i32))
    (import "dealloc" (param i32))

    (func (export "execute"))
  )

  (*
   * this link function overrides the type of this module from
   *   `[] -> []` to
   *   `[] -> [(export "run" (func))]`
   *)
  (link (export "run" (func))
    (* link function controls when containing module is instantiated *)
    module.instantiate $self

    (* extract the memory export from the instance *)
    instance.export "memory"

    (* instantiate instruction contains order of imports to pop from stack *)
    module.instantiate (import "memory") $nested-import

    (* need two instances to get two exports *)
    dup
    instance.export "alloc"
    (* need to move dup'ed instance to top of stack *)
    pick 1
    instance.export "dealloc"

    module.instantiate (import "dealloc") (import "alloc") $nested-direct

    (* leave resulting exports of this instance on stack *)
    instance.export "execute"
  )

)
```
