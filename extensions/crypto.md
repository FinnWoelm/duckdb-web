---
layout: community_extension
title: crypto
excerpt: |
  DuckDB Community Extensions
  Cryptographic hash functions and HMAC

extension:
  name: crypto
  description: Cryptographic hash functions and HMAC
  version: 1.0.0
  language: C++
  requires_toolchains: "rust"
  build: cmake
  license: MIT
  excluded_platforms: "windows_amd64_rtools;windows_amd64"
  maintainers:
    - rustyconover

repo:
  github: rustyconover/duckdb-crypto-extension
  ref: b6ccda3451d4fac8a2c0ae5ab2bca5216f22424c

docs:
  hello_world: |
    -- Calculate the MD5 hash value of 'abcdef'
    SELECT crypto_hash('md5', 'abcdef');
    ┌──────────────────────────────────┐
    │   crypto_hash('md5', 'abcdef')   │
    │             varchar              │
    ├──────────────────────────────────┤
    │ e80b5017098950fc58aad83c8c14978e │
    └──────────────────────────────────┘

    -- Calculate a HMAC
    SELECT crypto_hmac('sha2-256', 'secret key', 'secret message');
    ┌──────────────────────────────────────────────────────────────────┐
    │     crypto_hmac('sha2-256', 'secret key', 'secret message')      │
    │                             varchar                              │
    ├──────────────────────────────────────────────────────────────────┤
    │ 2df792e08cefdc0ea9900c67c93cbe66b98231b829a5dccd3857a03baac35963 │
    └──────────────────────────────────────────────────────────────────┘
  extended_description: |
    `crypto` provides two functions:

    - `crypto_hash` applies cryptographically secure hash functions
    and returns the result as a hex encoded value.

    - `crypto_hmac` calculates the HMAC using a secret key and a
    specific hash function.

    The supported hash functions are:
      - `blake2b-512`
      - `keccak224`
      - `keccak256`
      - `keccak384`
      - `keccak512`
      - `md4`
      - `md5`
      - `sha1`
      - `sha2-224`
      - `sha2-256`
      - `sha2-384`
      - `sha2-512`
      - `sha3-224`
      - `sha3-256`
      - `sha3-384`
      - `sha3-512`
extension_star_count: 4

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

| function_name | function_type |                                                   description                                                    | comment |                             example                             |
|---------------|---------------|------------------------------------------------------------------------------------------------------------------|---------|-----------------------------------------------------------------|
| crypto_hash   | scalar        | Apply a cryptographic hash function specified as the first argument to the data supplied as the second argument. |         | SELECT crypto_hash('md5', 'test');                              |
| crypto_hmac   | scalar        | Calculate a HMAC value                                                                                           |         | SELECT crypto_hmac('sha2-256', 'secret key', 'secret message'); |



---

