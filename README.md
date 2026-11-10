# egg-json

JSON (RFC 8259) for [Egg](https://github.com/femto/egg): parse a document
and read it, or build one.

```
clutch add json
```

```egg
uses io, alloc

const json = @import("json")

fn main(io: Io, a: Allocator) !void {
    const doc = try json.parse(a, "{\"user\": {\"name\": \"egg\", \"tags\": [\"a\", \"b\"]}}")
    defer a.free(doc)
    const user = doc.get(doc.root(), "user") orelse return
    io.print(doc.str(doc.get(user, "name") orelse return) orelse "?")
    var it = doc.items(doc.get(user, "tags") orelse return)
    while (it.next(doc)) |tag| {
        io.print(doc.str(tag) orelse "?")
    }

    var b = json.Builder.init()
    b.begin_object(a)
    b.key(a, "ok")
    b.boolean(a, true)
    b.end_object(a)
    const out = try b.finish(a)
    defer a.free(out)
    io.print(out)          // {"ok":true}
}
```

## Reading

`json.parse(a, src) !Doc` reads the document that is all of `src`.
`json.parse_at(a, src, &at)` also says where it stopped, which on an error
is the byte offset where `src` stops being JSON. The errors are
`error.Syntax` and `error.TooDeep` (arrays and objects nested deeper than
512).

A `Doc` owns everything it read, in two buffers: its nodes, every value in
the order it was written, and the text of its strings with their escapes
undone. `a.free(doc)` frees both. A `Value` names one node. It is a plain
value, copied freely, and meaningful only with its doc. There is no tree of
small allocations, because an Egg union cannot hold a buffer. The layout is
simdjson's tape.

| `doc.` | gives |
| --- | --- |
| `root()` | the whole document's value |
| `kind(v)` | `.nil`, `.boolean`, `.number`, `.string`, `.array` or `.object` |
| `is_null(v)` | whether v is `null` |
| `boolean(v)` | `?bool` |
| `int(v)` | `?i64`: a number written without a fraction or an exponent, that fits an i64 |
| `float(v)` | `?f64`: any number, rounded to the nearest f64 (inf when too big) |
| `str(v)` | `?[]u8`: a string's text, a part of the doc |
| `len(v)` | the number of elements or members (0 for anything else) |
| `get(v, key)` | `?Value`: an object's member; the last one when a name repeats, as in JavaScript |
| `at(v, i)` | `?Value`: an array's element i (it walks the elements before it) |
| `items(v)` | the elements of an array: `while (it.next(doc)) \|e\| { ... }` |
| `members(v)` | the members of an object: `while (it.next(doc)) \|m\| { m.key, m.value }` |

The parser is strict:
- no trailing commas, comments, single quotes, `NaN` or leading zeros;
- strings must be valid UTF-8 with no raw control characters;
- a `\u` escape that is half of a surrogate pair is an error;
- white space is allowed only around values.

## Building

A `Builder` appends one document to a buffer. It writes the commas and
colons, and `quote` escapes the strings.

| `b.` | writes |
| --- | --- |
| `begin_object(a)`, `end_object(a)`, `begin_array(a)`, `end_array(a)` | a container |
| `key(a, k)` | the name of an object's next member |
| `str(a, s)`, `int(a, n)`, `float(a, x)`, `boolean(a, t)`, `nil(a)` | a value (`float` writes inf and nan as `null`, as JavaScript does) |
| `raw(a, text)` | JSON text you made, as one value |
| `finish(a)` | `!Buf(u8)`: the document |

Calls out of order are reported by `finish`: `error.Misuse` for a value
where a key belongs, a key outside an object, a second top-level value or
the wrong `end_`; `error.Incomplete` for a container left open, or for
nothing written at all. `json.quote(a, &out, s)` appends `s` as a JSON
string by itself.

## Tests

```
cd test && clutch run
```

Needs an eggc with `std/text`'s `parse_float` (egg #98).
