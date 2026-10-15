# json

JSON ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)) for rok: a `Value` tree, a strict parser with `line:col` errors, a compact and an indented writer, a streaming tokenizer for large inputs, typed encode / decode of your own types with `#[derive(Json)]`, and JSON Pointer ([RFC 6901](https://www.rfc-editor.org/rfc/rfc6901)). The API follows std's `rokon` module: `parse` / `to_string` / `from_str` / `encode` / `decode`, and `ToJson` / `FromJson` traits in place of `ToRokon` / `FromRokon`.

## Usage

```rokon
Package {
    deps: { "json": "0.1" },
}
```

```rok
let v = try json::parse("{\"name\": \"geom\", \"sizes\": [1, 2.5, 12345678901234567890]}");
println(v.get("name")?.as_str() ?? "");                  // geom
let big = v.pointer("/sizes/2")?.as_int() ?? 0;          // 12345678901234567890 (exact)
println(v.to_pretty(2));
```

### Typed

```rok
#[derive(Json)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Json)]
enum Mode { Fast, Limit(u32), Custom { level: i32; } }

#[derive(Json)]
struct Config { name: str; servers: List<Server>; mode: Mode; }

fn load(text: str) throws (json::JsonError) -> Config {
    try json::from_str(text)   // errors are "line:col: msg"
}

let cfg = try load("{\"name\": \"x\", \"servers\": [{\"host\": \"a\"}], \"mode\": {\"Limit\": 3}}");
println(json::to_string(cfg));
// {"name":"x","servers":[{"host":"a","port":80,"tls":null}],"mode":{"Limit":3}}
```

| rok                    | JSON                                                          |
|------------------------|---------------------------------------------------------------|
| struct `S { a: .. }`   | object; a missing member takes `= default`, `none` for `T \| none`, else is an error; unknown members are an error |
| tuple struct `S(a, b)` | array (one field: the field itself)                           |
| unit struct `S`        | `null`                                                        |
| enum variant           | `"V"`, `{ "V": x }`, `{ "V": [a, b] }`, `{ "V": { .. } }`     |
| `T \| none`            | `null` or the value                                           |
| integer types, `int`   | integer (range-checked); `int` is exact                       |
| `f32`, `f64`, `float`  | number (an integer is taken too)                              |
| `bool`, `str`, `char`  | `true` / `false`, string, one-character string                |
| `List`, `Set`, `OrderedSet`, `[T; N]`, tuples (1-6) | array                            |
| `Dict<K, V>`, `OrderedDict<K, V>` | object; keys are strings, integers or bools written as strings |
| `Value`                | any value                                                     |

A decode error carries a JSON Pointer `path` (`/servers/0/port`); `from_str` turns it into the `line:col` of that value (or of the key, for an unknown member).

### Streaming

`Tokenizer` yields `Event`s (`BeginObject`, `Key`, `Int`, `EndArray`, ..., `End`) and checks the grammar as it goes. `Tokenizer::from_reader` reads any `io::Reader` in 64 KiB chunks, so memory stays flat however large the input is.

```rok
use rok::std::fs;

fn count_ids(path: str) throws -> usize {
    let t = json::Tokenizer::from_reader(try fs::File::open(path));
    let n = 0usize;
    loop {
        match try t.next() {
            json::Event::Key(k) => if k == "id" { n += 1; },
            json::Event::End => break,
            _ => {},
        }
    }
    n
}
```

`json::parse_tokens(t)` builds a `Value` from a tokenizer (for a stream, or to change `t.max_depth`).

## API

| Item | Description |
|------|-------------|
| `parse(s: str) throws (JsonError) -> Value` | Read one document (nesting up to `MAX_DEPTH` = 128) |
| `parse_tokens(t: Tokenizer) throws (JsonError) -> Value` | Read one document from a tokenizer |
| `to_string<T: ToJson>(x: T) -> str` | Compact JSON |
| `to_pretty<T: ToJson>(x: T) -> str` | JSON indented by 2 spaces |
| `from_str<T: FromJson>(s: str) throws (JsonError) -> T` | Read a document as a `T` |
| `encode<T: ToJson>(x: T) -> Value` / `decode<T: FromJson>(v: Value, seg: str) throws (JsonError) -> T` | Typed conversion of a `Value` |
| `Value` | `Null` `Bool(bool)` `Int(int)` `Float(f64)` `Str(str)` `Array(List<Value>)` `Object(OrderedDict<str, Value>)` |
| `Value::to_string()` / `to_pretty(indent)` | Write; also `Display` |
| `Value::get(key)` / `at(i)` / `pointer(p)` | Member, item, JSON Pointer lookup (`none` if absent or malformed) |
| `Value::kind()` / `is_null()` / `as_bool()` / `as_int()` / `as_f64()` / `as_str()` / `as_array()` / `as_object()` | Inspection |
| `JsonError { msg, line, col, path }` | `Display`: `line:col: msg`, or `at <path>: msg` without a position |
| `Tokenizer::new(s)` / `from_reader(r)`, `next()`, `position()`, `depth()`, `max_depth` | Pull tokenizer |
| `escape_token(k)` | `k` as a JSON Pointer reference token (`~0`, `~1`) |

### Details

- Numbers without a fraction or exponent are exact `int`s of any size; others are `f64`. `-0` reads as the float `-0.0`. A number beyond the `f64` range (`1e400`) is an error.
- Floats are written with the shortest digits that read back to the same value, always with a `.` or an exponent so they read back as floats (`1.0`, `1e300`). Writing NaN / infinity panics (JSON has no form for them).
- Objects keep insertion order. A repeated key keeps its first position and its last value.
- Strings are written with `"`, `\` and control characters escaped; everything else as UTF-8.
- Input is a `str`, so it is valid UTF-8. Lone surrogate escapes (`"\ud800"`) are an error, since a `str` cannot hold them. A byte order mark is an error.

## Conformance

[JSONTestSuite](https://github.com/nst/JSONTestSuite) (`tests/jsontestsuite.txt`, every case of `test_parsing`): all 95 `y_` cases are accepted and all 188 `n_` cases rejected, by both `parse` and the streaming tokenizer (fed 7 bytes at a time). Of the 35 implementation-defined `i_` cases, 5 are accepted (huge integers, underflowing floats) and 30 rejected: invalid UTF-8 and UTF-16 input, lone surrogates, a BOM, numbers beyond `f64`, and 500-deep nesting (over `MAX_DEPTH`). The `y_` cases also round-trip through both writers.

## Performance

`rok bench` (`-O2`, Apple M-series), on a 6.8 MB document of 25 000 records:

| | Time | Throughput |
|-|------|------------|
| Tokenizer events only | 0.28 s | 24 MB/s |
| `parse` to a `Value` | 2.1 s | 3.2 MB/s |
| `Value::to_string` | 0.45-1.2 s | 6-15 MB/s |

Building the tree is dominated by allocating many small `OrderedDict`s and `List`s; for large inputs that only need a few fields, read events from the `Tokenizer`.

## Development

```bash
rok test        # tests
rok bench       # benchmarks
rok fmt --check
```

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at your option. `tests/jsontestsuite.txt` is from JSONTestSuite, MIT License, Copyright (c) 2016 Nicolas Seriot.
