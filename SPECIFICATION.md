## Overview
These specs allow for command line programs to have a specification similar to
jsonrpc while also providing type safety and other benefits that you can
only get on the command line (using pipes).

jsoncmd programs receive a single json blob as their options. They also receive
the input of streaming json through stdin. They output json to other programs
(or the user) through stdout. Application errors are also output through stdout,
with return-code values being reserved for critical errors.

A number of conventional unix programs (ls, cat, rm, etc) as well as jsoncmd
specific programs are defined as part of this specification for working with
jsoncmd programs similar to how one works with the standard unix shell.

# jsoncmd specification
A command line program is jsoncmd compliant if it meets the following
requirements.

## vocabulary
The following vocabulary is used
- unix: means the conventions used for unix, things like options
  being passed as `-a ARG`, etc.

## Type Specification
When the `--jsoncmd-types` flag is passed, the command must output the types
in the following format:

```
{
    "params": "PARAMS_TYPE",
    "input": "INPUT_TYPE",
    "output": "OUTPUT_TYPE",
    "types": "TYPES",
    "stream": {
       "input": BOOL,
       "output": BOOL
   }
}
```

Where `params`, `input` and `output` all take a single type argument (i.e.
`Array<Int>`) and `types` is used to specify more complex types.

Any or all of `params`, `input` and `output` can be excluded for programs
which do not have them.

Specifying your types might be something like (note: this uses `#` for comments
which is not valid json):
```
{
    # These specify types that can be used in addition to the builtin types.
    # Types can be built from eachother (i.e. you can define a type MyObject
    # that uses type MyValue).
    "types": [
        # specify the type we will use for PARAMS_TYPE. The convention is to use
        # "Params" to name this type
        {
            "name": "Params",
            "type": "Object",
            # Object layout is specified as a list of Objects containing attributes
            # attr, type and (optional) default
            "layout": [
                {
                    "attr": "path",
                    "type": "String"
                },
                {
                    "attr": "all",
                    "type": "bool",
                    "default": false  # must be a valid type for this attr
                }
            ]
        },

        # specify a type to use for indexes to our input Array
        # for an object type we specify type=Object
        # and specify the layout
        {
            "name": "Index",
            "type": "Object",
            "layout": [
                {
                    "attr": "name",
                    "type": "String",
                },
                {
                    "attr": "data",
                    "type": "Data",
                    "default": 0  # specify a default if one exists
                }
            ]
        },

        # Specify a data type for this application's output data
        {
            "name": "Data",
            "type": "Object",
            "layout": [
                {
                    "attr": "height",
                    "type": "Float",
                },
                {
                    "attr": "weight",
                    "type": "Float",
                }
            ]
        }
    ],

    # Now we can use the above types to specify our
    # inputs and outputs

    "params": "Params",       # use custom type defined in TYPES
    "input": "List<Index>",   # a List of custom type Index
    "output": "List<String>", # a List of builtin type String

    # see the "Stream Input and Output" section
    "stream": {
       "input": true,
       "output": true
   }
}
```

The following types are builtin and are very similar to json:
- `Bool` for boolean types (true and false)
- `Int` for integer types
- `Float` for floating point types
- `String` for a utf8 valid String
- `Bytes` for a utf8 string encoded in Base64 that should be converted to raw
  `Array[u8]`
- `Null` for the null type
- `Optional<TYPE>` for types that may be null
- `Enum<TYPE1, TYPE2, ...>` if multiple types are valid.
    - i.e. if your application can accept String or Float, use
      `Enum<String, Float>`
- `Object` with a corresponding `layout` specifying it's attributes (see above)
- `Array<TYPE>` for array types
    - Can be chained with Optional or Enum to allow multiple types in the List:
        `List<Enum<String, Number>>`

## Arguments
Specifications for jsoncmd arguments:
- **must** accept a `--jsoncmd JSON` argument, where JSON is a valid json
  string.
- **must** accept a `--jsoncmd-types` flag, which returns the types specified
  in the "Type Specification" section.
- **must not** accept any other arguments when `--jsoncmd` or
  `--jsoncmd-types`is specified.
    - additional arguments **must** return rc=101 immediately.
- **should** accept all "long" unix arguments as key/value pairs in the `JSON`
  blob to be as unix compatible as possible
  - i.e. `ls -al` should be able to be called with:
    `ls --jsoncmd '{"all": true, "long": true}'`

## Stream Input and Output
If you set `stream["input"]` or `stream["output"]` equal to true, your program
must adhere to the following specifications.

Programs that can "stream" are the primary mechanism by which jsoncmd
achieves high performance when chaining multiple commands together.
This only works, however, if there is some mechanism by which programs
can prevent data from buffering out of control. It is recommended that most
programs which *can* stream, *should* stream.

The simple rules are of being stream compliant are:
- Programs that are taking data should only accept data when they are ready to
  process it (don't accept 100MB when you process 1MB at a time).
- Programs that are outputting data should block until their stdout buffer has
  been flushed.

By having programs with "streaming output" block until the program which
accepts their input is ready, data can be processed "just in time" rather
than buffered.

The only types that are allowed to have `stream==true` are:
- `Bytes`: a stream of raw bytes
- `String`: a stream of utf8 encoded bytes
- `Array`: a stream of array indexes

## stdout
Specifications for jsoncmd program's stdout
- **must** output two null character before json. It is recommended not to
  output anything before the two null characters (in a future specification that
  data may be used for metadata by shell programs).
- **must** be valid json after the two null characters
- **may** return an error by outputting two null bytes followed by text which
  **must** adhere to "Error Output" specification below.
- **must** end output with three null bytes (for both result and error types)

### Error Output
Error outputs **must** be valid json that are returned after two null bytes. It
**must** adhere to the jsonrpc specification's "Error object" specification by
having only the following attributes:

- code: A Number that indicates the error type that occurred. This MUST be an
  integer.
- message: A String providing a short description of the error. The message
  SHOULD be limited to a concise single sentence.
- data: A Primitive or Structured value that contains additional information
  about the error. This may be omitted. The value of this member is defined by
  the application (e.g. detailed error information, nested errors etc.).
- caused: An Array that contains the previous error objects. Used for chaining
  errors. Note that this is jsoncmd specific.

Error codes **should** adhere to the jsonrpc specification, which is located
[here][1] (substitute "Server" for "application").

[1]: http://www.jsonrpc.org/specification

## stdin
Specifications for jsoncmd program's stdin:
- **must** ignore the input until two null characters are received
- after the two null characters, **must** accept only valid json until at least
  one null character is received
    - invalid json in the input **must** return rc=102 immediately
- **must** accept three consecutive null characters to be the end of the output.
- **must** interpret two additional consecutive null characters followed by
  valid json then three null characters to be a returned error. The program
  **must** either:
  - handle the error and return a non-error
  - return the error as it was found
  - append the old error to `caused` and return a new error

# Implementation Design Suggestions
The following are suggestions for libraries which enable jsoncmd compliance.

## stdin Handlers
Libraries **should** support at least three kinds of stdin input handlers:
- `ByteStream`: handler which ONLY accepts a String input and converts it from
  Base64 to raw u8 bytes. This is useful for handling raw binary data like
  files.
- `StringStream`: handler which ONLY accepts a String input and streams it one
  utf8 character at a time.
- `ValueStream`: handler which ONLY accepts an Array input and emits each
  buffered Value when it completes. This should validate that each value is
  valid json as it is processed.
- `Value`: handler which buffers any json input and outputs it as a single
  validated Value.

These are the four main output types of jsoncmd programs and cover the vast
majority of high-performance use case needs.

