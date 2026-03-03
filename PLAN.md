# Plan: Fix VM Boundary Compilation Errors (Incremental)

This plan fixes the 46 compilation errors on the `dh/pass-vm` branch with minimal
ownership changes. The VM continues to **borrow** heap/namespaces. We extend the VM's
lifetime at post-execution boundaries and keep `(heap, interns)` signatures at
pre-execution boundaries.

## Principles

1. **Post-execution conversion** — keep the VM alive through result conversion by
   passing `&mut VM` to conversion functions (or doing conversion before VM drops).
2. **Pre-execution input conversion** — keep `(heap, interns)` signatures for
   `to_value` and the Dict/Set operations it calls, since the VM genuinely doesn't
   exist yet.
3. **Module initialization** — pass `&mut VM` since the VM is alive at the call site.

---

## Step 1: Module init — pass `&mut VM` to `create_module`

The VM is alive when modules are created (`vm/mod.rs:1582`).

**Files:** `modules/mod.rs`, `modules/asyncio.rs`, `modules/os.rs`, `modules/sys.rs`,
`modules/typing.rs`, `modules/pathlib.rs`

- Change `BuiltinModule::create(self, heap, interns)` → `create(self, vm: &mut VM<...>)`
- Change all `create_module(heap, interns)` → `create_module(vm: &mut VM<...>)`
- Inside each, replace `heap`/`interns` with `vm.heap`/`vm.interns`
- Update the call site in `vm/mod.rs:1582`:
  `module.create(self.heap, self.interns)` → `module.create(self)` (requires extracting
  into a helper since `self` is the VM)

**Fixes:** ~22 of the "takes 3 arguments but 4 supplied" errors on `set_attr`.

## Step 2: Dict/Set — add `(heap, interns)` overloads for pre-VM callers

The Dict/Set methods were changed to take `&mut VM`, but `to_value` (called before
VM exists) still needs `(heap, interns)` versions.

**Files:** `types/dict.rs`, `types/set.rs`

For each method that now takes `vm` but is also called from pre-VM contexts:

- **`Dict::from_pairs`** — add `from_pairs_with(pairs, heap, interns)` that takes
  `(heap, interns)`, and have `from_pairs(pairs, vm)` delegate to it.
  Or: keep `from_pairs` taking `(heap, interns)` and add a `from_pairs_vm` convenience.
  Pick whichever keeps the most-used call site (VM) cleanest.
- **`Dict::set`** — same pattern. The pre-VM caller is `object.rs::to_value` via
  `from_pairs`.
- **`Set::add`** — same pattern. Pre-VM caller is `object.rs::to_value`.
- **`Dict::get`** / **`Dict::get_by_str`** — check if any pre-VM callers exist.
  If not, keep `vm`-only.

The underlying hashing (`Value::py_hash`) already takes `&mut VM`. For the pre-VM
path, we need a `py_hash` that takes `(heap, interns)` — or more precisely, the hash
computation only needs heap for `HeapData` lookups and interns for string content.
Add a `py_hash_with(heap, interns)` method on `Value` that the pre-VM Dict/Set
overloads call.

**Fixes:** errors in `object.rs` (`to_value` path), `signature.rs:402`.

## Step 3: `MontyObject::new` / `from_value` — take `&VM`

**Files:** `object.rs`

- Change `MontyObject::new(value, heap, interns)` →
  `MontyObject::new(value, vm: &VM<...>)` (or `&mut VM`)
- Change `from_value(value, heap, interns)` → `from_value(value, vm: &VM<...>)`
- Change `from_value_inner(...)` and `from_value_inner_impl(...)` to take `&VM`
- Inside `from_value_inner_impl`: `py_repr(heap, interns)` → `py_repr(vm)`,
  `py_repr_fmt(..., heap, visited, interns)` → `py_repr_fmt(..., vm, visited)`

This makes all post-execution conversion go through `&VM`.

**Fixes:** errors at `object.rs:437,442,482,511` (the `py_repr` calls).

## Step 4: `run.rs::Executor::run()` — convert result before dropping VM

**File:** `run.rs`

Current flow:
```
vm runs → vm.cleanup() → heap accessed directly → frame_exit_to_object(result, heap, interns)
```

New flow:
```
vm runs → convert Return value through VM → vm.cleanup() → access heap → done
```

Concretely:
```rust
// After the loop handling NameLookup/ExternalCall:
let return_obj = match &frame_exit_result {
    Ok(FrameExit::Return(value)) => {
        let obj = MontyObject::from_value(value, &vm);
        value.drop_with_heap(&mut *vm.heap);
        Some(obj)
    }
    _ => None,
};

vm.cleanup();
// vm drops (NLL), heap/namespaces accessible again

if heap.size() > heap_capacity { ... }
#[cfg(feature = "ref-count-panic")]
namespaces.drop_global_with_heap(&mut heap);

// For Return, use pre-converted object; for errors, convert without VM
match frame_exit_result {
    Ok(FrameExit::Return(_)) => Ok(return_obj.unwrap()),
    // Other arms don't need py_repr, just error conversion
    ...
}
```

Alternatively, add a helper method on VM: `vm.finish(frame_exit_result) -> (MontyObject, heap_size)`
that does cleanup + conversion in one shot before the VM drops.

**Fixes:** the remaining `frame_exit_to_object` errors in `run.rs`.

## Step 5: `run.rs::Executor::run_ref_counts()` — same pattern

**File:** `run.rs`

Convert the return value while VM is alive (before `namespaces.into_global()`).
The ref-count inspection still happens after VM drops since it needs direct
namespace access.

## Step 6: `run_progress.rs::handle_vm_result()` — pre-convert Return value

**File:** `run_progress.rs`

`handle_vm_result` is called from `MontyRun::start()` and `Snapshot::run()` after
the VM's borrows end (NLL). Change it to receive an optional pre-converted
`MontyObject` for the Return case:

```rust
pub(crate) fn handle_vm_result<T: ResourceTracker>(
    result: RunResult<FrameExit>,
    vm_state: Option<VMSnapshot>,
    executor: Executor,
    mut heap: Heap<T>,
    mut namespaces: Namespaces,
    return_obj: Option<MontyObject>,  // pre-converted from caller
) -> Result<RunProgress<T>, MontyException>
```

At each call site, convert the Return value while VM is alive:
```rust
let vm_result = vm.run_module(...);
let vm_state = vm.check_snapshot(&vm_result);
let return_obj = if let Ok(FrameExit::Return(ref value)) = vm_result {
    Some(MontyObject::from_value(value, &vm))
} else { None };
// VM drops here
handle_vm_result(vm_result, vm_state, executor, heap, namespaces, return_obj)
```

Inside `handle_vm_result`, the Return arm uses `return_obj.unwrap()` and just
drops the original value:
```rust
Ok(FrameExit::Return(value)) => {
    value.drop_with_heap(&mut heap);
    #[cfg(feature = "ref-count-panic")]
    namespaces.drop_global_with_heap(&mut heap);
    Ok(RunProgress::Complete(return_obj.unwrap()))
}
```

The non-Return arms (`ExternalCall`, `OsCall`, `MethodCall`, `NameLookup`,
`ResolveFutures`) don't need `py_repr` — they just extract function names via
`into_string(interns)` and convert args via `into_py_objects(heap, interns)`,
which don't need `&VM`.

**Fixes:** error at `run_progress.rs:683`.

## Step 7: REPL `frame_exit_to_object` — take `&mut VM`

**File:** `repl.rs`

The REPL's local `frame_exit_to_object` (line 145) currently takes `(heap, interns)`
but its body already references `vm`. Change the signature to take `&mut VM`:

```rust
fn frame_exit_to_object(
    frame_exit_result: RunResult<FrameExit>,
    vm: &mut VM<'_, '_, impl ResourceTracker>,
) -> RunResult<MontyObject>
```

Update body: `args.drop_with_heap(vm)`, `method_name.as_str(vm.interns)`, and
`MontyObject::new(return_value, vm)`.

Update call sites:
- `MontyRepl::new()` line 301: VM is alive, pass `&mut vm`
- `MontyRepl::feed()` line 426: Move call before `vm.cleanup()`, pass `&mut vm`

**Fixes:** 4 "cannot find value `vm`" errors in `repl.rs`.

## Step 8: `handle_repl_vm_result` — same pre-conversion pattern

**File:** `repl.rs`

Same approach as Step 6: at each call site, pre-convert the Return value while VM
is alive, pass it to `handle_repl_vm_result`.

The call sites are:
- `MontyRepl::start()` line 364 — VM is alive in the block above
- `ReplSnapshot::run()` line 947 — VM is alive
- `ReplResolveFutures::resume()` line 884 — VM is alive
- `ReplNameLookup::resume()` line 759 — VM is alive

## Step 9: Remaining errors — `value.rs`, `heap.rs`, `exception_private.rs`, `signature.rs`

**`value.rs` (lines ~640-730):** The `py_repr` implementation on `Value` (the
`match self` that dispatches to each variant) was updated to take `&VM` but some
inner calls still pass `(heap, interns)`. Update the inner dispatch to use `vm`.

**`value.rs` (lines ~1639, 1685):** Uses of `heap.with_entry_mut(...)` that now
need `Heap::with_entry_mut(vm, ...)`. Update these call sites.

**`heap.rs:1216`:** `with_entry_mut` already takes `&mut VM` — the errors are at
call sites, not the definition.

**`exception_private.rs:312`:** `key_error(key, heap, interns)` → `key_error(key, vm)`.
Update call sites.

**`signature.rs:402`:** `excess_kwargs.set(key, value, heap, interns)` → use the
`(heap, interns)` overload from Step 2, since this may be in a non-VM context.
Check the calling context — if VM is available, use `set(key, value, vm)`.

**`types/tuple.rs:259`, `types/dataclass.rs:127`, `types/list.rs:443`:** Various
methods with signature mismatches. Update to pass `vm` where available.

## Step 10: Format and lint

```bash
make format-rs
make lint-rs
```

Fix any remaining warnings (e.g., unused imports of `Heap`/`Interns` in
`types/module.rs`).

## Step 11: Run tests

```bash
make test-ref-count-panic
```

---

## Error Count Estimate by Step

| Step | Errors fixed (approx) |
|------|-----------------------|
| 1. Module init | ~12 (set_attr calls) |
| 2. Dict/Set overloads | ~8 |
| 3. MontyObject takes VM | ~4 |
| 4-6. run.rs / run_progress.rs | ~4 |
| 7-8. repl.rs | ~6 |
| 9. Remaining scattered | ~12 |
| **Total** | **~46** |
