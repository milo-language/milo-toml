# toml

A [TOML v1.0.0](https://toml.io/en/v1.0.0) parser and serializer for
[Milo](https://github.com/milo-language/milo). No dependencies beyond the
standard library.

```bash
milo add github.com/milo-language/milo-toml
```

```milo
from "toml" import { Toml }

fn main(): i32 {
    let doc = Toml.parse("name = \"myapp\"\n[server]\nport = 8080\n")!
    print(doc.str("name") ?? "unnamed")
    print((doc.i64Path("server.port") ?? 80).toString())
    return 0
}
```

## Handles, not trees

A document parses into a flat node pool; a `Toml` is a lightweight handle into
it. Milo has no pointers in safe code, so a recursive value type would need an
arena anyway — and a pool keeps the whole document one contiguous, cheaply
cloned value. This is the same shape `std/json` uses, and the accessor
vocabulary matches it deliberately:

```milo
doc.get("key")          doc.at(3)             doc.len()
doc.str("k")            doc.i64("k")          doc.f64("k")   doc.bool("k")
doc.table("k")          doc.keys()
doc.path("a.b.c")       doc.strPath("a.b")    doc.i64Path("a.b")
doc.asStr()             doc.isArray()         doc.kind()
```

## Conformance

Graded against Python's `tomllib` by `scripts/oracle.py` over a 78-file corpus
(31 valid, 47 invalid), comparing **values and types** — not just shape — on
both a direct parse and a `parse → stringify → parse` round trip.

Every TOML v1.0.0 construct is implemented: dotted keys, `[[arrays of tables]]`,
inline tables, multi-line basic and literal strings, integer radixes, float
specials, and all four datetime forms. Duplicate keys, duplicate tables, and
redefining a table as a non-table are errors with line and column.

**Datetimes carry their lexical text**, tagged `TomlKind.DateTime` and read back
with `asStr`. This library builds no temporal values — a lossy conversion would
be worse than the original text. Richer typing over `std/datetime` is future
work.

## License

MIT
