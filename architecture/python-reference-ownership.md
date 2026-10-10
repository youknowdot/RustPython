# Python reference ownership

`PyObjectRef` is an alias for `PyRef<PyObject>`. Typed payload references,
erased references, and typed tuple views share the same owning representation.

## Representation and invariants

- `PyRef<T>` stores `NonNull<PyObject>` and a zero-sized
  `PhantomData<NonNull<Py<T>>>`. The latter preserves the previous typed pointer's
  covariance and auto-trait/`Unpin` shape. The transparent owner and its `Option`
  remain one pointer wide with pointer alignment.
- The sealed `object::PyRefTarget` selects `Deref::Target`: payload `T` maps to
  `Py<T>`, `PyObject` maps to itself, and immutable typed tuples map recursively
  to their existing typed `Py` view. Downstream safe code cannot register an
  unrelated layout. `PyObject` does not implement `PyPayload`.
- The owner itself has no new generic bound. `Clone` and `Drop` share one
  implementation over the existing object header. Object allocation, vtable
  destruction, weakrefs, GC, and trashcan handling are unchanged.
- Typed erasure uses `T: PyRefTarget<Target = Py<T>>`. This excludes the erased
  type from the conversion blanket, avoiding overlap with Rust's identity
  `From<T> for T`, while retaining typed and nested tuple conversions.
- Ownership transfers use `ManuallyDrop` or `forget`, without an extra incref or
  decrement. The existing raw APIs retain their ownership-transfer contracts.
- C-API typed output uses the same target equality. An erased owner supports
  `*mut PyObject`, but cannot produce the fictitious `*mut Py<PyObject>` output.
  Duplicate erased owner/member-layout/FFI implementations are removed.
- Feature-dependent `Send`/`Sync` behavior is retained. The non-threading erased
  owner is neither; the threading owner implements both.

## API compatibility considerations

Existing payload `PyRef<T>` and `PyObjectRef` use remains supported, but aliasing
makes some inference less direct:

- Distinct raw constructor signatures remain: erased
  `PyObjectRef::from_raw(NonNull<PyObject>)` and typed
  `PyRef::<T>::from_raw(*const Py<T>)`. Unqualified `PyRef::from_raw` can now be
  ambiguous and require the explicit type parameter.
- Associated-target dereferencing can require an explicit downcast type or a
  `PyRefTarget` equality bound in generic typed-view code. Arbitrary non-payload
  views no longer automatically gain typed dereferencing; supported internal
  views must be registered in the sealed selector.
- Rust `Debug` formatting now consistently uses the allocation's vtable rather
  than the statically selected payload's formatter. This is a Rust diagnostic
  difference, not a Python semantic assertion.
