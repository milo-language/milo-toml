# toml

A [TOML v1.0.0](https://toml.io/en/v1.0.0) parser and serializer for
[Milo](https://github.com/milo-language/milo). No dependencies beyond the
standard library.

## Install

```bash
milo add github.com/milo-language/milo-toml
```

Then import it:

```milo
from "toml" import { Toml }
```

## Quick start

Copy this into `main.milo` and run `milo run main.milo`:

```milo
from "toml" import { Toml }

pub fn main(): i32 {
    let doc = Toml.parse("
name = \"checkout-api\"
debug = false

[server]
host = \"0.0.0.0\"
port = 8080
")!

    print(doc.str("name") ?? "unnamed")
    print(doc.strPath("server.host") ?? "127.0.0.1")
    print((doc.i64Path("server.port") ?? 80).toString())
    return 0
}
```

```
checkout-api
0.0.0.0
8080
```

## Reading a config file

Every accessor returns an `Option`, so `??` gives you a default in one line —
no missing-key branch to write.

```milo
from "toml" import { Toml }
from "std/fs" import { readFile }

pub fn main(): i32 {
    let cfg = Toml.parse(readFile("app.toml")!)!

    let name = cfg.str("name") ?? "unnamed"          // top-level string
    let debug = cfg.bool("debug") ?? false           // top-level bool
    let port = cfg.i64Path("server.port") ?? 8080    // dotted path into [server]
    let ratio = cfg.f64Path("limits.ratio") ?? 1.0   // dotted path, float

    print(name + " debug=" + debug.toString() + " port=" + port.toString())
    return 0
}
```

Sub-tables can also be pulled out and reused as their own handle:

```milo
let server = cfg.table("server")!
print((server.str("host") ?? "127.0.0.1") + ":" + (server.i64("port") ?? 80).toString())
```

## Arrays

```milo
// tags = ["web", "api", "beta"]
let tags = cfg.get("tags")!
for i in 0..tags.len() {
    print(tags.at(i)!.asStr() ?? "")
}
```

## Arrays of tables

`[[route]]` blocks come back as an array, in document order:

```milo
// [[route]]
// path = "/health"
// methods = ["GET"]
//
// [[route]]
// path = "/orders"
// methods = ["GET", "POST"]

let routes = cfg.get("route")!
for i in 0..routes.len() {
    let r = routes.at(i)!
    print(r.str("path") ?? "?")

    let methods = r.get("methods")!
    for m in 0..methods.len() {
        print("  " + (methods.at(m)!.asStr() ?? ""))
    }
}
```

## Walking unknown keys

```milo
let env = cfg.table("env")!
for k in env.keys() {
    print(k + " = " + (env.str(k) ?? ""))
}
```

## Handling parse errors

`Toml.parse` returns a `Result`. `!` unwraps it (panicking on a bad document),
`?` propagates it, and `match` lets you handle it. Errors carry a message plus
the line and column:

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

## Writing TOML

`stringify` renders a document back to text that re-parses to an equal document:

```milo
let doc = Toml.parse(readFile("app.toml")!)!
print(doc.stringify())
```

## Runnable examples

`examples/` holds a complete program you can run against a real config:

```bash
milo run examples/config.milo examples/app.toml
```

- `examples/config.milo` — typed lookups with defaults, a nested table, and an
  array of tables walked in order
- `examples/app.toml` — the service config it reads

`tests/corpus/` has 78 more TOML files covering every construct in the spec, if
you want to see what a given piece of syntax parses to.

## API

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

## Notes

- **Strict, not last-wins.** Duplicate keys, duplicate table headers, and
  redefining a table as a non-table are errors with a line and column — not
  silently accepted.
- **Datetimes keep their original text.** They are recognised, range-checked,
  and tagged `TomlKind.DateTime`; read them back with `asStr`. This library
  builds no temporal values, because a lossy conversion would be worse than the
  original text.
- **Conformance.** Graded against Python's `tomllib` over a 78-file corpus
  (31 valid, 47 invalid), comparing values *and* types, on both a direct parse
  and a `parse → stringify → parse` round trip. Every TOML v1.0.0 construct is
  implemented: dotted keys, arrays of tables, inline tables, multi-line basic
  and literal strings, integer radixes, float specials, and all four datetime
  forms.

## License

MIT
