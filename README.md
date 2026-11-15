# libjson

## Introduction

libjson is JSON for [Echo](https://github.com/echolang/echo). When your application talks to an API, reads a config file, or builds a payload to send back, you need a way to turn text into a value you can walk, and a value back into text.

That is the whole job. Parse it, dump it, poke at it.

The library is written entirely in Echo. There is no C shim, no extra package, and no build step of its own. Add it with epm and start calling `json::parse`.

```echo
json::Value $res = guard json::parse($body) else ($e) {
    die("bad json: {$e}");
};

string $login = $res['login']->asStr() ?? '';
int64  $id    = $res['id']->asInt() ?? 0;

echo "{$login} #{$id}";
```

If some of the pieces here are new (`guard`, `Value`, the `??` after `asStr`), don't lose heart. We'll walk through each one, and every option is documented below.

## Getting Started

### Adding the dependency

From your project directory:

```bash
epm add echolang/libjson --git https://github.com/echolang/libjson --range ^0.1
```

That writes a `#[requires:]` line and vendors the sources. Nothing to link. Echo sees the `json` namespace as soon as the module loads.

libjson needs echoc 0.14.2 or later. Older compilers refuse its `==` on `Value`.

### Your first parse

Let's start with a small API body and pull two fields out of it. `json::parse` is the only way text becomes a `Value`. There is no `decode`, no `load`, and no `fromString` beside it. A second entry point is surface that has to stay consistent with the first one forever, and it is the half that drifts.

```echo
string $body = '{"login":"echolang","id":1,"site_admin":false}';

json::Value $res = guard json::parse($body) else ($e) {
    die("bad json: {$e}");
};

string $login = $res['login']->asStr() ?? '';
int64  $id    = $res['id']->asInt() ?? 0;

echo "{$login} #{$id}";
```

If the text is JSON, you hold a `Value`. If it is not, the `else` arm of `guard` runs with an `Error`. We'll talk more about that error in [Handling Errors](#handling-errors).

### Reading a file first

Sometimes the JSON lives on disk rather than in a string you already have. `parse` does not open files. A missing path is an IO problem, and `std::io::readfile` already answers that. Mixing the two errors would be a second entry point.

You may read the bytes yourself, then hand them to `parse`:

```echo
string $text = guard std::io::readfile('config.json') else ($e) {
    die("{$e}");
};
json::Value $v = guard json::parse($text) else ($e) {
    die("{$e}");
};
```

## Turning Values Back Into Text

Once you hold a `Value`, you will often want to send it, log it, or write it back to disk. Two functions cover that:

```echo
string $compact = json::dump($v);
string $pretty  = json::pretty($v);
```

`dump` is compact RFC 8259, with no trailing newline. `pretty` is the same bytes with newlines and two-space indent.

They are two functions on purpose, not an options struct. Echo has no default parameter values, and a struct for one boolean is surface we would then have to keep.

A well-formed tree cannot fail to dump, so both return `string` rather than `result`. If you construct a `NaN` or an infinity by hand and try to dump it, that is a `die`. Those are not JSON numbers.

For convenience, interpolating a `Value` is the compact dump:

```echo
echo "payload: {$v}";
```

## Reading and Writing BSON

Sometimes the document arrives as BSON rather than text: a MongoDB dump, a `.bson` file, a message another service packed. You don't need a second tree for that. BSON reads into the same `Value`, and a `Value` writes back out as BSON.

```echo
use json::bson;

string $bytes = guard std::fs::readfile('user.bson');
json::Value $doc = guard json::parse<bson>(bytes: $bytes);

string $id = $doc['_id']['$oid']->asStr() ?? '';
$doc['seen'] = true;

string $out = guard json::dump<bson>(value: $doc);
```

`bson` is the format, passed as a type. The `use` line only shortens `json::bson` to `bson`. The labels `bytes:` and `value:` are required. They are what keeps these two apart from the text `parse` and `dump`, which take the same argument types.

Everything else works as it does for text. Reading errors are the same `Error` and go through the same `guard`, and the tree you get back is an ordinary `Value` you can index, walk, compare, and `json::dump` as text.

### How the types map

The kinds JSON shares with BSON map straight across. int32 and int64 both read as an `int`, and on the way out an `int` is written as an int32 whenever it fits.

The kinds JSON has no word for are read as [MongoDB Extended JSON](https://www.mongodb.com/docs/manual/reference/mongodb-extended-json/) objects, so nothing is lost and the tree stays plain JSON:

| BSON | Reads as |
|---|---|
| ObjectId | `{"$oid": "65a1f0c2e4b0a1b2c3d4e5f6"}` |
| datetime | `{"$date": {"$numberLong": "1700000000000"}}` (milliseconds since the epoch) |
| binary | `{"$binary": {"base64": "AAEC", "subType": "00"}}` |
| regex | `{"$regularExpression": {"pattern": "^a", "options": "i"}}` |
| timestamp | `{"$timestamp": {"t": 1700000000, "i": 1}}` |
| JavaScript code | `{"$code": "..."}`, or `{"$code": "...", "$scope": {...}}` with a scope |
| min / max key | `{"$minKey": 1}` / `{"$maxKey": 1}` |
| a NaN or infinite double | `{"$numberDouble": "NaN"}`, `"Infinity"` or `"-Infinity"` |
| the deprecated ones | `{"$undefined": true}`, `{"$symbol": "..."}`, `{"$dbPointer": {"$ref": "...", "$id": {"$oid": "..."}}}` |

The non-finite doubles get a wrapper so that a `json::dump` of a tree you read from BSON can never die on a `NaN`.

Writing goes the other way. An object that has exactly one of these shapes becomes that BSON type again, so you build an ObjectId or a date the same way you would read one:

```echo
$doc['createdAt'] = json::object(['$date' => json::object(['$numberLong' => '1700000000000'])]);
```

There are two more wrappers that only matter on the way out. `{"$numberLong": "5"}` writes an int64 even though `5` would fit in an int32. `{"$numberDouble": "2.5"}` writes a double from its text.

The match is exact: the right keys, nothing extra, every value of the right kind. `{"$oid": "nope"}` is not an ObjectId. It is an ordinary document with a key called `$oid`, and it is written as one. The same goes for a `$timestamp` that does not fit in 32 bits, or a regex whose pattern or options hold a `0x00` byte: BSON cannot write those as the type, so they go out as the documents they are. Nothing is refused for looking almost like a wrapper.

### What does not come back byte for byte

Read BSON, then write it, and you get the same bytes back with two exceptions:

- An int64 that fits in 32 bits comes back as an int32. The number is the same. Wrap it in `$numberLong` if the width matters to whoever reads it next.
- Array keys are written as `"0"`, `"1"`, ... whatever they were on the way in. BSON says they must be, and the reader does not look at them.

Regex options are written in alphabetical order, as the spec asks, so `"mix"` goes out as `"imx"`.

### When writing BSON fails

Text `dump` cannot fail, but some trees cannot be BSON at all, so `dump<bson>` returns a `result` with a `json::DumpError`:

| Problem | Case |
|---|---|
| the root is not an object (BSON has no top-level number or array) | `notObject` |
| a key holds a `0x00` byte | `nulByte($pointer)` |

JSON allows `"\u0000"` in a key, and BSON ends every key with that byte, so there is no way to write one. This is the case to expect when you turn untrusted JSON into BSON. The pointer is the RFC 6901 path to the member whose key it is, so `{"a": [{"x\u0000": 1}]}` fails with `/a/0/x` followed by the `0x00`.

```echo
string $out = guard json::dump<bson>(value: $doc) else ($e) {
    echo "not stored: {$e}";
    return;
}
```

`"{$e}"` reads as the message, like the other errors. Match on `.nulByte($at)` when you want the pointer itself.

### Reading errors

A BSON error is one of the [Error cases](#error-cases) you already match on. Its `Place` points at a byte: line is always 1, column is the offset plus one, and `offset()` is the number you want.

| Problem | Case |
|---|---|
| no bytes at all | `empty` |
| the input ends before a length says it should | `unterminated` |
| lengths that disagree, a missing terminator, an unknown type byte, a boolean that is not 0 or 1 | `unexpected` |
| a string or key that is not UTF-8 | `badUtf8` |
| documents nested past 128, the root counted | `tooDeep` |
| bytes after the document | `trailing` |

## Handling Errors

### Using `guard`

`parse` answers `result<Value, Error>`, which is what the whole of `std::io` does. You are free to inspect the result yourself, but `guard` is the short way through: on success you get the `Value`, and on failure the `else` arm runs with the `Error`.

```echo
json::Value $v = guard json::parse($text) else ($e) {
    die("bad json: {$e}");
};
```

A `Value` exists only when the text was JSON. Truncated input is an `Error`. `"x": null` is a perfectly good `Value` whose `isNull()` answers true. Those are different outcomes, and the type system keeps them apart.

### Matching on the case

`Error` is an enum, so the `else` arm of `guard` is something you can `match`. Sometimes you may want to treat one failure specially (a number that will not fit, nesting that went too deep) and give everything else the same sentence:

```echo
json::Value $v = guard json::parse($text) else ($e) {
    match ($e) {
        .tooBig($at) => {
            die("number too large at {$at->line}:{$at->column}");
        },
        .tooDeep($at) => {
            die("nested too deep");
        },
        else => {
            die("bad json: {$e}");
        },
    }
};
```

Don't worry about memorizing the cases. The set is closed, and each one is listed in [Error cases](#error-cases) below.

### Reading the place

Every case carries a `Place`: the `$line`, `$column`, and `$offset` of the offending byte. Line and column are 1-based. Offset is the 0-based byte index into the input. Column is counted in bytes, not graphemes.

You may pull those numbers off the error without matching:

```echo
int64 $line   = $e->line();
int64 $column = $e->column();
int64 $offset = $e->offset();
```

`"{$e}"` renders the sentence from the case. There is a `str::from` overload for it, so interpolation and `$e->message()` say the same thing:

```
empty input at 1:1
unexpected token at 3:12
```

### Error cases

Each case is a named way this parser refuses input. There is no leftover "other" variant: the set is closed, and we construct the case ourselves.

#### `empty`

The input was only whitespace, or nothing at all.

#### `unexpected`

The next byte was not the start of a value, or a control character appeared inside a string.

#### `unterminated`

The input ended in the middle of a value: an open string, an unfinished literal, a truncated `\u` escape.

#### `trailing`

A complete value was followed by more non-whitespace. `parse` accepts one value, not a stream of them.

#### `badNumber`

The token looked like a number and then wasn't: a leading zero on a multi-digit integer, a lone `.`, an exponent with no digits.

#### `tooBig`

An integer token does not fit in `int64`, or a float overflowed to infinity. Snowflake IDs that fit stay exact. Ones that do not stay honest.

#### `tooDeep`

Nesting went past 128. This is a parse error, not a stack overflow.

#### `badEscape`

A `\...` sequence was not a legal JSON escape, or a `\uXXXX` did not decode to a real scalar. Lone surrogates land here.

#### `badUtf8`

A string contained a byte sequence that is not UTF-8.

## Working With Values

A JSON value is a `json::Value` enum. A number is not as wide as an object. Arrays and objects share a small box, so **assignment still shares the node.** `$b = $a; $b['x'] = 1;` mutates `$a` too.

That is the PHP / JS bargain, and it is the one that makes `$obj['name'] = ...` a real write rather than a copy you throw away.

If that sharing sounds alarming, it is the same rule you already live with in those languages. `$v->clone()` is the deep copy, written out so sharing is never implied by `=`.

```echo
json::Value $a = json::object();
$a['x'] = json::int(1);

json::Value $b = $a->clone();
$b['x'] = json::int(2);

// $a still holds 1
```

A cycle (only possible if you tie a node to an ancestor through `[]=`) overflows the stack, the same way `dump` would.

## Building a Document

Sometimes you are not parsing anything. You are assembling a payload to send. Static constructors build each kind, and the leading-dot shorthand works wherever the destination already names `Value`:

```echo
json::Value $user = json::object();
$user['name']  = 'mario';
$user['id']    = 32;
$user['ok']    = true;
$user['score'] = 1.5;
$user['extra'] = json::nil();
$user['tags']  = json::arr();
$user['tags'][] = 'echo';
$user['tags'][] = 'json';
```

Two names are not the JSON ones, because Echo already spent those words:

| JSON | This library | Why |
|---|---|---|
| `null` | `json::nil()` | `null` is reserved |
| `[]` | `json::arr()` | `array` is already the container type, and a function of that name would steal every `array<T>` |

`json::bool`, `json::int`, `json::float`, `json::str`, and `json::object` keep their names. `.nil()` and `.int(7)` work when the destination is a `Value`. For convenience, `.array()` is also safe: the name lives on `Value`, so it does not hide the root `array<T>` type the way a free `json::array()` would.

A bare `'mario'`, `32`, `true`, or `1.5` on the right of `[]=` becomes a `Value` on the way in, so `$user['id'] = 32` and `$tags[0] = 'echo'` do not need a wrap.

`$user['tags'][] = 'echo'` works: the first bracket reads the array, and append mutates the shared box.

### From Echo containers

Often the data is already sitting in an Echo array or map. You don't have to push it in one item at a time. An `array`, an `ordered_map<string, …>`, or a `map<string, …>` of strings, ints, floats, bools, or `Value`s converts on the way in, just like a bare scalar does:

```echo
array<string> $tags = ['echo', 'json'];
ordered_map<string, int64> $scores = ['ann' => 3, 'bob' => 5];

$user['tags']   = $tags;     // ["echo","json"]
$user['scores'] = $scores;   // {"ann":3,"bob":5}
```

An `ordered_map` keeps its order. A plain `map` has no order of its own, so its keys go in sorted, and two dumps of the same map agree.

For a document with mixed kinds, `json::arr` and `json::object` take a literal. Each element converts as it goes in, which is the closest thing to nlohmann's `{{"a", 1}}`:

```echo
json::Value $user = json::object([
    'name' => 'mario',
    'id'   => 32,
    'tags' => json::arr(['echo', 1, true, json::nil()]),
]);
```

Hand the literal to `json::arr` / `json::object`, or put it in a typed variable first. A bare `$user['tags'] = ['echo', 'json'];` is refused by the compiler today: the literal fits more than one conversion (`array<string>` and `array<Value>` both take it), and echoc does not pick one yet.

Writing `[]=` on a non-object, appending to a non-array, or writing past the end of an array is a `die`. That is a bug in the caller, not a parse failure.

`$user->count()` answers how many entries an array or object holds, and zero for every other kind.

## Reading a Document

Reads and writes both use the bracket. Echo lets a type declare two contracts over `[]`, and the *position* chooses: `$v['name'] = 'mario'` is `[]=`, `$v['name']->asStr()` is the borrow.

A missing key has no slot to hand back, so the place is a shared JSON `null`. `$v['nope']->asStr()` answers without inserting. That is how a read stays a read.

Passing a missing-key place into a `Value&` parameter writes that shared null. Don't do that. Use `[]=` to insert.

### Telling a missing key from JSON `null`

A missing key is Echo `null`. JSON `null` is a `Value` whose `isNull()` is true. Those are different things, and `get` keeps them apart:

```echo
json::Value? $email = $user->get('email');

if ($email == null) {
    // key was not there
} else if ($email->isNull()) {
    // "email": null
}

string $name = $user->get('name')?->asStr() ?? '';
int64  $age  = $user->get('age')?->asInt() ?? 0;
```

`get($key)` is for objects. `at($i)` is for arrays. Either one answers Echo `null` when the receiver is the wrong shape or the slot is not there.

So, what happens if you use the bracket instead? `$user['missing']->isNull()` is true, and so is `$user['present']->isNull()` when the JSON said `"present": null`. Reach for `get` or `at` when you need to tell those two apart.

### Walking a path

Sometimes you may want a nested field without writing a chain of `get` calls. `pick` walks a path and stops at the first miss:

```echo
json::Value? $city = $user->pick(['address', 'city']);
json::Value? $first = $res->pick(['users', '0', 'name']);
```

On an object a segment is a key. On an array it is an index, spelled the way JSON Pointer spells one: `0`, or digits with no leading zero. If anything along the way is missing, or is a scalar, the result is Echo `null`. A present `"city": null` is still a `Value` you can ask `isNull()` on.

`pick` also takes a [JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901) (RFC 6901) as one string:

```echo
json::Value? $name = $res->pick('/users/0/name');
```

`''` is the whole document. Inside a key, `~1` stands for `/` and `~0` for `~`, so `/a~1b` reads the key `a/b`. A string that is not a pointer (no leading `/`, or a `~` followed by anything but `0` or `1`) finds nothing, and so does `-`, which names the slot after the last element.

### Flattening

`flatten()` turns a whole tree into one object that maps each leaf's pointer to the leaf. `unflatten()` builds the tree back:

```echo
json::Value $doc = guard json::parse('{"a":{"b":[1,2]},"tags":[]}');

echo json::dump($doc->flatten());   // {"/a/b/0":1,"/a/b/1":2,"/tags":[]}

json::Value $back = guard $doc->flatten()->unflatten();
```

An empty array or object stays a leaf of its own, rather than turning into `null` the way nlohmann flattens it, so the round trip gives back exactly the tree you started with.

`unflatten` answers `result<Value, PointerError>`, because the object you hand it might not have come from `flatten`. Where every segment under a node is an index, the node is an array, and a gap fills with `null`. One segment that is not an index makes it an object, so `{"a":{"0":1,"x":2}}` comes back as the object it was. The order of the keys never changes the result. The cases:

| Case | When |
|---|---|
| `notObject` | the receiver is not an object |
| `malformed($pointer)` | a key is not a JSON Pointer |
| `notLeaf($pointer)` | a value is a non-empty array or object |
| `conflict($pointer)` | the pointer runs through a place another key ends at, such as `/a/b` beside `/a`. The deeper pointer is the one named |
| `outOfRange($pointer)` | an index is past the number of members, which keeps a hostile `/a/99999999999` from filling memory with `null`s |
| `tooDeep($pointer)` | more than 128 segments |

`"{$e}"` and `$e->message()` give the sentence, same as `Error`.

## Asking What a Value Is

When a field might be several shapes, you will want to ask before you take it out. Each predicate answers for one kind. `isNumber()` is the convenience for "int or float":

```echo
$v->isNull()
$v->isBool()
$v->isInt()
$v->isFloat()
$v->isNumber()   // int or float
$v->isStr()
$v->isArray()
$v->isObject()
```

## Taking Values Out

Once you know (or don't mind being wrong about) the shape, the `as*` methods hand back the Echo value. Wrong shape is Echo `null`, not an `Error`. You already parsed. A missing field in a response is ordinary.

```echo
$v->asBool()   : bool?
$v->asInt()    : int64?     // int only. a 1.0 stays a float
$v->asFloat()  : float64?   // int widens, float as-is
$v->asStr()    : string?
$v->asArray()  : array<json::Value>?
$v->asObject() : ordered_map<string, json::Value>?
```

`asArray` and `asObject` copy the container and share the children. Mutating a child still reaches the original tree. Replacing the container you were handed does not.

If you only want to walk the container, you don't need that copy at all. See [Iterating](#iterating).

## Iterating

`elements()` walks an array and `members()` walks an object, without copying either one:

```echo
foreach ($res['users']->elements() as $i => const &$user) {
    string $name = $user->get('name')?->asStr() ?? '';
    echo "{$i}: {$name}";
}

foreach ($config->members() as $key => const &$value) {
    echo "{$key} = {$value}";
}
```

The loop holds the shared box, and each item is the same place `[]` would name, so `as &$item` writes reach the tree. A `const &` item reads through `get` / `at` and the `as*` methods; the bracket wants a place it could write. `members()` goes in key order and copies only the list of keys. On any other kind, either loop runs zero times.

If you only need the names, `keys()` hands back the object's keys in order (empty for anything that is not an object).

As with any Echo container, don't add or remove entries while you are iterating. Collect what you want to change and apply it after the loop.

## Comparing Values

`==` and `!=` compare two values by content, all the way down:

```echo
json::Value $a = guard json::parse('{"a":1,"b":[1,2]}');
json::Value $b = guard json::parse('{"b":[1,2],"a":1}');

$a == $b            // true: key order does not matter
$a['a'] == 1        // true: a bare scalar converts
$a['b'] != 'x'      // true
```

Three rules, the same ones nlohmann uses:

- Arrays compare element by element, in order.
- Objects compare as sets of members, so `{"a":1,"b":2}` equals `{"b":2,"a":1}`.
- Numbers compare by value across int and float, so `1` equals `1.0`. The int is widened to a `float64` first, the same widening `asFloat()` does. Two ints beyond 2^53 that widen to the same double therefore compare equal.

## Removing and Merging

`remove($key)` takes a member out of an object, and `removeAt($i)` takes an element out of an array, shifting the rest down. Both answer whether there was anything to remove, so a missing key, an index past the end, or the wrong kind is `false` rather than a `die`. `clear()` empties an array or object in place, which every alias of that box sees too.

```echo
$user->remove('password');
$user['tags']->removeAt(0);
$cache->clear();
```

To combine two documents, there are three verbs:

| Call | What it does |
|---|---|
| `$a->update($b)` | copies every member of `$b` into `$a`, overwriting keys `$a` already has |
| `$a->merge($b)` | `update`, recursively: where both sides hold an object under the same key, that object is merged rather than replaced |
| `$a->extend($b)` | appends every element of array `$b` to array `$a` |

```echo
json::Value $config = guard json::parse('{"db":{"host":"localhost","port":5432}}');
json::Value $local  = guard json::parse('{"db":{"port":6543}}');

$config->merge($local);   // {"db":{"host":"localhost","port":6543}}
```

A rewritten key keeps its place, and a new one goes last. What comes in is cloned, so changing `$b` afterwards never reaches `$a`. `update` and `merge` need two objects, and `extend` needs two arrays. Anything else is a `die`, the same as `[]=` on a non-object.

## Working With Numbers

A JSON number is not an Echo type. Two cases, classified by the token, not by magnitude:

- `42`, `-0`, `9223372036854775807` become `int`
- `1.0`, `1e2`, `0.5` become `float`

An integer token that does not fit in `int64` is a parse error (`.tooBig`), not a silent `float64`. Snowflake IDs that fit stay exact. Ones that do not stay honest.

`asFloat()` still answers for an `int` (widened), so code that only cares about "a number" does not have to branch.

`dump` writes a float as the shortest text that reads back as the same double, so a dump then parse hands back exactly the bits that went in. `0.30000000000000004` stays `0.30000000000000004`, `0.1` stays `0.1`, and `5e-324` stays `5e-324`.

The token is always one `parse` will classify as a float. `1.0` dumps as `1.0`, not `1`, which would come back as an int. Below `1e15` a float is spelled out in full (`100.0`, `150000000000000.0`), and from there on it takes an exponent (`1e+15`, `1e+300`).

## What `parse` Accepts

RFC 8259, nothing else.

- One value, optional surrounding whitespace.
- Objects, arrays, strings, numbers, `true` / `false` / `null`.
- Strings are UTF-8. `\uXXXX` (and a surrogate pair) must decode to a real scalar. Lone surrogates are `.badEscape`. Invalid UTF-8 is `.badUtf8`.
- Duplicate keys: last write wins, first-seen order kept. A dump walks keys in the order they arrived, so a pretty-printed object does not reshuffle itself.
- No comments, no trailing commas, no single quotes, no `NaN` / `Infinity`.
- Nesting past 128 is `.tooDeep`, not a stack overflow.

`dump` and `pretty` of a well-formed tree cannot fail. They return `string`. A `NaN` or an infinity you constructed by hand is a `die`. Those are not JSON numbers.

`"{$v}"` is compact dump. `"{$e}"` is the error sentence.

## Running the Tests

```bash
epm install
echoc test
```

No network, no C. The suite covers the constructors, the bracket, dump, pretty, float round trips, RFC fixtures, unicode, matching the error cases, equality, iteration, removing and merging, and the RFC 6901 pointer examples. `tests/bsoncorpus.eco` is MongoDB's BSON corpus for every type `bson` reads: each valid case has to read and write back its canonical bytes, and each malformed one has to be refused. Its checks are [ecotest](https://github.com/echolang/ecotest)'s `expect`, which only the test target requires, so a consumer of libjson never compiles it.

For coverage, run the ecotest runner:

```bash
epm run ecotest --coverage --min-coverage 100 --min-branch-coverage 100
```

That is the gate CI holds on macOS and Linux: every line of `src/` runs and every branch goes both ways, and the build fails when one stops. Windows runs the same runner without coverage.

## Trying the Examples

Two small programs live under `examples/`. The first is the parse we started with. The second builds a document and pretty-prints it.

```bash
echoc run -m . examples/parse.eco
echoc run -m . examples/build.eco
```

## What Is Not Here Yet

These are deliberate postponements, not accidents.

- **Streaming.** The tree is built in memory. Same postponement as a lot of first libraries.
- **Comments, trailing commas, JSON5, JSON Lines.** Strict RFC 8259. A later `parseRelaxed` would be a second verb with its own tests, not a flag that changes `parse`.
- **Typed decode into your structs.** Echo structs have no runtime metadata, so this would be an interface people implement by hand. Useful, and a follow-up.
- **A byte cap.** A hostile payload can grow the tree until the process dies.
- **BSON decimal128.** Reading it needs its own decoding, so its type byte is refused as `.unexpected` for now. It would read as `{"$numberDecimal": "..."}`.
- **Relaxed dates.** A BSON date reads as `{"$date": {"$numberLong": "..."}}`, not as an ISO-8601 string.

## Where Things Live

| File | What it holds |
|---|---|
| `src/value.eco` | `Value` (the enum). `Items` / `Fields` own get, slot, set, remove, and clone. |
| `src/access.eco` | `[]` and `[]=` |
| `src/convert.eco` | Echo arrays and maps into a `Value` |
| `src/cursor.eco` | `Elements` and `Members`, the cursors behind `elements()` / `members()` |
| `src/equal.eco` | `==` / `!=` on `Value` and the deep comparison behind them |
| `src/merge.eco` | `update`, `merge`, `extend` |
| `src/pointer.eco` | JSON Pointer, `flatten` / `unflatten`, and `PointerError` |
| `src/parse.eco` | `json::parse`, the cursor, first-byte dispatch, array / object parsers |
| `src/utf8.eco` | the UTF-8 walk parse uses for strings |
| `src/dump.eco` | `json::dump`, `json::pretty`, a `Writer`, float spelling |
| `src/format.eco` | `Format`, `DumpError`, and the labelled `json::parse<F>` / `json::dump<F>` |
| `src/bson.eco` | `json::bson`, the BSON type bytes |
| `src/bsonread.eco` | the BSON reader, length-checked against the input and the enclosing document |
| `src/bsonwrite.eco` | the BSON writer and the Extended JSON wrapper match |
| `src/base64.eco` | base64 for `$binary` |
| `src/error.eco`, `src/from.eco` | `Error` (the enum), `Place`, and the `str::from` that makes `"{$e}"` work for it and for `PointerError` |
| `src/text.eco` | ASCII names parse and dump compare against |

`LANGUAGE.md` is a historical note about what writing this library asked of Echo. It is not a user guide.
