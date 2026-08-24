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

Sometimes you may want a nested field without writing a chain of `get` calls. `pick` walks a list of object keys and stops at the first miss (or the first non-object):

```echo
json::Value? $city = $user->pick(['address', 'city']);
```

If `address` is missing, or is not an object, or has no `city`, the result is Echo `null`. A present `"city": null` is still a `Value` you can ask `isNull()` on.

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

## Working With Numbers

A JSON number is not an Echo type. Two cases, classified by the token, not by magnitude:

- `42`, `-0`, `9223372036854775807` become `int`
- `1.0`, `1e2`, `0.5` become `float`

An integer token that does not fit in `int64` is a parse error (`.tooBig`), not a silent `float64`. Snowflake IDs that fit stay exact. Ones that do not stay honest.

`asFloat()` still answers for an `int` (widened), so code that only cares about "a number" does not have to branch.

`dump` of a float always emits a token `parse` will classify as a float. `%g` of `1.0` is `1`; that would come back as an int, so a missing `.` / `e` / `E` gets a `.0` appended.

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
echoc test
```

No network, no extra packages, no C. The suite covers the constructors, the bracket, dump, pretty, RFC fixtures, unicode, and matching the error cases.

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

## Where Things Live

| File | What it holds |
|---|---|
| `src/value.eco` | `Value` (the enum). `Items` / `Fields` own get, slot, set, and clone. |
| `src/access.eco` | `[]` and `[]=` |
| `src/parse.eco` | `json::parse`, the cursor, first-byte dispatch, array / object parsers |
| `src/utf8.eco` | the UTF-8 walk parse uses for strings |
| `src/dump.eco` | `json::dump`, `json::pretty`, a `Writer`, float spelling |
| `src/error.eco`, `src/from.eco` | `Error` (the enum), `Place`, and the `str::from` that makes `"{$e}"` work |
| `src/text.eco` | ASCII names parse and dump compare against |

`LANGUAGE.md` is a historical note about what writing this library asked of Echo. It is not a user guide.
