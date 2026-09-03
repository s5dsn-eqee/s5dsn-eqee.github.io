---
title: "Exposing Rust DataFusion UDFs to Python via the PyCapsule protocol"
date: 2026-09-02T10:45:00+02:00
tags: ["datafusion", "python", "rust", "ffi", "pyo3"]
summary: "How to bridge a Rust-only DataFusion crate into datafusion-python with datafusion-ffi, PyO3 and a single PyCapsule dunder, no fork of datafusion-python needed."
---

DataFusion keeps its core function library small. Aggregates like `mode`,
`skewness` and `kurtosis` live in the Rust-only
[datafusion-extra-functions](https://github.com/datafusion-contrib/datafusion-extra-functions)
crate. I wanted them in Python without forking datafusion-python, and it turns
out the whole bridge fits in one small `lib.rs`:
[datafusion-extra-functions-ffi](https://github.com/s5dsn-eqee/datafusion-extra-functions-ffi).

### The protocol

datafusion-python accepts foreign UDFs through PyCapsule protocols, the same
pattern Arrow uses for `__arrow_c_stream__`. When you pass an object to
`udaf()`, it looks for a `__datafusion_aggregate_udf__` method. That method
must return a `PyCapsule` named `datafusion_aggregate_udf` wrapping an
`FFI_AggregateUDF`, a stable-ABI struct from the
[datafusion-ffi](https://docs.rs/datafusion-ffi) crate. Your extension module
and datafusion-python stay separate compiled artifacts, the capsule is the
only contract between them.

### The implementation

Wrap the native `AggregateUDF` in a PyO3 class and implement the dunder:

```rust
use datafusion_ffi::udaf::FFI_AggregateUDF;
use pyo3::types::PyCapsule;

#[pyclass]
pub struct ExtraAggregateUDF {
    inner: Arc<AggregateUDF>,
}

#[pymethods]
impl ExtraAggregateUDF {
    fn __datafusion_aggregate_udf__<'py>(
        &self,
        py: Python<'py>,
    ) -> PyResult<Bound<'py, PyCapsule>> {
        let name = CString::new("datafusion_aggregate_udf").unwrap();
        let provider = FFI_AggregateUDF::from(Arc::clone(&self.inner));
        PyCapsule::new(py, provider, Some(name))
    }
}
```

`FFI_AggregateUDF::from` does the heavy lifting: it turns the trait object
into a `#[repr(C)]` vtable-style struct that survives crossing a shared
library boundary. The rest is a name lookup over
`all_extra_aggregate_functions()` and a `#[pymodule]` exporting it. There are
sibling structs for scalar and window UDFs, table providers and catalogs, so
the same recipe covers most extension points.

On the Python side:

```python
from datafusion import udaf
import datafusion_extra_functions_ffi as ffi

skew = udaf(ffi.udaf_by_name("skewness"))
```

### Packaging and the ABI trap

maturin builds the `cdylib` into a wheel. With PyO3's `abi3-py310` feature one
wheel per platform covers every Python from 3.10 up, no per-version builds.

The trap: `FFI_AggregateUDF` is stable across Python versions, but not across
DataFusion versions. Its layout changes with the DataFusion major, and a
mismatch between your crate and the one inside datafusion-python fails at
runtime with no clear error. Compile against the same major that
datafusion-python bundles and pin the Python dependency to it, `>=54,<55` in
my case. Bumping DataFusion means rebuilding and re-pinning, there is no way
around it.
