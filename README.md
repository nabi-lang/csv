<img src="https://raw.githubusercontent.com/nabi-lang/nabi/main/assets/icons/icon-512-transparent.png" alt="Nabi" width="96">

# csv

[![CI](https://github.com/nabi-lang/csv/actions/workflows/ci.yml/badge.svg)](https://github.com/nabi-lang/csv/actions/workflows/ci.yml) [![packages.nabi-lang.org](https://img.shields.io/github/v/tag/nabi-lang/csv?label=packages.nabi-lang.org&sort=semver)](https://packages.nabi-lang.org/packages/csv) [![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](#license)

Fast, streaming CSV for Nabi: RFC 4180 and the dialects found in the
wild, read from any stream, a `str` or bytes. Fields are views of the
input, not copies; rows decode into your structs with `#[derive(Csv)]`,
matched to the header by name; the writer quotes only what needs it.
The reader finds delimiters, quotes and line ends 16 bytes at a time
with `core::simd`.

## Add it

```sh
nabi add csv
```

Needs Nabi 0.1 or later. Pure Nabi: every target works, wasm included.

## Usage

Records from a string, fields by position or by header name:

```nabi
let r = csv::from_str("city,country,pop\nOslo,NO,709000\n\"Washington, D.C.\",US,689000\n");
let h = try! r.headers();
let total = 0;
try! for rec in r {
    println(f"{rec[0]} ({rec.by_name(h, "country") ?? "?"})");
    total += try! rec[2].parse::<i64>();
}
assert(total == 1398000);
```

Typed rows. A field of type `T | none` takes an empty field or a missing
column as `none`; a field with a default takes the default there:

```nabi
#[derive(csv::Csv, Debug)]
struct Trip {
    id: u64;
    fare: f64;
    tip: f64 = 0.0;
    note: str | none;
}

let text = "id,fare,tip,note\n1,12.50,2,\n2,8.00,,late\n";
let trips = try! csv::from_str(text).decode::<Trip>().collect::<List<Trip>>();
assert(trips[1].tip == 0.0 && trips[1].note == "late");
println(try! csv::to_str(trips));
```

A large file, one record at a time, so memory stays flat:

```nabi norun
let r = csv::reader(try std::fs::open("trips.csv"));
let rec = csv::Record::new();
let sum = 0.0;
while try r.read(ref rec) {
    sum += try rec[3].parse::<f64>();
}
```

Writing, to a file or to memory:

```nabi
let w = csv::Dialect::new().to_buffer();
try! w.write_record(["name", "note"]);
try! w.write_record(["Ann", "says \"hi\", twice"]);
assert(w.as_str() == "name,note\nAnn,\"says \"\"hi\"\", twice\"\n");
```

### Dialects

`Dialect::new()` is RFC 4180 with a header; each method changes one
thing:

```nabi
let d = csv::Dialect::new()
    .delimiter(';')
    .comment('#')
    .trim(csv::Trim::All)
    .escape('\\');
let rec = try! d.from_str("# export\nname ; note\nAnn ; \"a \\\"b\\\"\"\n").next() ?? csv::Record::new();
assert(rec.to_list() == ["Ann", "a \"b\""]);
```

| Method | Default | Effect |
|---|---|---|
| `delimiter(c)` | `,` | the field separator; `Dialect::tsv()` is a tab |
| `quote(c)` / `no_quotes()` | `"` | the quote, or none at all |
| `escape(c)` | none | an escape inside quotes (`\"`) |
| `double_quotes(b)` | `true` | `""` inside quotes is one quote |
| `comment(c)` | none | lines starting with `c` are skipped |
| `trim(t)` | `Trim::None` | strip blanks outside quotes: `Headers`, `Fields`, `All` |
| `flexible(b)` | `false` | records may have different field counts |
| `has_header(b)` | `true` | the first record is the header |
| `skip_blank_lines(b)` | `true` | empty lines are skipped, else one empty field |
| `crlf(b)` | `false` | the writer ends records with `\r\n` |
| `quote_style(s)` | `Necessary` | when the writer quotes: `Always`, `NonNumeric`, `Never` |
| `write_bom(b)` | `false` | the writer starts with a UTF-8 byte order mark |
| `buffer_size(n)` | 256 KiB | bytes read from a stream at a time |

Readers take LF, CRLF and CR line ends and skip a UTF-8 byte order mark.

### Errors

Every error has a kind, a 1-based line and column, and says what to
write instead:

```text
line 2, column 3: a quoted field is never closed; end it with `"` (write a quote inside a field as `""`)
line 3, column 3, field `qty`: `x2` is not a whole number that fits in i64; write one such as `42`
line 5: this record has 4 fields, but the header has 3; give every record 3 fields, or read with `Dialect::flexible(true)`
```

## What is in it

| Item | Contents |
|---|---|
| `reader`, `from_str`, `from_bytes`, `writer`, `to_str` | start with `Dialect::new()` |
| `Dialect`, `Trim`, `QuoteStyle` | the format, as above |
| `Reader` | `next` (also `try for`), `read` into a kept `Record`, `read_columns` (straight into per-column lists, for column stores), `headers`, `decode::<T>()`, `count`, `read_all`, `line`, `position` |
| `Record` | `len`, `get`, `rec[i]`, `iter`, `to_list`, `by_name`, `is_quoted`, `line`, `position`, `from_list` |
| `Headers` | `names`, `index`, `get`, `len` |
| `Writer` | `write_record`, `write` (a `Record`), `serialize` (a `#[derive(Csv)]` row), `write_field`, `write_quoted`, `write_unquoted`, `write_raw`, `end_record`, `flush`, `as_str` |
| `#[derive(Csv)]`, `FromRecord`, `ToRecord` | rows as structs; `Columns` and `Cursor` are what the derive calls |
| `FromField`, `ToField` | one field: every integer type, `f32`, `f64`, `bool`, `char`, `str`, `T \| none` |
| `CsvError`, `ErrorKind` | `kind`, `line`, `column`, `field`, `msg` |

## Speed

Fields are views of the text: a field is copied only when it holds an
escaped quote and is asked for. A stream is read in chunks, and each
chunk's complete lines become one `str` in one copy; a `str` is read a
window at a time, its fields views of the `str` itself. The reader scans
up to 256 records per call into one index, so the work per record stays
small.

Reading (fields counted, one record reused) and writing three files,
MB/s, best of 5:

| | Apple M5: csv | Rust `csv` 1.3 | CPython 3.9 `csv` | x86_64: csv | Rust `csv` 1.3 | CPython 3.14 `csv` |
|---|---|---|---|---|---|---|
| read `numeric` | 473 | 1404 | 144 | 259 | 867 | 99 |
| read `quoted` | 691 | 1588 | 178 | 423 | 1097 | 126 |
| read `wide` | 639 | 1411 | 147 | 390 | 1137 | 107 |
| read `numeric`, every field as a `str` | 383 | | | 206 | | |
| read `numeric` from a file | 455 | | | 224 | | |
| write `numeric` | 333 | 1206 | 125 | 159 | 743 | 121 |
| write `quoted` | 225 | 1911 | 160 | 114 | 842 | 142 |

Rust reads into a reused `StringRecord` (UTF-8 checked, as here);
CPython iterates `csv.reader` over an in-memory string. `nabi bench`
at `-O2`.

`numeric` is four short fields per row (an id, a price, a tag, a date;
31 MB), `quoted` three fields with quoted commas and `""` (66 MB),
`wide` twenty numbers per row (69 MB).

## Docs

https://packages.nabi-lang.org/packages/csv/latest/docs

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option.
