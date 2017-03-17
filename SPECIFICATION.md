## Overview
These specs allow for a json-cmd program to simultaniously support
json-cmd shells as well as conventional bash shells, through the reserved
`--json-cmd` argument.

json-cmd programs receive a single json blob as their options. They also receive
the input of streaming json through stdin. They output json to other programs
(or the user) through stdout. Application errors are also output through stdout, with
return-code values being reserved for critical errors.

stderr uses the jlog format to be easy to parse and view by json-cmd shells.

A number of conventional unix programs (ls, cat, rm, etc) as well as json-cmd
specific programs are defined as part of this specification for working with
json-cmd programs similar to how one works with the standard unix shell.

# json-cmd specification
A command line program is json-cmd compliant if it meets the following
requirements.

## vocabulary
The following vocabulary is used
- unix: means the conventions used for unix, things like options
  being passed as `-a ARG`, etc.

## Arguments
Specifications for json-cmd program's arguments:
- **must** be runnable in a bash compliant shell
- **must** accept a `--json-cmd JSON` argument, where JSON is a valid json
  string.
- **must** accept a `--json-cmd-types` flag, which returns the input and
  output types in the format specified in Type Format
- **must not** accept any other arguments when `--json-cmd` is specified.
    - additional arguments **must** return rc=101 immediately.
- **must** accept the following keys to `--json-cmd`, which are reserved:
    - args: an Array of Values corresponding to the unix "positional arguments"
    - flags: an Array of Strings corresponding to the unix `-f --flag` like syntax
- positional arguments **must**:
    - be accepted as a positional argument: `{"args": ["value"]}`
    - be accepted as a key: `{"arg": "value"}` or shortened `{"a": "value"}`
    - All of these inputs **must** be converted to the full form of `{"arg": "value"}`
    - attempting to specify multiple **must** cause an error with rc=100
- flags **must**:
    - be accepted as a key: `{"flag": true}` or `{"f": true}`
    - be accepted within the `flags` value: `{"flags": ["f"]}`
    - All of these inputs **must** be converted to the full form: `{"flag": true}`
    - attempting to specify multiple **must** cause an error with rc=100
- **must** convert an input of a String `"string"` to `{"args": ["string"]}`,
  which will be converted to the full form.
- **must** convert an input of an Array `["arg0", "arg1"]` to `{"args": ["arg0", "arg1"]}`,
  which will be converted to the full form.
- **should** accept all unix arguments as key/value pairs in the `JSON` blob.
  - i.e. `ls -al` can instead be called with:
  `ls --json-cmd '{"a": true, "l": true}'`

## stdout
Specifications for json-cmd program's stdout
- **must** output two null character before json. It is recommended not to
  output anything before the two null characters (in a future specification that
  data may be used for metadata by shell programs).
- **must** be valid json after the two null characters
- **may** return an error by outputting two null bytes followed by text which
  **must** adhere to "Error Output" specification below.
- **must** end output with three null bytes (for both result and error types)

### Error Output
Error outputs **must** be valid json that are returned after two null bytes. It
**must** adhere to the json-rpc specification's "Error object" specification by
having only the following attributes:

- code: A Number that indicates the error type that occurred. This MUST be an
  integer.
- message: A String providing a short description of the error. The message
  SHOULD be limited to a concise single sentence.
- data: A Primitive or Structured value that contains additional information
  about the error. This may be omitted. The value of this member is defined by
  the application (e.g. detailed error information, nested errors etc.).

An Additional required key is:
- caused: An Array that contains the previous error objects. Used for chaining
  errors.

Error codes **should** adhere to the json-rpc specification, which is located
[here][1] (substitute "Server" for "application").

[1]: http://www.jsonrpc.org/specification

## stdin
Specifications for json-cmd program's stdin:
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

## stderr
json-cmd programs which output on stderr **must** adhere to the json-cmd logging
specification (also called "jlog") defined below so that error logs, like
output, can be parsed and processed by json-cmd programs. Other logs generated
by json-cmd programs **should** adhere to the following spec as well to better
fit within the ecosystem.

stderr **must** be an array of records with **at least** the following keys:
- **name**: a String which denotes the name of the logger
- **msg**: a String with a short description of the failure
- **epoch**: a Float which denotes the number of seconds since the [epoch][20]
  that the log message occured.. This is used instead of UTC time stamp because
  it requires significantly less overhead and can easily be converted by log
  viewing programs.
- **file**: an OPTIONAL String which denotes the path to the file which
  caused the log message.
- **line**: an OPTIONAL Int which denotes the line number which caused the log
  message.
- **data**: OPTIONAL arbitrary additional Value related to the error

Additional application specific keys **may** be used as well.

[20]: https://en.wikipedia.org/wiki/Unix_time

# Implementation Design Suggestions
The following are suggestions for libraries which enable json-cmd compliance.

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

These are the four main output types of json-cmd programs and cover the vast
majority of high-performance use case needs.

## Type Specification
When the `--json-cmd-types` flag is passed, the command must output the type
in the following format:

```
{
    "input": "INPUT_TYPE",
    "output": "OUTPUT_TYPE"
}
```

where the `*_TYPE` values are of the form:
- `String` for a utf8 valid String
- `Bytes` for a utf8 string encoded in Base64 that should be converted to bytes.
- `Int` for integer types
- `Float` for floating point types
- `Null` for null types
- `Optional<TYPE>` for types that may be null (or not exist in a record).
- `Enum<TYPE1, TYPE2, ...>` if multiple types are valid.
- `{ "key1": TYPE, "key2": TYPE}` for records
- `List<TYPE>` for list types.
    - Can be chained with Optional or Enum to allow multiple types in the List:
        `List<Enum<String, Number>>`
- `Any` to completely disable type checking (not recommended).
- `Log`: shortcut for the json-cmd log type:
  `List<{"name": "String", "msg": "String", ...}>`
