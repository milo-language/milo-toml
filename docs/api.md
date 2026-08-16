# toml API

```milo
Toml.parse(text)          // Result<Toml>

// tables
doc.get("key")            // Option<Toml>    — any value
doc.str("key")            // Option<string>
doc.i64("key")            // Option<i64>     — None for a float
doc.f64("key")            // Option<f64>     — an integer widens
doc.bool("key")           // Option<bool>
doc.table("key")          // Option<Toml>    — sub-table only
doc.keys()                // Vec<string>     — in document order

// dotted paths
doc.path("a.b.c")         // Option<Toml>
doc.strPath("a.b")        doc.i64Path("a.b")
doc.f64Path("a.b")        doc.boolPath("a.b")

// arrays
doc.at(0)                 // Option<Toml>
doc.len()                 // i64             — array elements, or table entries

// the value at this handle
doc.asStr()               doc.asI64()        doc.asF64()      doc.asBool()

// what is here
doc.kind()                // TomlKind.Str | Int | Float | Bool | DateTime | Array | Table
doc.isStr()               doc.isInt()        doc.isFloat()    doc.isNum()
doc.isBool()              doc.isDateTime()   doc.isArray()    doc.isTable()

// out
doc.stringify()           // string
```

## Walking a table with unknown keys

`keys()` returns the entries in document order, so a table whose schema you do
not know ahead of time can be iterated directly:

```milo
let env = cfg.table("env")!
for k in env.keys() {
    print(k + " = " + (env.str(k) ?? ""))
}
```

## Handling parse errors

`Toml.parse` returns a `Result`. `!` unwraps it (panicking on a bad document),
`?` propagates it, and `match` handles it. Errors carry a message plus the line
and column:

```milo
match Toml.parse(text) {
    Result.Ok(doc) => {
        print(doc.str("name") ?? "unnamed")
    }
    Result.Err(e) => {
        eprint("bad config: " + e)   // toml: duplicate key 'a' (line 2, column 1)
        return 1
    }
}
```

Strict, not last-wins: duplicate keys, duplicate table headers, and redefining a
table as a non-table are all errors here, rather than being silently accepted.

## Datetimes

The four TOML datetime forms (offset, local datetime, local date, local time)
are recognised, range-checked, and tagged `TomlKind.DateTime`. Their original
text is what you read back, with `asStr`. This library deliberately builds no
temporal values: a lossy conversion into one would be worse than handing back
what the file said.

## Round-trip fidelity

`stringify` guarantees that the text re-parses to an equal document, and that
rendering the reparse is byte-identical. It does not preserve formatting:

- comments are dropped
- `30_000` comes back as `30000`
- `'literal'` strings come back double-quoted
- an inline table that owns only tables comes back as a `[section]`

Key order, values, types and structure are preserved.

## Conformance

Graded against Python's `tomllib` over a 78-file corpus (31 valid, 47 invalid),
comparing values *and* types, on both a direct parse and a
`parse → stringify → parse` round trip. Every TOML v1.0.0 construct is
implemented: dotted keys, arrays of tables, inline tables, multi-line basic and
literal strings, integer radixes, float specials, and all four datetime forms.

`tests/corpus/` holds those 78 files, if you want to see what a given piece of
syntax parses to.

## Tests

```bash
milo test tests
milo run examples/config.milo examples/app.toml
```
