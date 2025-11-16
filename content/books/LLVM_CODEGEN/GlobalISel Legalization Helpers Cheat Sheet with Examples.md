### `legalForTypesWithMemDesc`

Marks (op, type) pairs as legal when they match a full memory description tuple:
`{result type, address type, stored type, alignment}
`

**Rule**

```cpp
// Legal: load/store s16 from addrspace0, 8-bit alignment
getActionDefinitionsBuilder(G_LOAD)
  .legalForTypesWithMemDesc({{LLT::scalar(16), LLT::pointer(0, PtrBits), LLT::scalar(16), 8}});
getActionDefinitionsBuilder(G_STORE)
  .legalForTypesWithMemDesc({{LLT::scalar(16), LLT::pointer(0, PtrBits), LLT::scalar(16), 8}});

// Legal: extending load / truncating store (s16 result, s8 memory)
getActionDefinitionsBuilder(G_LOAD)
  .legalForTypesWithMemDesc({{LLT::scalar(16), LLT::pointer(0, PtrBits), LLT::scalar(8), 8}});
getActionDefinitionsBuilder(G_STORE)
  .legalForTypesWithMemDesc({{LLT::scalar(16), LLT::pointer(0, PtrBits), LLT::scalar(8), 8}});
```

**Effect**

* `{s16, p0, s16, 8}` = load/store a 16-bit scalar in addrspace 0, byte-aligned.
* `{s16, p0, s8, 8}` = extending load or truncating store.

**Before**

```
%p = COPY %addr
%v16 = G_LOAD s16, %p :: (load 1 from %p, addrspace 0, stored s8, align 8)
```

**After**

* Unchanged, because it’s marked legal.

---

### `lowerIf`

Applies the **lowering action** if a predicate evaluates true. Replaces the op with a simpler sequence.

**Rule**

```cpp
auto IsIntGT32 = [](const LegalityQuery &Q){
  return Q.Types[0].isScalar() && Q.Types[0].getScalarSizeInBits() > 32;
};
getActionDefinitionsBuilder(G_UREM)
  .lowerIf(IsIntGT32);
```

**Before**

```
%r = G_UREM s64 %x, %y
```

**After**

```
%q = G_UDIV s64 %x, %y
%t = G_MUL  s64 %q, %y
%r = G_SUB  s64 %x, %t
```

---

### `legalIf`

Marks an operation as legal only under a condition (no rewrite when true).

**Rule**

```cpp
auto IsPtr32 = [](const LegalityQuery &Q){
  return Q.Types[1].isPointer() && Q.Types[1].getSizeInBits() == 32;
};
getActionDefinitionsBuilder(G_LOAD)
  .legalIf(IsPtr32)   // load is legal only with 32-bit pointers
  .lower();           // otherwise lower
```

**Before (legal)**

```
%r = G_LOAD s64, %p  // with %p: p0 (32-bit ptr)
```

**After (legal)**

```
%r = G_LOAD s64, %p  // unchanged
```

**Before (illegal)**

```
%r = G_LOAD s64, %p  // with %p: p1 (64-bit ptr)
```

**After (lowered)**

```
%r_lo = G_LOAD s32, %p
%r_hi = G_LOAD s32, %p+4
%r    = G_MERGE_VALUES %r_lo, %r_hi
```

---

### `scalarize`

Converts a vector op into multiple scalar ops on its elements.

**Rule**

```cpp
getActionDefinitionsBuilder(G_ADD)
  .legalFor({ LLT::fixed_vector(2, 32) })
  .scalarize(0);
```

**Before**

```
%v = G_ADD <4 x s32> %a, %b
```

**After**

```
%a0, %a1, %a2, %a3 = G_UNMERGE_VALUES %a
%b0, %b1, %b2, %b3 = G_UNMERGE_VALUES %b
%s0 = G_ADD s32 %a0, %b0
%s1 = G_ADD s32 %a1, %b1
%s2 = G_ADD s32 %a2, %b2
%s3 = G_ADD s32 %a3, %b3
%v  = G_BUILD_VECTOR %s0, %s1, %s2, %s3
```

---

### `lower`

Always lowers the instruction (no predicate). The replacement depends on the opcode.

**Rule**

```cpp
getActionDefinitionsBuilder(G_FNEG)
  .lower();
```

**Before**

```
%r = G_FNEG s32 %x
```

**After**

```
%negzero = G_FCONSTANT float -0.0
%r       = G_FSUB s32 %negzero, %x
```

---

### `clampScalar`

Constrains the legal scalar type at a given index to a range. Below min → widen. Above max → narrow.

**Rule**

```cpp
getActionDefinitionsBuilder(G_ADD)
  .clampScalar(0, LLT::scalar(16), LLT::scalar(32));
```

**Case A (widen)**

* **Before:** `%r = G_ADD s8 %x, %y`
* **After:**

```
%xw = G_ANYEXT s16 %x
%yw = G_ANYEXT s16 %y
%rw = G_ADD   s16 %xw, %yw
%r  = G_TRUNC s8  %rw
```

**Case B (narrow)**

* **Before:** `%r = G_ADD s64 %x, %y`
* **After:**

```
%x_lo, %x_hi = G_UNMERGE_VALUES %x
%y_lo, %y_hi = G_UNMERGE_VALUES %y
%lo   = G_ADD  s32 %x_lo, %y_lo
%hi   = G_ADDE s32 %x_hi, %y_hi, /*carry_in*/
%r    = G_MERGE_VALUES %lo, %hi
```
