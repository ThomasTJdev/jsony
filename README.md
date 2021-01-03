# JSONy - A loose, direct to object json parser and serializer with hooks.

`nimble install jsony`

```nim
@[1, 2, 3].toJson() -> "[1,2,3]"
"[1, 2, 3]".fromJson(seq[int]) -> @[1, 2, 3]
```

Real world json is *never what you want*. It might have extra fields that you don't care about. It might have missing fields requiring default values. It might change or grow new fields at any moment. Json might use `camelCase` or `snake_case`. It might use inconsistent naming.

With this library you can use json your way, from the mess you get to the objects you want.

## Fast.

Currently the Nim standard module first parses or serializes json into JsonNodes and then turns the JsonNodes into your objects with the `to()` macro. This is slower and creates unnecessary work for the garbage collector. This library skips the JsonNodes and creates the objects you want directly.

Another speed up comes from not using `StringStream`. `Stream` has a function dispatch overhead because it has not know if you are using it as a `StringStream` or `FileStream`. Jsony skips the overhead and just directly writes to memory buffers.

### Parse speed.
```
name ............................... min time      avg time    std dv  times
treeform/jsony .................... 10.121 ms     10.809 ms    ±1.208   x100
planetis-m/eminim ................. 12.006 ms     12.574 ms    ±1.156   x100
nim std/json ...................... 23.282 ms     34.679 ms    ±7.512   x100
```

### Serialize speed.
```
name ............................... min time      avg time    std dv  times
treeform/jsony ..................... 4.047 ms      4.172 ms    ±0.225   x100
planetis-m/eminim .................. 7.173 ms      7.324 ms    ±0.253   x100
disruptek/jason ................... 10.220 ms     11.155 ms    ±0.689   x100
nim std/json ...................... 11.526 ms     15.181 ms    ±0.857   x100
```

Note: If you find a faster nim json parser or serializer let me know!

## Can parse or serializer most types:

* numbers and strings
* objects and ref objects
* enums
* tuples
* seq and arrays
* tables
* and `parseHook()` enables you to parse any type!

## Not strict.

Extra json fields are ignored and missing json fields keep their default values.

```nim
type Entry1 = object
  color: string
var s = """{"extra":"foo"}"""
var v = fromJson[Entry1](s)
doAssert v.color == ""
```

## Converts snake_case to camelCase.

Nim usually uses `camelCase` for its variables, while a bunch of json in the wild uses `snake_case`. This library will convert `snake_case` to `camelCase` for you when reading json.

```nim
type Entry4 = object
colorBlend: string

var v = fromJson[Entry4]("""{"colorBlend":"red"}""")
doAssert v.colorBlend == "red"

v = fromJson[Entry4]("""{"color_blend":"red"}""")
doAssert v.colorBlend == "red"
```

## Has hooks.

### `proc newHook()` Can be used to populate default values.

Sometimes the absence of a field means it should have a default value. Normally this would just be Nim's default value for the variable type but that isn't always what you want. With the newHook() you can initialize the object's defaults before the main parsing happens.

```nim
type
  Foo5 = object
    visible: string
    id: string
proc newHook(foo: var Foo5) =
  # Populates the object before its fully deserialized.
  foo.visible = "yes"

var s = """{"id":"123"}"""
var v = fromJson[Foo5](s)
doAssert v.id == "123"
doAssert v.visible == "yes"
```

### `proc postHook()` Can be used to run code after the object is fully parsed.

Some times we need run some code after the object is created. For example to set other values based on values that where set but are not part of the json data. Maybe to sanitize the object or convert older versions to new versions. Here I need to retain the original size as I will be messing with the object's regular size:

```nim
type Sizer = object
  size: int
  originalSize: int

proc postHook(v: var Sizer) =
  v.originalSize = v.size

var sizer = fromJson[Sizer]("""{"size":10}""")
doAssert sizer.size == 10
doAssert sizer.originalSize == 10
```

### `proc enumHook()` Can be used to parse enums.

In the wild json enum names almost never match to Nim enum names which usually have a prefix. The enumHook() allows you to rename the enums to your internal names.

```nim
type Color2 = enum
  c2Red
  c2Blue
  c2Green

proc enumHook(v: string): Color2 =
  case v:
  of "RED": c2Red
  of "BLUE": c2Blue
  of "GREEN": c2Green
  else: c2Red

doAssert fromJson[Color2](""" "RED" """) == c2Red
doAssert fromJson[Color2](""" "BLUE" """) == c2Blue
doAssert fromJson[Color2](""" "GREEN" """) == c2Green
```

### `proc renameHook()` Can be used to rename fields at run time.

In the wild json field names can be reserved words such as type, class, or array. With the renameHook you can rename fields to what you want.

```nim
type Node = ref object
  kind: string

proc renameHook(v: var Node, fieldName: var string) =
  if fieldName == "type":
    fieldName = "kind"

var node = fromJson[Node]("""{"type":"root"}""")
doAssert node.kind == "root"
```

### `proc parseHook()` Can be used to do anything.

Json can't store dates, so they are usually stored as strings. You can use
`parseHook()` to override default parsing and parse `DateTime` as a `string`:

```nim
proc parseHook(s: string, i: var int, v: var DateTime) =
  var str: string
  parseHook(s, i, str)
  v = parse(str, "yyyy-MM-dd hh:mm:ss")

var dt = fromJson[DateTime](""" "2020-01-01 00:00:00" """)
```

Sometimes json gives you an object of entries with their id as keys, but you might want it as a sequence with ids inside the objects. You can handle this and many other scenarios with `parseHook()`:

```nim
type Entry = object
  id: string
  count: int
  filled: int

let data = """{
  "1": {"count":12, "filled": 11},
  "2": {"count":66, "filled": 0},
  "3": {"count":99, "filled": 99}
}"""

proc parseHook(s: string, i: var int, v: var seq[Entry]) =
  var table: Table[string, Entry]
  parseHook(s, i, table)
  for k, entry in table.mpairs:
    entry.id = k
    v.add(entry)

let s = fromJson[seq[Entry]](data)
```

Gives us:
```
@[
  (id: "1", count: 12, filled: 11),
  (id: "2", count: 66, filled: 0),
  (id: "3", count: 99, filled: 99)
]"""
```

### `proc dumpHook()` Can be used to serializer into custom representation.

Just like reading custom data types you can also write faster data types with `dumpHook`. Using `Fraction` from the above example:

```nim
proc dumpHook(s: var string, v: Fraction) =
  ## Output fraction type as a string "x/y".
  s.add '"'
  s.add $v.numerator
  s.add '/'
  s.add $v.denominator
  s.add '"'

var f = Fraction(numerator: 10, denominator: 13)
let s = f.toJson()
```

Gives us:
```
"10/13"
```
