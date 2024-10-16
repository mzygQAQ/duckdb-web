---
layout: community_extension_doc
title: lindel
excerpt: |
  DuckDB Community Extensions
  Linearization/Delinearization, Z-Order, Hilbert and Morton Curves

docs:
  extended_description: "This `lindel` extension adds functions for the linearization\
    \ and\ndelinearization of numeric arrays in DuckDB. It allows you to order\nmulti-dimensional\
    \ data using space-filling curves.\n\n## What is linearization?\n\n[Linearization](https://en.wikipedia.org/wiki/Linearization)\
    \ maps multi-dimensional data into a one-dimensional sequence while [preserving\
    \ locality](https://en.wikipedia.org/wiki/Locality_of_reference), enhancing the\
    \ efficiency of data structures and algorithms for spatial data, such as in databases,\
    \ GIS, and memory caches.\n\n> \"The principle of locality states that programs\
    \ tend to reuse data and instructions they have used recently.\"\n\nIn SQL, sorting\
    \ by a single column (e.g., time or identifier) is often sufficient, but sometimes\
    \ queries involve multiple fields, such as:\n\n- Time and identifier (historical\
    \ trading data)\n- Latitude and Longitude (GIS applications)\n- Latitude, Longitude,\
    \ and Altitude (flight tracking)\n- Latitude, Longitude, Altitude, and Time (flight\
    \ history)\n\nSorting by a single field isn't optimal for multi-field queries.\
    \ Linearization maps multiple fields into a single value, while preserving locality—meaning\
    \ values close in the original representation remain close in the mapped representation.\n\
    \n#### Where has this been used before?\n\nDataBricks has long supported Z-Ordering\
    \ (they also now default to using the Hilbert curve for the ordering).  This [video\
    \ explains how Delta Lake queries are faster when the data is Z-Ordered.](https://www.youtube.com/watch?v=A1aR1A8OwOU)\
    \ This extension also allows DuckDB to write files with the same ordering optimization.\n\
    \nNumerous articles describe the benefits of applying a Z-Ordering/Hilbert ordering\
    \ to data for query performance.\n\n- [https://delta.io/blog/2023-06-03-delta-lake-z-order/](https://delta.io/blog/2023-06-03-delta-lake-z-order/)\n\
    - [https://blog.cloudera.com/speeding-up-queries-with-z-order/](https://blog.cloudera.com/speeding-up-queries-with-z-order/)\n\
    - [https://www.linkedin.com/pulse/z-order-visualization-implementation-nick-karpov/](https://www.linkedin.com/pulse/z-order-visualization-implementation-nick-karpov/)\n\
    \nFrom one of the articles:\n\n![Delta Lake Query Speed Improvement from using\
    \ Z-Ordering](https://delta.io/static/c1801cd120999d77de0ee51b227acccb/a13c9/image1.png)\n\
    \nYour particular performance improvements will vary, but for some query patterns\
    \ Z-Ordering and Hilbert ordering will make quite a big difference.\n\n## When\
    \ would I use this?\n\nFor query patterns across multiple numeric or short text\
    \ columns, consider sorting rows using Hilbert encoding when storing data in Parquet:\n\
    \n```sql\nCOPY (\n  SELECT * FROM 'source.csv'\n  order by\n  hilbert_encode([source_data.time,\
    \ source_data.symbol_id]::integer[2])\n)\nTO 'example.parquet' (FORMAT PARQUET)\n\
    \n-- or if dealing with latitude and longitude\n\nCOPY (\n  SELECT * FROM 'source.csv'\n\
    \  order by\n  hilbert_encode([source_data.lat, source_data.lon]::double[2])\n\
    ) TO 'example.parquet' (FORMAT PARQUET)\n```\n\nThe Parquet file format stores\
    \ statistics for each row group. Since rows are sorted with locality into these\
    \ row groups the query execution may be able to skip row groups that contain no\
    \ relevant rows, leading to faster query execution times.\n\n## Encoding Types\n\
    \nThis extension offers two different encoding types, [Hilbert](https://en.wikipedia.org/wiki/Hilbert_curve)\
    \ and [Morton](https://en.wikipedia.org/wiki/Z-order_curve) encoding.\n\n### Hilbert\
    \ Encoding\n\nHilbert encoding uses the Hilbert curve, a continuous fractal space-filling\
    \ curve named after [David Hilbert](https://en.wikipedia.org/wiki/David_Hilbert).\
    \ It rearranges coordinates based on the Hilbert curve's path, preserving spatial\
    \ locality better than Morton encoding.\n\nThis is a great explanation of the\
    \ [Hilbert curve](https://www.youtube.com/watch?v=3s7h2MHQtxc).\n\n### Morton\
    \ Encoding (Z-order Curve)\n\nMorton encoding, also known as the Z-order curve,\
    \ interleaves the binary representations of coordinates into a single integer.\
    \ It is named after Glenn K. Morton.\n\n**Locality:** Hilbert encoding generally\
    \ preserves locality better than Morton encoding, making it preferable for applications\
    \ where spatial proximity matters.\n\nEncoded Output is limited to a 128-bit `UHUGEINT`.\
    \ The input array size is validated to ensure it fits within this limit.\n\n|\
    \ Input Type | Maximum Number of Elements | Output Type (depends on number of\
    \ elements) |\n|---|--|-------------|\n| `UTINYINT`   | 16 | 1: `UTINYINT`<br/>2:\
    \ `USMALLINT`<br/>3-4: `UINTEGER`<br/> 4-8: `UBIGINT`<br/> 8-16: `UHUGEINT`|\n\
    | `USMALLINT`  | 8 | 1: `USMALLINT`<br/>2: `UINTEGER`<br/>3-4: `UBIGINT`<br/>4-8:\
    \ `UHUGEINT` |\n| `UINTEGER`   | 4 | 1: `UINTEGER`<br/>2: `UBIGINT`<br/>3-4: `UHUGEINT`\
    \ |\n| `UBIGINT`    | 2 | 1: `UBIGINT`<br/>2: `UHUGEINT` |\n| `FLOAT`      | 4\
    \ | 1: `UINTEGER`<br/>2: `UBIGINT`<br/>3-4: `UHUGEINT` |\n| `DOUBLE`     | 2 |\
    \ 1: `UBIGINT`<br/>2: `UHUGEINT` |\n"
  hello_world: "WITH elements as (\n  SELECT * as id FROM range(3)\n)\nSELECT\n  a.id\
    \ as a,\n  b.id as b,\n  hilbert_encode([a.id, b.id]::tinyint[2]) as hilbert,\n\
    \  morton_encode([a.id, b.id]::tinyint[2]) as morton\nFROM\n  elements as a cross\
    \ join elements as b;\n┌───────┬───────┬─────────┬────────┐\n│   a   │   b   │\
    \ hilbert │ morton │\n│ int64 │ int64 │ uint16  │ uint16 │\n├───────┼───────┼─────────┼────────┤\n\
    │     0 │     0 │       0 │      0 │\n│     0 │     1 │       3 │      1 │\n│\
    \     0 │     2 │       4 │      4 │\n│     1 │     0 │       1 │      2 │\n│\
    \     1 │     1 │       2 │      3 │\n│     1 │     2 │       7 │      6 │\n│\
    \     2 │     0 │      14 │      8 │\n│     2 │     1 │      13 │      9 │\n│\
    \     2 │     2 │       8 │     12 │\n└───────┴───────┴─────────┴────────┘\n\n\
    -- Encode two 32-bit floats into one uint64\nSELECT hilbert_encode([37.8, .2]::float[2])\
    \ as hilbert;\n┌─────────────────────┐\n│       hilbert       │\n│       uint64\
    \        │\n├─────────────────────┤\n│ 2303654869236839926 │\n└─────────────────────┘\n\
    \n-- Since doubles use 64 bits of precision the encoding\n-- must result in a\
    \ uint128\n\nSELECT hilbert_encode([37.8, .2]::double[2]) as hilbert;\n┌────────────────────────────────────────┐\n\
    │                hilbert                 │\n│                uint128         \
    \        │\n├────────────────────────────────────────┤\n│ 42534209309512799991913666633619307890\
    \ │\n└────────────────────────────────────────┘\n\n-- 3 dimensional encoding.\n\
    SELECT hilbert_encode([1.0, 5.0, 6.0]::float[3]) as hilbert;\n┌──────────────────────────────┐\n\
    │           hilbert            │\n│           uint128            │\n├──────────────────────────────┤\n\
    │ 8002395622101954260073409974 │\n└──────────────────────────────┘\n\n-- Demonstrate\
    \ string encoding\nSELECT hilbert_encode([ord(x) for x in split('abcd', '')]::tinyint[4])\
    \ as hilbert;\n┌───────────┐\n│  hilbert  │\n│  uint32   │\n├───────────┤\n│ 178258816\
    \ │\n└───────────┘\n\n-- Start out just by encoding two values.\nSELECT hilbert_encode([1,\
    \ 2]::tinyint[2]) as hilbert;\n┌─────────┐\n│ hilbert │\n│ uint16  │\n├─────────┤\n\
    │       7 │\n└─────────┘\n\n-- Decode an encoded value\nSELECT hilbert_decode(7::uint16,\
    \ 2, false, true) as values;\n┌─────────────┐\n│   values    │\n│ utinyint[2]\
    \ │\n├─────────────┤\n│ [1, 2]      │\n└─────────────┘\n\n-- The decoding functions\
    \ take four parameters:\n-- 1. **Value to be decoded:** This is always an unsigned\
    \ integer type.\n-- 2. **Number of elements to decode:** This is a `TINYINT` specifying\
    \ how many elements should be decoded.\n-- 3. **Float return type:** This `BOOLEAN`\
    \ indicates whether the values should be returned as floats (REAL or DOUBLE).\
    \ Set to true to enable this.\n-- 4. **Unsigned return type:** This `BOOLEAN`\
    \ indicates whether the values should be unsigned if not using floats.\n-- The\
    \ return type of these functions is always an array, with the element type determined\
    \ by the number of elements requested and whether \"float\" handling is enabled\
    \ by the third parameter.\n\nSELECT hilbert_decode(hilbert_encode([1, -2]::bigint[2]),\
    \ 2, false, false) as values;\n┌───────────┐\n│  values   │\n│ bigint[2] │\n├───────────┤\n\
    │ [1, -2]   │\n└───────────┘\n"
extension:
  build: cmake
  description: Linearization/Delinearization, Z-Order, Hilbert and Morton Curves
  language: C++
  license: Apache-2.0
  maintainers:
  - rustyconover
  name: lindel
  requires_toolchains: rust
  version: 1.0.0
repo:
  github: rustyconover/duckdb-lindel-extension
  ref: 87bb25c92bc2073e5bb085543a649477a964fff0

extension_star_count: 24

---

### Installing and Loading
```sql
INSTALL {{ page.extension.name }} FROM community;
LOAD {{ page.extension.name }};
```

{% if page.docs.hello_world %}
### Example
```sql
{{ page.docs.hello_world }}```
{% endif %}

{% if page.docs.extended_description %}
### About {{ page.extension.name }}
{{ page.docs.extended_description }}
{% endif %}

### Added Functions

<div class="extension_functions_table"></div>

| function_name  | function_type |                           description                           | comment |                           example                           |
|----------------|---------------|-----------------------------------------------------------------|---------|-------------------------------------------------------------|
| hilbert_encode | scalar        | Encode an array of values using the Hilbert space filling curve |         | select hilbert_encode([43, 3]::integer[2]);                 |
| hilbert_decode | scalar        | Decode a Hilbert encoded set of values                          |         | select hilbert_decode(7::uint16, 2, false, true) as values; |
| morton_encode  | scalar        | Encode an array of values using Morton encoding                 |         | select morton_encode([43, 3]::integer[2]);                  |
| morton_decode  | scalar        | Decode an array of values using Morton encoding                 |         | select morton_decode(7::uint16, 2, false, true) as values;  |



---
