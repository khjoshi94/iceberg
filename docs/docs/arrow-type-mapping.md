---
title: "Arrow Type Mapping"
---
<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may not use this file except in compliance with
 - the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing, software
 - distributed under the License is distributed on an "AS IS" BASIS,
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 - See the License for the specific language governing permissions and
 - limitations under the License.
 -->

# Arrow ↔ Iceberg Type Mapping (Java)

This page documents how Iceberg’s Java implementation maps Iceberg types to Apache Arrow types.  
It is intended as a reference for developers working with Arrow-based readers, writers, or integrations.

> **Note:**  
> This document describes the observable behavior of the Java implementation.  
> It does not duplicate or restate the logic in `ArrowSchemaUtil`.  
> For authoritative behavior, refer directly to the implementation.

---

## Overview

Iceberg’s Java Arrow integration is implemented primarily in:

- `org.apache.iceberg.arrow.ArrowSchemaUtil`
- `IcebergToArrowTypeConverter`
- Vectorized readers/writers under `org.apache.iceberg.arrow.vectorized`

The mapping is deterministic but may differ from other language bindings (e.g., PyIceberg).  
This page documents the canonical mapping used by the Java implementation.

---

# 1. Iceberg → Arrow Type Mapping

The following table summarizes how Iceberg types are converted to Arrow types when constructing Arrow schemas or vectors.

| Iceberg type | Arrow type | Notes |
|--------------|------------|-------|
| `boolean` | `ArrowType.Bool` | |
| `int` | `ArrowType.Int(32, signed=true)` | |
| `long` | `ArrowType.Int(64, signed=true)` | |
| `float` | `ArrowType.FloatingPoint(SINGLE)` | |
| `double` | `ArrowType.FloatingPoint(DOUBLE)` | |
| `decimal(p, s)` | `ArrowType.Decimal(precision=p, scale=s, bitWidth=128)` | [1] |
| `string` | `ArrowType.Utf8` | |
| `binary` | `ArrowType.Binary` | |
| `fixed[n]` | `ArrowType.FixedSizeBinary(n)` | |
| `uuid` | `ArrowType.FixedSizeBinary(16)` | [2] |
| `date` | `ArrowType.Date(DAY)` | |
| `time` | `ArrowType.Time(64, MICROSECOND)` | |
| `timestamp` | `ArrowType.Timestamp(MICROSECOND, tz?)` | [3] |
| `timestamp_ns` (v3) | `ArrowType.Timestamp(NANOSECOND, tz?)` | [3][4] |
| `struct<…>` | `ArrowType.Struct` | |
| `list<T>` | `ArrowType.List` | [5] |
| `map<K,V>` | `ArrowType.Map` | [6] |

### Notes

1. **Decimal precision/scale**  
   Iceberg Java always uses 128‑bit Arrow decimals. Arrow may downcast when writing to Parquet, but the in‑memory representation is fixed.

2. **UUID**  
   Represented as 16‑byte fixed binary. Consumers must interpret the bytes as UUIDs.

3. **Timezone handling**  
   Arrow timezone metadata is set only when `shouldAdjustToUTC()` is enabled.  
   Iceberg timestamps are always stored as UTC instants.

4. **`timestamp_ns` is a format v3+ type**  
   Only available for tables using format version 3 or higher.

5. **Lists**  
   Element nullability is preserved using Arrow’s child field metadata.

6. **Maps**  
   Represented as a list of key/value structs following Arrow’s MAP logical type convention.

---

# 2. Arrow → Iceberg Type Mapping

The Java implementation does not currently provide a full Arrow → Iceberg schema converter.

Where conversions do occur (e.g., vectorized reads), they are limited to the types supported by the reader and are not exposed as a general-purpose schema conversion API.

If you need a full Arrow → Iceberg mapping, see the PyIceberg implementation for reference.

---

# 3. Format Version Notes (v3+)

Some Iceberg types exist only in format version 3 and above:

- `timestamp_ns`
- `timestamptz` (if/when supported)
- `UnknownType`
- `geometry` / `geography` (implementation-dependent)

Java may not support all v3 types yet. Unsupported types will raise errors during schema conversion.

---

# 4. Implementation References

For authoritative behavior, consult:

- `org.apache.iceberg.arrow.ArrowSchemaUtil`
- `IcebergToArrowTypeConverter`
- Vectorized readers/writers under `org.apache.iceberg.arrow.vectorized`
- Iceberg specification: Type system & format versioning

---

# 5. Caveats & Limitations

- Arrow has multiple equivalent representations (e.g., `string` vs `large_string`), but Iceberg Java uses a single canonical mapping.
- Decimal256 is not supported.
- Arrow timezone metadata may not be preserved across all operations.
- Arrow → Iceberg schema conversion is intentionally not implemented in Java.
