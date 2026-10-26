# json

JSON ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)) for Rok: a `Value`
tree, a strict parser with `line:col` errors, compact and indented writers, a
streaming tokenizer for large inputs, typed encode / decode with
`#[derive(Json)]`, and JSON Pointer. The API follows std's `rokon` module:
`parse` / `to_string` / `from_str` / `encode` / `decode`, with `ToJson` /
`FromJson` traits.

## Add it

```
rok add json
```

## Usage

```rok
let v = try json::parse("{\"name\": \"geom\", \"sizes\": [1, 2.5, 12345678901234567890]}");
println(v.get("name")?.as_str() ?? "");          // geom
let big = v.pointer("/sizes/2")?.as_int() ?? 0;  // 12345678901234567890 (exact)
println(v.to_pretty(2));
```

Derive `Json` on your own types for typed `from_str` / `to_string`, and read
large inputs event by event with `Tokenizer`.

## Docs

https://docs.rok-lang.org

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option. `tests/jsontestsuite.txt` is from JSONTestSuite, MIT License,
Copyright (c) 2016 Nicolas Seriot.
