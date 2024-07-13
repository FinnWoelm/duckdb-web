---
layout: community_extension
title: evalexpr_rhai
excerpt: |
  DuckDB Community Extensions
  Evaluate the Rhai scripting language in DuckDB

extension:
  name: evalexpr_rhai
  description: Evaluate the Rhai scripting language in DuckDB
  version: 1.0.0
  language: C++
  build: cmake
  requires_toolchains: "rust"
  license: Apache-2.0
  maintainers:
    - rustyconover
  excluded_platforms: "windows_amd64_rtools;windows_amd64"
  hello_world: |
    -- Calculate the value of an expression
    SELECT evalexpr_rhai('40+2');

    ┌───────────────────────────────┐
    │     evalexpr_rhai('40+2')     │
    │ union(ok json, error varchar) │
    ├───────────────────────────────┤
    │ 42                            │
    └───────────────────────────────┘

    -- Expression's return type is a union of
    --
    -- ok JSON - the result of the expression as a JSON value
    -- error VARCHAR - the error if any from evaluating the expression

    -- Demonstrate returning a JSON object from Rhai
    SELECT evalexpr_rhai('#{"apple": 5, "price": 3.52}').ok;

    ┌────────────────────────────────────────────────────┐
    │ (evalexpr_rhai('#{"apple": 5, "price": 3.52}')).ok │
    │                        json                        │
    ├────────────────────────────────────────────────────┤
    │ {"apple":5,"price":3.52}                           │
    └────────────────────────────────────────────────────┘

    -- Demonstrate what happens when the expression
    -- cannot be parsed, an error is returned.
    SELECT evalexpr_rhai('#{"apple: 5}').error;

    ┌────────────────────────────────────────────────────┐
    │       (evalexpr_rhai('#{"apple: 5}')).error        │
    │                      varchar                       │
    ├────────────────────────────────────────────────────┤
    │ Open string is not terminated (line 1, position 3) │
    └────────────────────────────────────────────────────┘

    -- Either .ok or .error will be populated but never both.

    -- When evaluating and expression you can also pass in
    -- variables via context.
    CREATE TABLE employees (name text, state text, zip integer);
    INSERT INTO employees values
      ('Jane', 'FL', 33139),
      ('John', 'NJ', 08520);

    -- Pass the row from the employees table in as "context.row"
    SELECT evalexpr_rhai(
      '
      context.row.name + " is in " + context.row.state
      ',
      {
        row: employees
      }) as result from employees;

    ┌───────────────────────────────┐
    │            result             │
    │ union(ok json, error varchar) │
    ├───────────────────────────────┤
    │ "Jane is in FL"               │
    │ "John is in NJ"               │
    └───────────────────────────────┘

    -- To demonstrate how Rhai can be used to implement
    -- a function in DuckDB, the next example creates
    -- a macro function that calls a Rhai function
    -- to calculate the Collatz sequence length.

    CREATE MACRO collatz_series_length(n) as
      evalexpr_rhai('
        fn collatz_series(n) {
            let count = 0;
            while n > 1 {
              count += 1;
              if n % 2 == 0 {
                  n /= 2;
              } else {
                  n = n * 3 + 1;
              }
            }
            return count
        }
        collatz_series(context.n)
      ', {'n': n});

    -- Use the previously defined macro.
    SELECT
      collatz_series_length(range).ok::bigint as sequence_length,
      range as starting_value
    FROM
      range(10000, 20000)
    ORDER BY 1 DESC limit 10;

    ┌─────────────────┬────────────────┐
    │ sequence_length │ starting_value │
    │      int64      │     int64      │
    ├─────────────────┼────────────────┤
    │             278 │          17647 │
    │             278 │          17673 │
    │             275 │          13255 │
    │             273 │          19593 │
    │             273 │          19883 │
    │             270 │          14695 │
    │             270 │          15039 │
    │             267 │          10971 │
    │             265 │          16457 │
    │             265 │          16777 │
    ├─────────────────┴────────────────┤
    │ 10 rows                2 columns │
    └──────────────────────────────────┘

  extended_description: |
    The `evalexpr_rhai` extension provides a single function:

    `evalexpr_rhai(VARCHAR, JSON) -> UNION['ok': JSON, 'error': VARCHAR]`

    The arguments in order are:

    1. The [Rhai](https://rhai.rs) expression to evaluate.
    2. Any context values that will be available to the Rhai expression by
    accessing a variable called context.

    The return value is a [union](https://duckdb.org/docs/sql/data_types/union.html) type. The union type is very similar to the
    [Result](https://doc.rust-lang.org/std/result/) type from Rust.

    If the Rhai expression was successfully evaluated the JSON result of the
    expression will be returned in the ok element of the union. If there was
    an error evaluating the expression it will be returned in the error
    element of the expression.

repo:
  github: rustyconover/duckdb-evalexpr-rhai-extension
  ref: 4acdf799b1b72d43af4c50659a2c859814140b33

extension_star_count: 1

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

| function_name | function_type | description | comment | example |
|---------------|---------------|-------------|---------|---------|
| evalexpr_rhai | scalar        |             |         |         |



---

