json-cmd is a *specification* for how to make command line programs who's input
and output can be easily filtered and passed to other programs with less
of the manual text processing that has to be done with conventional bash
programs.

json-cmd is created with the intent to create a superior shell scripting
language that has more useful types than just the string type and can draw from
the wealth of features that json-rpc has given to the web. For details on that,
see [JSH.md](JSH.md).

The full specification can be found at [SPECIFICATION.md](SPECIFICATION.md).
The proposed shell scripting language which uses json-cmd can be found at

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
command line programs that adhere to a spec similar to json-rpc. This does *not*
mean that they *are* json-rpc -- no, it is json-rpc for the command line.

Data should be easy to filter, combine and form into new data. JSON gives an
excellent combination of simplicity and value -- it is language agnostic, typed
and human readable. Let's make a new shell that uses it -- while still allowing
the programs that are written for the new shell to be 100% bash compatible.
