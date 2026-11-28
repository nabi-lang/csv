# Changelog

All notable changes to this package. Versions follow semver, independent of the Nabi toolchain.

## 0.1.1 (2026-11-29)

- A read error other than a wrong field count (a quote left open, text
  after a quote, bad UTF-8, a failed read) ends the stream: the next
  call gives the end instead of the same error again, so a loop that
  catches errors and reads on stops. A `FieldCount` error still skips
  its record.

## 0.1.0 (2026-11-22)

First release.

- Reader over a stream, a `str` or bytes (`reader`, `from_str`, `from_bytes`):
  RFC 4180 and common dialects, fields as views of the input.
- `Dialect`: delimiter, quote, escape, comments, trimming, flexible records,
  header, blank lines, CRLF, quote style, BOM and buffer size.
- `Record` and `Headers`: fields by position or by header name;
  `Reader::read_columns` for column stores.
- `#[derive(Csv)]` with `FromRecord` / `ToRecord`, and `FromField` /
  `ToField` for integers, floats, `bool`, `char`, `str` and `T | none`.
- `Writer` that quotes only what needs it (`QuoteStyle`), to a file or to
  memory.
- `CsvError` with kind, line, column and field.
- Pure Nabi, every target including wasm. Needs Nabi 0.1 or later.
