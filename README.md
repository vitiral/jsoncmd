json-cmd is a *specification* for how to make command line programs that are
bash compatible but can be used to make a new shell scripting language that is:
- easy to write, read and reason about
- uses strict inferred types to improve reliability
- easy to work with multiple Streams (and their corresponding processes)
  and combine their inputs
- no ambiguous syntax

The proposed scripting language can be found in [JSH.md](JSH.md) and the
specification for json-cmd compliant programs is at
[SPECIFICATION.md](SPECIFICATION.md).

## Intent
The command line is a nurturing place for software, giving freedom, simplicity
and modularity that few implementations can match. However, passing data between
command line programs is non-intuitive and prone to error especially when the
data becomes complex.

The current state of the art for command line communication is text-based
processing tools. In this domain, there is only one type: the stream of bytes.
If you've ever tried to work with bash processing tools like sed or awk you know
how powerful they can be and also how difficult and non-intuitive it can be to
use them for anything but the most simple tasks. These tools typically rely on
regular expressions or "columns" (spaces) to differentiate the different points
of data.

**json-cmd** was created because there should be a better way -- let's make
command line programs that adhere to a spec similar to json-rpc but use inferred
types to be less error prone.

Data should be easy to filter, combine and form into new data without worrying
that you made a minor mistake. JSON gives an excellent combination of simplicity
and value -- it is language agnostic, typed and human readable. Let's make a new
shell that uses it.
