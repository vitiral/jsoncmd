# jsh: a shell scripting language which utilizes the json-cmd specification

jsh is an *experimental* scripting language in it's design phase who's purpose
is to use the json-cmd specification to create a language that is:
- easy to write, read and reason about
- use strict types to improve reliability
- easy to work with multiple Streams (and their corresponding processes)
  and combine their inputs
- no ambiguous syntax

json-cmd guarantees that all input and output is in typed json-format, allowing for
filtering and restructuring of data using list comprehensions (similar syntax to
python). The way that jsh does this is it converts it's syntax into a call of
the command with the `--json-rpc JSON` flag.

The syntax for jsh is NOT bash compliant or even bash like. Instead it it is
inspired by elm and python with the goal of being able to construct expressive
one line statements that are unambiguous and easy to read.

It's primary goals are to be:

# Types
jsh is a strongly typed language. There are two main types:
- Value: a resolved JSON value
- Stream: A Stream of bytes that will resolve to valid JSON and is gotten
  through the `&` operator.

Types are gotten through calling the cmd with `--json-cmd-types`

Note that there is an `Any` type which disables type checking.

## Examples

- reserved keywords:
    - `in`: used in let, for, match, and Fn declarations
    - `for`: used in Array and Object comprehensions
    - `let`: used in declaring local variables
    - `match`: used in pattern matching
    - `type`: used in declaring types
    - `def`: used in declaring functions
- reserved variables:
    - `cwd`: the current working directory
    - `stdin`: streaming type inside functions for the stdin
- defined types:
    - `Bool`: a boolean value
    - `Flag`: a boolean value with default value of false, used for declaring
      flags in Fn declarations
    - `Int`: an integer, can be operated with Float
    - `Float`: equivalent to json Number
    - `String`: equivalent to json String
    - `Bytes`: json String encoded in Base64
    - `Array<TYPE>`: a stream or list of values
    - `Null`: empty type
    - `Object`: a key/value pair object where each key has a type. Declared with
        `{key: TYPE, ...}` syntax
    - `Enum<TYPES>`: a type composed of multiple types
    - `Option<Value>`: shorthand for `Enum<Value, Null>`
- operators: `+ - * ^ / ! && || // &`
    - `+ - * ^ / && || % !` are the same as they are in C
        - `+` works on Int+Float, String+String, Array+Array in the expected way
        - `! && ||` can only be used on boolean values
    - `//` is the same as python: division that always results in an Int
    - `&` is the stream operator and toggles whether the value is a Stream or a
      buffered Value
- default functions:
    - `ls`: see unix, outputs `List<String>` by default
    - `rm`: see unix
    - `cat`: see unix
    - ... other standard linux tools modified for jsh
    - `range [min/max, max, step]`: same api as python's range function
    - `flatten [array1, array2, ...]`: flattens an array of arrays into a
        single depth array. The type of this is an Enum of all the types of the
        inputs.
    - Object accessors:
        - `get ["key1", "key2"] Object`: gets keys from an object and returns an
          Array of their values. Returns `null` for each key that does not exist
        - `keys {} Object`: accepts an object and returns a List of it's keys
        - `values {} Object`: accepts an object and returns a Lis of it's values
    - regexp matching:
        - `reg-compile String`: compiles a regular expression into it's Bytes representation
          for faster lookups
        - `reg-match Enum<String, Bytes> String`: returns true if the String
          matches the pattern
        - ... other functions
    -
- defining types: `type MyType = { foo: Array<String>, bar: String }`
- calling a Fn: `grep "pattern" STREAM`
- assigning to a Value:`v = cmd {}`
- passing a Value: `cmd v`
- assigning to a Stream: `s = &cmd {}`
    - `s` is now a spawned process who's output is being buffered
- passing a Stream to a Fn: `cmd {} s`
- converting a Value to a Stream:
    - `s = &[1, 2, 3]`
    - `s = &v` where `v` is some Value
- passing a Stream as a Value (buffering it): `cmd &s`
    - note: idea is that `&` toggles "streamness"
- constructing a Value from Values, Streams and Fns:
    `v = {"foo": "bar", "value": v, "result": (cmd {})}`
    - note: any valid expression can be wrapped in parenthesis. All expressions
      return a Value
- list comprehension: `v = [n*10 for n in value]`
    - value can be *either* a `List` or `Stream<List<_>>` type (not String/Bytes)
- local variables: `v = let x=4, y=7 in [n*x+y for n in (range 10)]`
- line comment: `x = 4 -- this is a comment`
- block comment: `x = {- this is a comment -} 4`
- pattern matching of types where `n` is type `Enum<Int, String>`:
    `v = match n in Int (n) | String (int n) -- v becomes an Int no matter what`
- declaring a Fn:
```
def three = { 0 | arg0: String } List<Int> List<String> -- always three types
( [(str n) for n in stdin] )
```
- declaring a more complex Fn:
```
def f = (
    -- the types of the arguments
    { 0 | arg0: String  -- first positional argument or kwarg=arg0
    , 1 | arg1: Int     -- second positional argument or kwarg=arg1
    , f | flag1: Flag   -- flag=f or flag=flag1 or kwarg=f or kwarg=flag1
    , g | flag2: Flag   -- flag=g or flag=flag2 or kwarg=g or kwarg=flag2
    , kwarg1: Custom  -- kwarg=kwarg1 which is a custom type
    , kwarg2: Float   -- kwarg=kwarg2 which is a float
    }

    -- the type of the stream input
    List<Int>

    -- the type of the output
    List<String>)

    -- the declaration
(
    let
        -- argument types are available as local variables
        x = 4 + arg1 if flag1 else 0, -- do some complex crap
        y = 7 * kwarg2
    in
        [(str n*x+y) for n in stdin]
)
```
- accessing a function from a module: `mod::f`
- accessing a type from a module: `mod::Custom`
- accessing a type used in a function: `f::Custom`
- calling a bash function in compatibility mode
```
`ls -al /var/log` -- always returns Bytes
```

### Bash Compatibility:
- converting and visualizing columns (it uses the type to figure out what you
  want):
    - parsing columns as a `List<List<Enum<String, Float>>>`:
        `table {} BYTES`
    - formatting columns (with color):
        `table {} [["header", "row"], ["first", "row"]]`
    - doing the whole operation in one line:
        `table {} (table {} BYTES)`

