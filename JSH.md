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

## Important Guarantees and Concepts

jsh is intended to make working with multiple different running programs less
error prone. It does this through it's type system and it's compile-time
guarantees around live instances of the Fn and Ps types.

Function are typed like `Fn<Params, Ps<I>, Ps<O>>` where the three types are
the Params (the arguments passed to the function), the input Ps (process) "stream"
to the function (which includes stdin) and the stdout type of the function.

Let's start with a simple case
```
-- one line example
s = &echo {} (&cat "in.txt")
... do stuff with s

-- same as multi line example
e = &echo {}
c = &cat "in.txt"
e << c
s = e << _
... do stuff with s
```
- the type of `s` in both cases is now `Ps<String>`. Let's review the
  multiline example first for an explanation:
  - the type of echo is `Fn<{...}, String, String>`
  - the type of cat is `<Fn<{0 | path: String}, _, String>`, in other words
    it can not accept stdin
  - `e = echo {}`: the type of `e` is `OpenPs<String, String>`
    (it can still accept data in it's stdin and is running in the background)
  - `c = &cat "in.txt"`: the type of `c` is `Ps<String>`. Note that we didn't
    need to send any stdin because `Input=_`
  - `e << c`: this has caused several changes
    - `c` has been passed into `e` and it's type has become `_` (`c` has been
      consumed and can no longer be used)
    - the type of `e` remains `OpenPs<String, String>`, it could still
      receive more stream inputs.
  - `s = e << _`: this ends the stdin for `s`. The type of `s` is now
    `Ps<String>` as it is no longer "open"
- for the single line case, you are passing in the `Ps` for `s` to use when
  you instantiate it, so the `<< _` is implied (it is implied that you want
  a `Ps` not a dangling `Fn`.

Important guarantees that jsh provides:
- it is illegal in jsh scripts or functions to have unconsumed `Ps` or `OpenPs`
  instances -- in other words you cannot accidentally leave behind unexecuted
  processes
- there are two exceptions to this:
    - on the command line you can have open processes
    - functions can *return* a `Ps` (but not an `OpenPs`)

Chaining functions/processes:
```
g1 = &grep "foo"        -- filter for lines containing "foo"
g2 = &grep "bar" g1     -- filter or lines continaing BOTH "bar" and "foo"
                        -- (they have been chained)
c = &cat "in.txt"       -- create a stream of text
g2 << &cat "in1.txt"    -- input some text
g2 << &"\n"             -- input an extra newline just in case
g2 << &cat "in2.txt"    -- input some more text
s = e2 << _             -- close our stream and get the stdout Ps
write "out.txt" s       -- write our result to a file
```
The above code "chains" the two `grep` commands together and then inputs BOTH
"in1.txt" and "int2.txt" into that stream. The output of g1 will be passed into
g2, which will then be written to "out.txt". If you are familiar with grep, you
can probably guess what this will do: only lines that contain both "foo" and
"bar" will be in "out.txt".

## Reserved Keywords
- `in`: used in let, for, match, and Fn declarations
- `for`: used in Array and Object comprehensions
- `let`: used in declaring local variables
- `match`: used in pattern matching
- `type`: used in declaring types
- `def`: used in declaring functions
- `use`: importing a module

## Default Types
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
- `Ps<Output>`: the process type, which encompasses the typed stdout of a
  real or fake process.
- `OpenPs<Input, Output>`: a process who's stdin is still "open" -- i.e. it can
  receive values using the `<<` operator.
- `Fn<Params, Ps<I>, Ps<O>>`: function type. This type is special in general
  as it's definition does not execute the expression given and instead can
  later be executed. Some special syntax:
  - `Params` type accepts special syntax and simply injects the values into the
    defined expression. The syntax is intented to make it easy to define
    functions that are "unix like" and easy to call while still being intuitive.
  - The `Ps` type does not need to be given for the generic types `I` and `O`.
    You can declare a function type like:
    `type MyFn = Fn<{...}, String, String>`
- `_`: empty type, cannot occupy a value and discards if assigned to.
    - example: `type Fn<Params, _, _>` defines a function that does NOT accept
      a Ps stream and does NOT output a stream

## Reserved Variables
- `cwd`: the current working directory
- `stdin`: Ps type for the stdin (inside function definitions only)
- `eoi`: "end of input", used for telling a process with an open input that
  the input is done

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
- converting a Value to a "fake" Ps (has no pid, just a stream):
    - `s = &[1, 2, 3]`
    - `s = &v` where `v` is some Value
- passing a Ps as a Value (blocking until done and buffering it): `cmd &s`
    - note: idea is that `&` toggles "streamness"
- constructing a Value from Values, Ps and Fns:
    `v = {"foo": "bar", "ps": &s, "value": v, "result": (cmd {})}`
    - note: any valid expression can be wrapped in parenthesis. All expressions
      return a Value (even if that value is null)
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
my_func: Fn<
        { 0 | arg0: String  -- first positional argument or kwarg=arg0
        , 1 | arg1: Int     -- second positional argument or kwarg=arg1
        , f | flag1: Flag   -- flag=f or flag=flag1 or kwarg=f or kwarg=flag1
        , g | flag2: Flag   -- flag=g or flag=flag2 or kwarg=g or kwarg=flag2
        , kwarg1: Custom  -- kwarg=kwarg1 which is some custom type
        , kwarg2: Float   -- kwarg=kwarg2 which is a float
        },
        -- the type of the stream input
        List<Int>,
        -- the type of the output
        List<String>)> = (
    -- the declaration
    let
        -- note that argument types are injected as local variables
        x = 4 + arg1 if flag1 else 0, -- do some complex crap
        y = 7 * kwarg2
    in
        [(str n * x + y) for n in stdin] -- `stdin` is also injected
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

## Chained performance guarantees
> This section is experimental musings

Let's say you have a function F1, which is outputing to another function F2.
Let's also say that F1 outputs a lot of data very very quickly but F2 has to
take time to process each line. An example might be if F1 is `cat` and F2 is
sending each line to a webserver.

In such a case, you do not want to have to buffer all of F1's data, as that
could potentially be enourmous (gigabytes or even terrabytes). How can we avoid
this? Is there some kind of "signal" that can be sent telling F1 to wait?

Hopefully there is some way to tell your stdin stream that you are not ready for
more data. Maybe tell it your buffer is "full" or something to that effect.

# Use Case: creating an init system

One of the primary use cases for JSH *could* be in creating an init system. To
do that, we would need at least the following features:
- Ps and Fn types can be included in native types and passed to native functions
    - `x = {"echo": (&echo "some/path"), "fn": my_func}`
    - `out = &x.fn {} x.echo` (in other words, we are passing processes and
      functions around)
- methods attached to Array and Object types for mutating them

Once these two things happen, it should be possible to represent all init
commands as simple data, from which an init function (written in jsh) could
execute them according to their dependencies/metadata.

An example of the init types might be
```
--! init module types
-- declare the ServiceStart type that all services must use
type ServiceStart<P> = Fn<
        P,      -- generic params
        Int,    -- kill flag streaming input
                -- TODO: may want to use an Object that has more data
        Null>   -- no output

-- declare the service type for actually defining the service and
-- it's configuration. Note that the Params `P` are generic
type Service<P> = {
    start: ServiceStart<P>,        -- a specific function implementation
    requires: Array<Service<_>>,   -- accepts objects whos types cannot be used
                                   -- (they can only be compared/hashed)
    params: P,
}
```

An example init module might be defined at "/dev/init/examples/example1.jsh"
```
--! example init module (i.e. for starting a networking service)

mod init = "/dev/init/init.jsh"

type Cmd = { name: String }
example_start: Fn<Cmd, Int, Null> = (
    -- do stuff in the function
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

-- something like this could replace the fstab file
fs_service = {
    start: kernel::fs_start,
    requires: [kernel::boot_service],   -- maybe not necessary?
    params: {
        drives: [
            {
                device: "/dev/sda5",    -- use this device path
                mount: "/",             -- mount as root
                uuid: "d33df79f-e51e-4a17-acb1-b8f87c01d0d",
            },
        ]
    },
}

example_service = {
    start: example::example_start,
    requires: [fs_service],
    params: { name: "example-name" },
}

initd::services.extend [
    fs_service,
    example_service,
]
```

One of the most important aspects of this approach is that everything is *just
data* (nothing is executed until initd does so). This means that you can load
the init configuration and make assertions. For instance, it should be invalid
to have a servie that depends on a service that is not in `initd::services`.
