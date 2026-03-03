# Plan: VM Owns Heap and Namespaces (Ambitious Restructure)

This plan eliminates the boundary tension entirely by making the VM **own** its heap
and namespaces instead of borrowing them. This means the VM is the single execution
context for ALL operations — there is no "post-VM" or "pre-VM" boundary where heap
and interns exist without a VM.

## Motivation

The current architecture has an inherent tension:

1. The VM **borrows** `heap` and `namespaces` mutably
2. After execution, the VM must be **dropped** to release borrows before
   `heap`/`namespaces` can be moved into snapshots or accessed independently
3. But result conversion (`MontyObject::new`) needs `py_repr` which needs `&VM`
4. So conversion must happen either *before* VM drops (awkward split) or *after*
   (no VM available)

With PLAN.md (the incremental fix), we work around this by pre-converting the Return
value while the VM is alive. This works but creates a split where the Return path is
handled differently from other paths, and every call site needs the pre-conversion
boilerplate.

The owning approach eliminates the tension: the VM is alive for the entire lifecycle,
from input conversion through execution to result conversion.

## Key Changes

### 1. VM Struct — Own Instead of Borrow

```rust
// Before (borrows)
pub struct VM<'a, 'p, T: ResourceTracker> {
    pub(crate) heap: &'a mut Heap<T>,
    namespaces: &'a mut Namespaces,
    pub(crate) interns: &'a Interns,
    pub(crate) print_writer: &'a mut PrintWriter<'p>,
    // ...
}

// After (owns heap + namespaces)
pub struct VM<'a, 'p, T: ResourceTracker> {
    pub(crate) heap: Heap<T>,           // owned
    namespaces: Namespaces,              // owned
    pub(crate) interns: &'a Interns,     // still borrowed (from Executor)
    pub(crate) print_writer: &'a mut PrintWriter<'p>,  // still borrowed
    // ...
}
```

`interns` and `print_writer` remain borrowed since they come from outside the VM's
lifecycle (Executor owns interns, caller owns print_writer).

### 2. VM Lifecycle Methods

```rust
impl<'a, 'p, T: ResourceTracker> VM<'a, 'p, T> {
    /// Creates a new VM, taking ownership of heap and namespaces.
    pub fn new(
        heap: Heap<T>,
        namespaces: Namespaces,
        interns: &'a Interns,
        print_writer: &'a mut PrintWriter<'p>,
    ) -> Self { ... }

    /// Restores a VM from a snapshot, taking ownership of heap and namespaces.
    pub fn restore(
        snapshot: VMSnapshot,
        module_code: &'a Code,
        heap: Heap<T>,
        namespaces: Namespaces,
        interns: &'a Interns,
        print_writer: &'a mut PrintWriter<'p>,
    ) -> Self { ... }

    /// Consumes the VM and returns its owned heap and namespaces.
    /// Used when creating snapshots or when the caller needs the data back.
    pub fn into_parts(mut self) -> (Heap<T>, Namespaces) {
        self.cleanup();
        (self.heap, self.namespaces)
    }

    /// Returns current heap size (for capacity tracking).
    pub fn heap_size(&self) -> usize {
        self.heap.size()
    }

    /// Converts a return Value to MontyObject while VM is alive.
    pub fn value_to_object(&self, value: &Value) -> MontyObject {
        MontyObject::from_value(value, &self)
    }
}
```

### 3. `Executor::run()` — Simplified Flow

```rust
fn run(&self, inputs: Vec<MontyObject>, resource_tracker: impl ResourceTracker,
       print: &mut PrintWriter<'_>) -> Result<MontyObject, MontyException> {
    let heap_capacity = self.heap_capacity.load(Ordering::Relaxed);
    let mut heap = Heap::new(heap_capacity, resource_tracker);
    let namespaces = self.prepare_namespaces(inputs, &mut heap)?;

    // VM takes ownership
    let mut vm = VM::new(heap, namespaces, &self.interns, print);
    let frame_exit_result = vm.run_module(&self.module_code);

    // Handle NameLookup/ExternalCall loop — VM is alive
    // ...

    // Convert result while VM is alive — no boundary problem!
    let result = vm.frame_exit_to_object(frame_exit_result);

    // Get heap size for capacity tracking, then drop VM
    let heap_size = vm.heap_size();
    let (mut heap, namespaces) = vm.into_parts();

    if heap_size > heap_capacity {
        self.heap_capacity.store(heap_size, Ordering::Relaxed);
    }

    #[cfg(feature = "ref-count-panic")]
    namespaces.drop_global_with_heap(&mut heap);

    result.map_err(|e| e.into_python_exception(&self.interns, &self.code))
}
```

### 4. `handle_vm_result()` — Takes VM Instead of Separate Parts

```rust
pub(crate) fn handle_vm_result<T: ResourceTracker>(
    result: RunResult<FrameExit>,
    vm: VM<'_, '_, T>,  // takes ownership of VM
    executor: Executor,
) -> Result<RunProgress<T>, MontyException> {
    match result {
        Ok(FrameExit::Return(value)) => {
            // Convert while VM alive
            let obj = vm.value_to_object(&value);
            let (mut heap, mut namespaces) = vm.into_parts();
            value.drop_with_heap(&mut heap);
            #[cfg(feature = "ref-count-panic")]
            namespaces.drop_global_with_heap(&mut heap);
            Ok(RunProgress::Complete(obj))
        }
        Ok(FrameExit::ExternalCall { function_name, args, call_id, .. }) => {
            let function_name = function_name.into_string(vm.interns);
            let (args_py, kwargs_py) = args.into_py_objects(&mut vm);
            let vm_state = vm.snapshot();
            let (heap, namespaces) = vm.into_parts();
            Ok(RunProgress::FunctionCall(FunctionCall::new(
                function_name, args_py, kwargs_py, call_id.raw(), false,
                Snapshot { executor, vm_state, heap, namespaces },
            )))
        }
        // ... similar for other variants
    }
}
```

### 5. `MontyRun::start()` — Simplified

```rust
pub fn start<T: ResourceTracker>(self, inputs: Vec<MontyObject>,
    resource_tracker: T, print: &mut PrintWriter<'_>,
) -> Result<RunProgress<T>, MontyException> {
    let executor = self.executor;
    let mut heap = Heap::new(executor.namespace_size, resource_tracker);
    let namespaces = executor.prepare_namespaces(inputs, &mut heap)?;

    let mut vm = VM::new(heap, namespaces, &executor.interns, print);
    let vm_result = vm.run_module(&executor.module_code);
    // VM is alive — pass it directly
    handle_vm_result(vm_result, vm, executor)
}
```

No need for `check_snapshot` before `handle_vm_result` — the snapshot can be taken
inside `handle_vm_result` since the VM is alive.

### 6. `Snapshot::run()` — VM Owns the Data

```rust
pub(crate) fn run(self, result: impl Into<ExtFunctionResult>,
    print: &mut PrintWriter<'_>,
) -> Result<RunProgress<T>, MontyException> {
    let ext_result = result.into();

    // VM takes ownership of heap and namespaces from the snapshot
    let mut vm = VM::restore(
        self.vm_state, &self.executor.module_code,
        self.heap, self.namespaces,
        &self.executor.interns, print,
    );

    let vm_result = match ext_result { ... };

    handle_vm_result(vm_result, vm, self.executor)
}
```

### 7. `prepare_namespaces` — Move Into VM

Currently `prepare_namespaces` runs before the VM exists, calling `to_value` which
calls `Dict::from_pairs` and `Set::add`.

**Option A:** Keep `prepare_namespaces` pre-VM with `(heap, interns)` signatures.
This is pragmatic — input conversion is simple data mapping, not execution.

**Option B:** Create VM first, then load inputs through it:
```rust
let heap = Heap::new(...);
let namespaces = Namespaces::new_empty(namespace_size);
let mut vm = VM::new(heap, namespaces, interns, print);
vm.load_inputs(inputs)?;  // calls to_value internally, has full VM access
```

Option B is cleaner long-term but requires `to_value` to work through `&mut VM`.

### 8. `NameLookup::resume()` — Resolve Through VM

Currently, name resolution happens before `VM::restore()` because the VM borrows
heap/namespaces. With VM owning, restore first, resolve through VM:

```rust
pub fn resume(self, result: impl Into<NameLookupResult>,
    print: &mut PrintWriter<'_>,
) -> Result<RunProgress<T>, MontyException> {
    // Restore VM first (takes ownership of heap + namespaces)
    let mut vm = VM::restore(
        self.snapshot.vm_state, &self.snapshot.executor.module_code,
        self.snapshot.heap, self.snapshot.namespaces,
        &self.snapshot.executor.interns, print,
    );

    // Resolve name through the live VM
    let vm_result = match result.into() {
        NameLookupResult::Value(obj) => {
            let value = obj.to_value(&mut vm)?;  // VM is alive!
            // Cache in namespace through VM
            vm.cache_name(self.namespace_slot, self.is_global, value);
            vm.run()
        }
        NameLookupResult::Undefined => {
            let err = ExcType::name_error(&self.name);
            vm.resume_with_exception(err.into())
        }
    };

    handle_vm_result(vm_result, vm, self.snapshot.executor)
}
```

### 9. REPL — Same Pattern

The REPL mirrors the main execution path. `MontyRepl` owns `heap` and `namespaces`.
When starting a snippet, these are moved into the VM. After execution, they're
moved back out.

```rust
pub fn start(self, code: &str, print: &mut PrintWriter<'_>)
    -> Result<ReplProgress<T>, Box<ReplStartError<T>>>
{
    // ...compile snippet...

    let mut vm = VM::new(self.heap, self.namespaces, &executor.interns, print);
    let vm_result = vm.run_module(&executor.module_code);
    let (heap, namespaces) = vm.into_parts();

    // Reconstruct self with returned heap/namespaces
    let mut this = self;
    this.heap = heap;
    this.namespaces = namespaces;
    handle_repl_vm_result(vm_result, executor, this)
}
```

Note: `MontyRepl` can't simply give away its heap permanently since it needs it
for subsequent `feed()` calls. The pattern is: lend to VM → get back after
execution.

### 10. Module Init — Trivially Fixed

With VM ownership, `BuiltinModule::create(&mut self)` works naturally since `self`
is the VM and it owns the heap:

```rust
// In vm/mod.rs:
let heap_id = module.create(self)?;  // self is &mut VM
```

### 11. `run_ref_counts()` — Extract Parts After Conversion

```rust
fn run_ref_counts(&self, inputs: Vec<MontyObject>) -> Result<RefCountOutput, MontyException> {
    let mut heap = Heap::new(self.namespace_size, NoLimitTracker);
    let namespaces = self.prepare_namespaces(inputs, &mut heap)?;
    let mut print = PrintWriter::Stdout;
    let mut vm = VM::new(heap, namespaces, &self.interns, &mut print);
    let frame_exit_result = vm.run_module(&self.module_code);

    // Convert return value while VM alive
    let return_obj = match &frame_exit_result {
        Ok(FrameExit::Return(value)) => Some(MontyObject::from_value(value, &vm)),
        _ => None,
    };

    // Extract parts for ref-count inspection
    let (mut heap, namespaces) = vm.into_parts();
    // ... inspect ref counts using heap and namespaces directly ...
}
```

---

## Migration Path

This is a large refactoring. Recommended order:

1. **Change VM struct** to own `Heap<T>` and `Namespaces`
2. **Update `VM::new` and `VM::restore`** to take ownership
3. **Add `VM::into_parts()`** to extract owned data
4. **Update `run.rs`** — `Executor::run()`, `MontyRun::start()`
5. **Update `run_progress.rs`** — `handle_vm_result`, `Snapshot::run`,
   `ResolveFutures::resume`, `NameLookup::resume`
6. **Update `repl.rs`** — all execution paths
7. **Update `MontyObject::new`/`from_value`** to take `&VM`
8. **Update module init** to take `&mut VM`
9. **Clean up**: remove now-unnecessary pre-conversion boilerplate from PLAN.md

## Tradeoffs

**Pros:**
- Eliminates the boundary tension entirely — no "VM doesn't exist" problem
- Simpler mental model: VM is THE execution context
- `handle_vm_result` takes a VM instead of 5+ separate arguments
- No pre-conversion boilerplate at every call site
- `check_snapshot` can be folded into `handle_vm_result` (VM is alive)
- `MontyObject::new`, `from_value`, `to_value` all go through `&VM`
- Module init naturally has VM access

**Cons:**
- Large refactoring touching `vm/mod.rs`, `run.rs`, `run_progress.rs`, `repl.rs`
- Heap/namespaces move in and out of VM at snapshot boundaries (ownership ping-pong
  between VM and Snapshot/MontyRepl). This is semantically correct but requires care
- `CallFrame<'a>` lifetime `'a` currently comes from the heap borrow — with owned
  heap, this needs to reference something else (the Code objects in interns)
- `run_ref_counts` needs extra extraction step for post-execution inspection
- REPL's "lend to VM, get back" pattern is slightly more ceremony than current
  direct mutable borrow

**Risk:** The `CallFrame<'a>` lifetime issue is the main technical risk. Currently
frames borrow `Code` from `interns` via the `'a` lifetime that also ties to
`heap`/`namespaces`. With owned heap, the `'a` lifetime would come only from
`interns` and `print_writer`, which is actually cleaner — it correctly represents
"code lifetime" rather than "everything lifetime".
