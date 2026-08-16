# toml

This is a package for the [Milo language](https://milo-language.github.io/milo/).

## Overview

A [TOML v1.0.0](https://toml.io/en/v1.0.0) parser and serializer, with no
dependencies beyond the standard library.

Every accessor returns an `Option`, so `??` supplies a default in one line and
there is no missing-key branch to write. `Toml.parse` returns a `Result` whose
error carries a message plus the line and column.

It is strict rather than last-wins: duplicate keys, duplicate table headers, and
redefining a table as a non-table are all errors, not silently accepted.
Datetimes are recognised, range-checked and tagged `TomlKind.DateTime`, but kept
as their original text and read back with `asStr` — this library builds no
temporal values, because a lossy conversion would be worse than the text.

Every TOML v1.0.0 construct is implemented, and the whole thing is graded
against Python's `tomllib` over a 78-file corpus on both a direct parse and a
`parse → stringify → parse` round trip.

Every function and method: [docs/api.md](docs/api.md).

## Installation

```bash
milo add github.com/milo-language/milo-toml
```

```milo
from "toml" import { Toml }
```

## Examples

### Reading a config file

Given `app.toml`:

```toml
name = 'checkout-api'
debug = false

[server]
host = '0.0.0.0'
port = 8080
```

```milo
from "toml" import { Toml }
from "std/fs" import { readFile }

pub fn main(): i32 {
    let cfg = Toml.parse(readFile("app.toml")!)!

    let name = cfg.str("name") ?? "unnamed"          // top-level string
    let debug = cfg.bool("debug") ?? false           // top-level bool
    let port = cfg.i64Path("server.port") ?? 8080    // dotted path into [server]
    let ratio = cfg.f64Path("limits.ratio") ?? 1.0   // absent, so the default answers

    print($"{name} debug={debug} port={port} ratio={ratio}")

    // A sub-table can be pulled out and reused as its own handle.
    let server = cfg.table("server")!
    print($"{server.str("host") ?? "127.0.0.1"}:{server.i64("port") ?? 80}")
    return 0
}
```

```
checkout-api debug=false port=8080 ratio=1
0.0.0.0:8080
```

### Arrays and arrays of tables

`[[route]]` blocks come back as an array, in document order. Arrays and tables
are both handles, so the same `len`/`at` walk reaches either:

```milo
let cfg = Toml.parse("
tags = ['web', 'api']

[[route]]
path = '/health'
methods = ['GET']

[[route]]
path = '/orders'
methods = ['GET', 'POST']
")!

let tags = cfg.get("tags")!
for i in 0..tags.len() {
    print(tags.at(i)!.asStr() ?? "")
}

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

```
web
api
/health
  GET
/orders
  GET
  POST
```

### Writing TOML

`stringify` renders a document back to text that re-parses to an equal document,
and rendering that again is byte-identical:

```milo
let doc = Toml.parse("
[server]
port = 30_000
host = 'localhost'
")!

let text = doc.stringify()
let again = Toml.parse(text.clone())!

print(text)
print($"round-trips: {text == again.stringify()}")
```

```
[server]
port = 30000
host = "localhost"

round-trips: true
```

Formatting is not preserved, which the output above shows: comments are dropped,
`30_000` comes back as `30000`, and `'literal'` strings come back
double-quoted. Key order, values, types and structure are preserved.

A complete program against a real service config:
`milo run examples/config.milo examples/app.toml`.
