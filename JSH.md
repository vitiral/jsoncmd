# jsh: a shell scripting language which utilizes the json-cmd specification

jsh is an *experimental* scripting language in it's design phase who's purpose
is to use the json-cmd specification to create a language that is:
- easy to write, read and reason about
- use strict types to improve reliability
- easy to work with multiple processes (Ps) and combine their stdouts
- no ambiguous syntax

json-cmd guarantees that all input and output is in typed json-format, allowing for
filtering and restructuring of data using list comprehensions (similar syntax to
python). The way that jsh does this is it converts it's syntax into a call of
the command with the `--json-rpc JSON` flag.

The syntax for jsh is NOT bash compliant or even bash like. Instead it it is
inspired by elm and python with the goal of being able to construct expressive
one line statements that are unambiguous and easy to read.

> Note:
>
> Types are gotten through calling the cmd with `--json-cmd-types`
>
> It will also support generic types for functions in the future

## Modules and Scripts
jsh modules are files which have the extention `.jsh` and contain expressions
that are only type, function and variable declarations. The method of importing
modules is in the Example Syntax

jsh scripts are executable files which do not have any particular extension, but
typically begin with:
```
#!/usr/bin/jsh
```

From there they have the same syntax as if they were being run in the REPL
terminal. A newline (in non-closing parenthesis) will cause a statement to be
"executed".

jsh scripts can be "imported" using the `source` keyword, which can only be used
in scripts (not in modules).
```
x = source "path/to/script"

-- source now contains the entire namespace of script
script_var = x::var

-- dump script's namespace into our own
* = x::(*)
```

jsh scripts are still type checked before being executed (along with their
underlying modules).

## Reserved Keywords
- `in`: used in let, for, match, and Fn declarations
- `for`: used in Array and Object comprehensions
- `let`: used in declaring local variables
- `match`: used in pattern matching
- `type`: used in declaring types
- `def`: used in declaring functions
- `use`: importing a module

## Defined Types
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
- `Ps<Type>`: the process type, which encompasses the typed stdout of a
  real or fake process.
- `Fn<Params, Ps, Output>`: function type. This type is special in general
  as it's definition does not execute the expression given and instead can
  later be executed. In addition, the `Params` type accepts special
  syntax and simply injects the values into the defined expression.

## Reserved Variables
- `cwd`: the current working directory
- `stdin`: Ps type for the stdin (inside function definitions only)

## Operators
`+ - * ^ / ! && || // &`
- `+ - * ^ / && || % !` are the same as they are in C
    - `+` works on Int+Float, Bytes, String+String, Array+Array in the expected way
    - `+` also works on Ps of Bytes, String and Array by chaining them
    - `! && ||` can only be used on Bool values
- `//` is the same as python: division that always results in an Int
- `&` is the stream operator and toggles whether the value is a Ps or a
  buffered Value

## Default Functions
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
    - `values {} Object`: accepts an object and returns a List of it's values
- regexp matching:
    - `reg-compile String`: compiles a regular expression into it's Bytes representation
      for faster lookups
    - `reg-match Enum<String, Bytes> String`: returns true if the String
      matches the pattern
    - ... other regexp functions

## Example Syntax
- calling a Fn: `grep "pattern" STREAM`
    - see [json-cmd spec](SPECIFICATION.md) for how the first argument is inferred
- assigning to a Value: `v = cmd {}`
- passing a Value to a command: `cmd v`
- assigning to a Ps from a command: `s = &cmd {}`
    - `s` is now a spawned process who's output is being buffered
- passing a Ps to a Fn: `cmd {} s`
- converting a Value to a Ps:
    - `s = &[1, 2, 3]`
    - `s = &v` where `v` is some Value
- passing a Ps as a Value (buffering it): `cmd &s`
    - note: idea is that `&` toggles "streamness"
- constructing a Value from Values, Ps and Fns:
    `v = {"foo": "bar", "value": v, "result": (cmd {})}`
    - note: any valid expression can be wrapped in parenthesis. All expressions
      return a Value or a Ps
- list comprehension: `v = [n*10 for n in value]`
    - value can be *either* a `List` or `Ps<List<_>>` type (not String/Bytes)
- local variables: `v = let x=4, y=7 in [n*x+y for n in (range 10)]`
- line comment: `x = 4 -- this is a line comment`
- block comment: `x = {- this is a block comment -} 4`
- pattern matching of types where `n` is type `Enum<Int, String>`:
    `v = match n in Int (n) | String (int n)`
- defining types: `type MyType = { foo: Array<String>, bar: String }`
- defining generics: `type MyType<G> = {foo: Array<G>}`
- using generics: `x: MyTpe<String> = {foo: ["hello"]}`
    - note: the generic type can often be inferred
- declaring a Fn:
```
three: Fn<{ 0 | arg0: String }, -- "|" is special syntax for positional arg
          List<Int>,
          List<String>> = (
    [(str n) for n in stdin] )
```
- declaring a more complex Fn:
```
def f: Fn<
        -- the argument types, uses some special syntax for only this
        { 0 | arg0: String  -- first positional argument or kwarg=arg0
        , 1 | arg1: Int     -- second positional argument or kwarg=arg1
        , f | flag1: Flag   -- flag=f or flag=flag1 or kwarg=f or kwarg=flag1
        , g | flag2: Flag   -- flag=g or flag=flag2 or kwarg=g or kwarg=flag2
        , kwarg1: Custom  -- kwarg=kwarg1 which is a custom type
        , kwarg2: Float   -- kwarg=kwarg2 which is a float
        },
        -- the type of the stream input
        List<Int>,
        -- the type of the output
        List<String>)> = (
    -- the declaration
    let
        -- note that argument types are available as local variables
        x = 4 + arg1 if flag1 else 0, -- do some complex crap
        y = 7 * kwarg2
    in
        [(str n * x + y) for n in stdin]
)
```
- importing a module: `use mod = "path/to/mod.jsh"
- unpacking some functions: ` * = mod::[f1, f2]`
- unpacking and renaming a function: `g = mod::f`
- unpacking all functions: ` * = mod::[ * ]`
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
        ```
        table {} `bash-cmd`
        ```
    - formatting columns (with color):
        `table {} [["header", "row"], ["first", "row"]]`
    - doing the whole operation in one line:
        ```
        table {} (table {} `bash-cmd`)
        ```

# Use Case: creating an init system

One of the primary use cases for JSH *could* be in creating an init system. To
do that, we would need at least the following features:
- Ps and Fn types can be included in native types and passed to native functions
    - `x = {"echo": (&echo "some/path"), "fn": my_func}`
    - `out = &x["fn"] x` (in other words, we are passing processes and functions around)
- methods attached to Array and Object types for mutating them
- flush out how generic types work
- The script and module system will probably have to be flushed out more

Once these two things happen, it should be possible to represent all init
commands as simple data, from which an init function (written in jsh) could
execute them according to their dependencies/metadata.

An example of the init types might be
```
--! init module types
-- declare the ServiceStart function
type ServiceStart<Params> = Fn<
        Params,
        Int, -- kill flag streaming input
             -- TODO: may want to use an Object that has more data
        Null> -- no output

type Service<Params> = {
    start: ServiceStart<Params>,
    requires: Array<Service>,
    params: Params,
}
```

An example init module might be defined at "/dev/init/examples/example1.jsh"
```
--! example init module

mod init = "/dev/init/init.jsh"

type Params = { name: String }
start_ex: Fn<Params, Int, Null> = (
    -- do stuff...
)
```

Finally, in the full system configuration you would have
something like

```
#!/usr/bin/jsh

-- os's init configuration
use kernel = "/dev/init/kernel.jsh"
use example = "/dev/init/examples/example1.jsh"

source initd = "/dev/init/initd"

initd::services.extend [
    {
        start: kernel::necessary_service_start,
        requires: [],
        params: {},
    },
    {
        start: example::start_ex,
        requires: [kernel::necessary_service_start],
        params: { name: "example-name" }
    },
    -- ... other services
]
```
