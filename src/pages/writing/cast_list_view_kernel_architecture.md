---
layout: ../../layouts/Layout.astro
title: "High-Performance Zero-Copy Casting Kernels for Apache Arrow ListView Arrays"
author: "Jay Salvi"
date: "Sept 18, 2026"
---

# Deep Dive: High-Performance Zero-Copy Casting Kernels for Apache Arrow `ListView` Arrays

**Author:** Jay Salvi  
**Focus:** Columnar Memory Systems, C++ Kernel Optimization, Vectorized Data Processing  

---

## 1. Introduction & Motivation

In columnar data engines like **Apache Arrow**, **DuckDB**, **Polars**, and **Velox**, array representation directly dictates query execution performance. Nested data types—specifically arrays containing sub-lists—are widely used in vector databases, machine learning feature stores, and document analytics.

Historically, Apache Arrow supported the standard `List<T>` (and `LargeList<T>`) layout. In recent format specifications, Arrow introduced **`ListView<T>`**, a more flexible layout for nested data.

While `ListView` offers distinct advantages for zero-copy slicing and out-of-order data references, downstream analytical operations (such as arrow compute expressions, parquet writers, or IPC serialization protocols) frequently expect standard `List<T>` representations.

This technical article details the design and implementation of **`CastListViewToVarList`**, a specialized compute kernel engineered to convert `ListView` arrays to standard `List` arrays with **zero memory allocation overhead** whenever possible, falling back to a hardware-optimized bitmap traversal path when memory is non-contiguous.

---

## 2. Memory Layout Breakdown: `List<T>` vs. `ListView<T>`

To appreciate the design of the conversion kernel, we must examine the underlying memory buffers of both data structures.

### Standard `List<T>` Layout
A standard `List<T>` array is defined by two primary buffers:
1. **Validity Bitmap:** 1 bit per list entry (null vs. valid).
2. **Offsets Buffer:** $N + 1$ integers defining contiguous boundaries.

```
Logical Array:  [[10, 20], null, [30, 40, 50]]

Offsets Buffer: [  0,        2,     2,            5  ]
                ↓         ↓      ↓             ↓
Values Data:    [ 10,  20,        30,   40,   50 ]
```

*Key Property:* The memory layout is strictly sequential. List item $i$ occupies physical memory from `offsets[i]` to `offsets[i+1]`. The size of list $i$ is implicitly `offsets[i+1] - offsets[i]`.

---

### `ListView<T>` Layout
A `ListView<T>` array introduces a third buffer to decouple memory ordering from logical sequence:
1. **Validity Bitmap:** 1 bit per list entry.
2. **Offsets Buffer:** $N$ integers specifying the starting position of each list.
3. **Sizes Buffer:** $N$ integers specifying the length of each list.

```
Logical Array:  [[10, 20], null, [30, 40, 50]]

Offsets Buffer: [  5,        0,     0  ]
Sizes Buffer:   [  2,        0,     3  ]
                ↓         ↓      ↓
Values Data:    [ 30,  40,  50,  _,  _,  10,  20 ]  <-- Unsorted physical RAM
```

*Key Property:* List elements can be stored in arbitrary order, overlap with one another, or leave gaps in physical memory.

---

## 3. Kernel Architecture & Algorithm Design

Converting `ListView` to `List` presents a trade-off:
- If the `ListView` elements happen to be ordered sequentially in RAM (the common case in analytical queries), we can **avoid allocating a new values buffer entirely**.
- If the elements are scattered or out-of-order, we must re-order the data, creating a new contiguous values buffer while skipping null entries efficiently.

```
                    ┌──────────────────────────────┐
                    │ Input ListView ArraySpan     │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                   ┌────────────────────────────────┐
                   │ Single-Pass Contiguity &       │
                   │ Destination Offset Loop        │
                   └───────────────┬────────────────┘
                                   │
                 Is Contiguous? ───┼────────────────┐
                       │                            │
                      YES                           NO
                       │                            │
                       ▼                            ▼
        ┌────────────────────────────┐  ┌────────────────────────────┐
        │ Zero-Copy Fast Path        │  │ Non-Contiguous Fallback     │
        │ • Shift dest_offsets       │  │ • SetBitRunReader 64-bit   │
        │ • Slice values buffer      │  │   bitmap scanning          │
        │ • O(1) Memory Overhead     │  │ • Recompute offsets        │
        └──────────────┬─────────────┘  │ • Call Flatten()           │
                       │                └─────────────┬──────────────┘
                       │                              │
                       └──────────────┬───────────────┘
                                      │
                                      ▼
                        ┌───────────────────────────┐
                        │ Output List<T> ArrayData  │
                        └───────────────────────────┘
```

---

## 4. Implementation Analysis

### 4.1 Single-Pass Contiguity & Offset Calculation

In a single pass through the input array length $N$, the kernel populates the output `dest_offsets` array while validating whether the memory region is strictly contiguous:

```cpp
src_offset_type start_offset = offsets[0];
bool is_contiguous = true;

for (int64_t i = 0; i < in_array.length; ++i) {
  dest_offsets[i] = static_cast<dest_offset_type>(offsets[i] - start_offset);
  
  if (in_array.IsNull(i)) {
    if (sizes[i] != 0) {
      is_contiguous = false; // Null entry contains non-zero size gap
    }
  } else if (i < in_array.length - 1 && offsets[i] + sizes[i] != offsets[i + 1]) {
    is_contiguous = false; // Gap or overlap between consecutive elements
  }
}

src_offset_type abs_end_offset = offsets[in_array.length - 1] + sizes[in_array.length - 1];
dest_offsets[in_array.length] = static_cast<dest_offset_type>(abs_end_offset - start_offset);
```

---

### 4.2 Path A: The Zero-Copy Fast Path ($O(1)$ Memory Overhead)

If `is_contiguous == true`, the `dest_offsets` buffer populated in the single-pass loop is already fully valid! 

The kernel simply slices the existing values buffer header without copying a single byte of underlying data:

```cpp
if (is_contiguous) {
  src_offset_type range = abs_end_offset - start_offset;
  
  // Downcast safety validation (e.g. LargeListView -> List)
  if constexpr (is_downcast) {
    if (range > std::numeric_limits<dest_offset_type>::max()) {
      return Status::Invalid("ListView array length exceeds target List offset capacity");
    }
  }

  // Zero-copy pointer slice
  values = values->Slice(start_offset, range);
}
```

---

### 4.3 Path B: Word-at-a-Time Bitmap Traversal (`SetBitRunReader`)

When data is non-contiguous, null entries must be handled without per-element branching overhead. 

Rather than executing an `if (IsNull(i))` check on every iteration (which leads to CPU branch mispredictions on sparse nulls), the kernel leverages **`arrow::internal::SetBitRunReader`** to traverse valid bit-runs 64 bits at a time:

```cpp
else {
  src_offset_type current_offset = 0;
  dest_offsets[0] = 0;
  const uint8_t* validity = in_array.buffers[0].data;

  if (validity == nullptr) {
    // Fast-path for arrays without nulls
    for (int64_t i = 0; i < in_array.length; ++i) {
      current_offset += sizes[i];
      dest_offsets[i + 1] = static_cast<dest_offset_type>(current_offset);
    }
  } else {
    // Traverse set bits in 64-bit word runs using hardware bit-counting (popcount/ctz)
    arrow::internal::SetBitRunReader reader(validity, in_array.offset, in_array.length);
    int64_t last_idx = 0;
    while (true) {
      const auto run = reader.NextRun();
      if (run.length == 0) break;
      
      // Fill null runs with stationary offsets
      for (int64_t i = last_idx; i < run.position; ++i) {
        dest_offsets[i + 1] = static_cast<dest_offset_type>(current_offset);
      }
      
      // Process contiguous runs of valid bits
      for (int64_t i = run.position; i < run.position + run.length; ++i) {
        current_offset += sizes[i];
        dest_offsets[i + 1] = static_cast<dest_offset_type>(current_offset);
      }
      last_idx = run.position + run.length;
    }
    
    // Fill remaining trailing nulls
    for (int64_t i = last_idx; i < in_array.length; ++i) {
      dest_offsets[i + 1] = static_cast<dest_offset_type>(current_offset);
    }
  }

  // Flatten non-contiguous elements into a new contiguous buffer
  ARROW_ASSIGN_OR_RAISE(auto flattened, list_view_array.Flatten(ctx->memory_pool()));
  values = flattened->data();
}
```

---

## 5. Algorithmic Complexity & Benchmark Summary

| Metric | Zero-Copy Path (Contiguous) | Fallback Path (Non-Contiguous) |
|:---|:---|:---|
| **Time Complexity** | $O(N)$ single-pass offset scan | $O(N + M)$ bitmap scan + flatten |
| **Space Complexity** | $O(1)$ extra allocations (Buffer slice) | $O(M)$ new values buffer allocation |
| **Branch Predictability** | Deterministic loop, zero branching | Bit-run vectorization (Branch-free validity) |
| **CPU Cache Efficiency** | High (L1/L2 prefetching active) | Dependent on scatter-gather pattern |

---

## 6. Engineering Takeaways

1. **Memory Layout Awareness:** Understanding data layout mechanics enables $O(1)$ zero-copy optimizations that completely bypass RAM bandwidth limits.
2. **Microarchitectural Branch Elimination:** Replacing per-element null checks with 64-bit word bitmap readers (`SetBitRunReader`) prevents CPU pipeline stalls on large datasets.
3. **Robust Systems Engineering:** Building low-level compute kernels requires balancing theoretical algorithmic bounds with concrete hardware execution behavior.

---

*Article authored by Jay Salvi. All code verified against Apache Arrow C++ compute kernel standards.*
