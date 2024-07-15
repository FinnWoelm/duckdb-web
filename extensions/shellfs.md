---
layout: community_extension
title: shellfs
excerpt: |
  DuckDB Community Extensions
  Allow shell commands to be used for input and output

extension:
  name: shellfs
  description: Allow shell commands to be used for input and output
  version: 1.0.0
  language: C++
  build: cmake
  excluded_platforms: "wasm_mvp;wasm_eh;wasm_threads"
  license: MIT
  maintainers:
    - rustyconover

repo:
  github: rustyconover/duckdb-shellfs-extension
  ref: d01c73d211544f5f0ff62acb8263a9874f973ddd

docs:
  hello_world: |
    -- Generate a sequence only return numbers that contain a 2
    SELECT * from read_csv('seq 1 100 | grep 2 |');
    ┌─────────┐
    │ column0 │
    │  int64  │
    ├─────────┤
    │       2 │
    │      12 │
    │      20 │
    │      21 │
    │      22 │
    └─────────┘

    -- Get the first multiples of 7 between 1 and 3 5
    -- demonstrate how commands can be chained together
    SELECT * from read_csv('seq 1 35 | awk "\$1 % 7 == 0" | head -n 2 |');
    ┌─────────┐
    │ column0 │
    │  int64  │
    ├─────────┤
    │       7 │
    │      14 │
    └─────────┘

    -- Do some arbitrary curl
    SELECT abbreviation, unixtime from
    read_json('curl -s http://worldtimeapi.org/api/timezone/Etc/UTC  |');
    ┌──────────────┬────────────┐
    │ abbreviation │  unixtime  │
    │   varchar    │   int64    │
    ├──────────────┼────────────┤
    │ UTC          │ 1715983565 │
    └──────────────┴────────────┘
    ```

  extended_description: |
    The `shellfs` extension for DuckDB enables the use of Unix pipes for input and output.

    By appending a pipe character `|` to a filename, DuckDB will treat it as a series of commands to execute and capture the output. Conversely, if you prefix a filename with `|`, DuckDB will treat it as an output pipe.

    While the examples provided are simple, in practical scenarios, you might use this feature to run another program that generates CSV, JSON, or other formats to manage complexity that DuckDB cannot handle directly.

    ### Reading input from a pipe

    Create a program to generate CSV in Python:

    ```python
    #!/usr/bin/env python3

    print("counter1,counter2")
    for i in range(10000000):
        print(f"{i},{i}")
    ```

    Run that program and determine the number of distinct values it produces:

    ```sql
    select count(distinct counter1)
    from read_csv('./test-csv.py |');
    ┌──────────────────────────┐
    │ count(DISTINCT counter1) │
    │          int64           │
    ├──────────────────────────┤
    │                 10000000 │
    └──────────────────────────┘
    ```

    When a command is not found or able to be executed, this is the result:

    ```sql
    SELECT count(distinct column0) from read_csv('foo |');
    sh: foo: command not found
    ┌─────────────────────────┐
    │ count(DISTINCT column0) │
    │          int64          │
    ├─────────────────────────┤
    │                       0 │
    └─────────────────────────┘
    ```

    The reason why there isn't an exception raised in this cause is because the `popen()` implementation starts a shell process, but that shell process


    ### Writing output to a pipe

    ```sql
    -- Write all numbers from 1 to 30 out, but then filter via grep
    -- for only lines that contain 6.
    COPY (select * from unnest(generate_series(1, 30)))
    TO '| grep 6 > numbers.csv' (FORMAT 'CSV');
    6
    16
    26

    -- Copy the result set to the clipboard on Mac OS X using pbcopy
    COPY (select 'hello' as type, from unnest(generate_series(1, 30)))
    TO '| grep 3 | pbcopy' (FORMAT 'CSV');
    type,"generate_series(1, 30)"
    hello,3
    hello,13
    hello,23
    hello,30

    -- Write an encrypted file out via openssl
    COPY (select 'hello' as type, * from unnest(generate_series(1, 30)))
    TO '| openssl enc -aes-256-cbc -salt -in - -out example.enc -pbkdf2 -iter 1000 -pass pass:testing12345' (FORMAT 'JSON');

    ```

    ## Configuration

    This extension introduces a new configuration option:

    `ignore_sigpipe` - a boolean option that, when set to true, ignores the SIGPIPE signal. This is useful when writing to a pipe that stops reading input. For example:

    ```sql
    COPY (select 'hello' as type, * from unnest(generate_series(1, 300))) TO '| head -n 100';
    ```

    In this scenario, DuckDB attempts to write 300 lines to the pipe, but the `head` command only reads the first 100 lines. After `head` reads the first 100 lines and exits, it closes the pipe. The next time DuckDB tries to write to the pipe, it receives a SIGPIPE signal. By default, this causes DuckDB to exit. However, if `ignore_sigpipe` is set to true, the SIGPIPE signal is ignored, allowing DuckDB to continue without error even if the pipe is closed.

    You can enable this option by setting it with the following command:

    ```sql
    set ignore_sigpipe = true;
    ```

    ## Caveats

    When using `read_text()` or `read_blob()` the contents of the data read from a pipe is limited to 2GB in size.  This is the maximum length of a single row's value.

    When using `read_csv()` or `read_json()` the contents of the pipe can be unlimited as it is processed in a streaming fashion.

    A demonstration of this would be:

    ```python
    #!/usr/bin/env python3

    print("counter1,counter2")
    for i in range(10000000):
        print(f"{i},{i}")
    ```

    ```sql
    select count(distinct counter1) from read_csv('./test-csv.py |');
    ┌──────────────────────────┐
    │ count(DISTINCT counter1) │
    │          int64           │
    ├──────────────────────────┤
    │                 10000000 │
    └──────────────────────────┘
    ```

    If a `limit` clause is used you may see an error like this:

    ```sql
    select * from read_csv('./test-csv.py |') limit 3;
    ┌──────────┬──────────┐
    │ counter1 │ counter2 │
    │  int64   │  int64   │
    ├──────────┼──────────┤
    │        0 │        0 │
    │        1 │        1 │
    │        2 │        2 │
    └──────────┴──────────┘
    Traceback (most recent call last):
      File "/Users/rusty/Development/duckdb-shell-extension/./test-csv.py", line 5, in <module>
        print(f"{i},{i}")
    BrokenPipeError: [Errno 32] Broken pipe
    Exception ignored in: <_io.TextIOWrapper name='<stdout>' mode='w' encoding='utf-8'>
    BrokenPipeError: [Errno 32] Broken pipe
    ```

    DuckDB continues to run, but the program that was producing output received a SIGPIPE signal because DuckDB closed the pipe after reading the necessary number of rows.  It is up to the user of DuckDB to decide whether to suppress this behavior by setting the `ignore_sigpipe` configuration parameter.


extension_star_count: 36

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

### Added Settings

<div class="extension_settings_table"></div>

|      name      |  description   | input_type | scope  |
|----------------|----------------|------------|--------|
| ignore_sigpipe | Ignore SIGPIPE | BOOLEAN    | GLOBAL |



---

