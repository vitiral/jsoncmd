# jsh: a shell scripting language which utilizes the json-cmd specification

jsh is an *experimental* scripting language in it's design phase who's purpose
is to use the json-cmd specification to make the glue code of conventional
scripts easier to reason about.

json-cmd guarantees that all input and output is in json-format, allowing for
filtering and restructuring of data using list comprehensions (similar syntax to
python). The way that jsh does this is it converts it's syntax into a call of
the command with the `--json-rpc JSON` flag.

The syntax for jsh is NOT bash compliant or even bash like. Instead it it is
inspired by elm and python with the goal of being able to construct expressive
one line statements that are unambiguous and easy to read.

It's primary goals are to be:
- easy to write, read and reason about
- easy to deal with multiple Streams (and their corresponding processes).
- no ambiguous syntax

# Types
jsh is a strongly typed language. There are two main types:
- Value: a resolved JSON value
- Stream: A Stream of bytes that will resolve to valid JSON and is gotten
  through the `&` operator.

Types are gotten through calling the cmd with `--json-cmd-types`

Note that there is an `Any` type which disables type checking.

# Syntax Overview
Every expression returns a value. Expressions can always wrapped in `(...)`,
An expression can be:
- a variable or reference to a variable (i.e. `variable[1]`)
- an assignment `x = ...` where `...` is another expression that may omit it's
  parenthesis.
- a json expression that may itself contain valid expressions wrapped in
  `()` to execute. This can be as simple as `"value"` or `123` to as complex as
  `{"foo": [1, 2, 3], "bar": (cmd ["a"])}`
- an operation, which is expression/s connected with operators
    `+ - * / < > % ^ &`
    - `&` is the stream operator and is used as a *toggle* for streams

We will use the json-cmd versions of the commands `ls` and `rm` for these
examples. In addition, the command `flat` is used, which takes an array
of arrays and makes them a single depth array.

- call: `ls {"l": true}`
- pass a result from one function to another:
    `rm {"paths": (ls ".*\.bak"), "f": true}`
- addition:
    - numbers: `1.0 + 3 == 4`
    - strings: `"foo" + "bar" == "foobar"`
    - lists: ["foo"] + ["bar"] == ["foo", "bar"]
- list comprehension:
    - `[ rm (p + ".bak") for p in ["foo", "bar"] ]`
    - `[ value for key, value in {"foo": "bar", "bar": "foo" } ]`
- assigning values: `value = <expression>`
- creating Streams from spawned processes
    - get Stream from a command `out = (&cmd {})`
        - note: `out = &(cmd {})` will buffer the cmd into a Value and then turn
          it into a Stream... probably not what you want (may raise an error).
    - create a Stream directly from a value: `out = &[1, 2, 3]`
        - or from a generator: `out = &[n for n in (range [100])]`
        (this can actually provide minor performance gains)
    - pass in a piped value to a new cmd: `(cmd {} out)`
        - the second argument to a cmd MUST be a pipe.
    - converting a Stream to a Value (`&` toggles it): `y = &out
        - returns rc=_ if the value is an error
    - buffering a pipe's error into a Value: `y = (collect {} out)["error"]
    - combine multiple Array Streams into a single Stream: `(&flat {} &[&out1, &out2])
      - note: this treats list and String addition types as special and streams
        "as values are ready" for performance gains. Other types are buffered.
    - get the stderr Stream as well with "&!": `out, err = (&!cmd {})`
- if statement: two ways to do the same thing
    - `result = rm (if value ("foo.txt") else ("bar.txt"))
    - `result = if value (rm "foo.txt") else (rm "bar.txt")`
        - this assigns the output of the command to `result`
- local variables:
    - `result = let x=4, y=7 in (x + y)`
    - `result = let x, y = [4, 7] in (x + y)`
    - `let ext=".bak" in [ rm (p + ext) for p in ["foo", "bar"] ]`
    - `[ (let x = value[key] in x + ".bak") for (key, value) in record ]`
