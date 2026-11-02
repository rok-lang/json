# Changelog

All notable changes to this stone. Versions follow semver, independent of the Rok toolchain.

## 0.2.0 (unreleased)

Needs Rok 0.2.

### Changed

- `Value::Object`, `Value::as_object` and derived `ToJson` / `FromJson` use `Dict`, which keeps
  insertion order.
- `Value::to_string` and `json::to_string` are now `to_str`.
- `json::encode` is now `to_value`; `json::decode(v, seg)` is now `from_value(v)`.
- `Tokenizer::from_reader` takes an `io::Read`.
- `JsonError` fields are private: read them with `message()`, `line()`, `col()` and `path()`.

### Removed

- The `OrderedDict` and `OrderedSet` impls of `ToJson` / `FromJson`; the `Dict` and `Set` ones
  cover them.
- The derive helpers `Fields`, `pack`, `unpack`, `variant_of`, `payload`, `no_payload`,
  `unknown_variant` and `seg` are internal; derived code still reaches them.

## 0.1.0

First release.
