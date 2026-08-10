# cmdline-plus

A simple command line parser for C++.

*[Русская версия](README.ru.md)*


This is a fork of [tanakh/cmdline](https://github.com/tanakh/cmdline) by Hideyuki Tanaka, which
is no longer maintained. This fork includes a number of improvements compared to the original project.

The text is modified and translated from japanese article from the [original website](https://tanakh.hatenablog.com/entries/2009/10/28).

## About

Cmdline - is a library that helps you parse command line arguments. Other libraries serve the same purpose -
`getopt` from the C standard library, Google's `gflags` - but cmdline aims to be casually usable
and reasonably convenient. `getopt` is awkward to work with, and you have to write the usage
message yourself. `gflags` is a heavier: you install a library and link against it.
cmdline is a single header file - copy it in and you are done.

## Installation

Copy [`cmdline.h`](cmdline.h) next to your sources (or point `-I` at this directory) and include
it. There are no dependencies beyond the standard library and nothing to link.

```cpp
#include "cmdline.h"
```

To build the bundled examples:

```sh
g++ -std=c++11 -Wall -o test  test.cpp
g++ -std=c++11 -Wall -o test2 test2.cpp
```

C++98 and later are supported.

## Usage

```cpp
// include cmdline.h
#include "cmdline.h"

int main(int argc, char *argv[])
{
  // create a parser
  cmdline::parser a;

  // add a variable of the specified type.
  // 1st argument is long name
  // 2nd argument is short name (no short name if '\0' specified)
  // 3rd argument is description
  // 4th argument is mandatory (optional. default is false)
  // 5th argument is default value (optional. it is used when mandatory is false)
  a.add<std::string>("host", 'h', "host name", true, "");

  // 6th argument is an extra constraint.
  // Here, the port number must be 1 to 65535, enforced by cmdline::range().
  a.add<int>("port", 'p', "port number", false, 80, cmdline::range(1, 65535));

  // cmdline::oneof() can restrict options.
  a.add<std::string>("type", 't', "protocol type", false, "http",
                     cmdline::oneof<std::string>("http", "https", "ssh", "ftp"));

  // Boolean flags can also be defined.
  // Call the add method without a type parameter.
  a.add("gzip", '\0', "gzip when transfer");

  // Run the parser.
  // It returns only if the command line arguments are valid.
  // If they are invalid, the parser prints error messages and exits the program.
  // If the help flag ('--help' or '-?') is given, it prints the usage message then exits.
  a.parse_check(argc, argv);

  // use flag values
  std::cout << a.get<std::string>("type") << "://"
            << a.get<std::string>("host") << ":"
            << a.get<int>("port") << std::endl;

  // boolean flags are queried with exist()
  if (a.exist("gzip")) std::cout << "gzip" << std::endl;
}
```

### Sample session

```
$ ./test
usage: ./test --host=string [options] ...
options:
  -h, --host    host name (string)
  -p, --port    port number (int [=80])
  -t, --type    protocol type (string [=http])
      --gzip    gzip when transfer
  -?, --help    print this message

$ ./test --host=github.com
http://github.com:80

$ ./test --host=github.com -t ftp
ftp://github.com:80

$ ./test --host=github.com -p 4545
http://github.com:4545

$ ./test --host=github.com --gzip
http://github.com:80
gzip

$ ./test --host=github.com -t ttp
option value is invalid: --type=ttp
usage: ./test --host=string [options] ...
options:
  -h, --host    host name (string)
  ...

$ ./test --host=github.com -p 100000
option value is invalid: --port=100000
usage: ./test --host=string [options] ...
options:
  -h, --host    host name (string)
  ...
```

## Extra options

### Rest of the arguments

Arguments that were not recognised as options are available through `rest()`, which returns a
vector of strings. Typically these are file names and the like.

```cpp
for (size_t i = 0; i < a.rest().size(); i++)
  std::cout << a.rest()[i] << std::endl;
```

### Footer

`footer()` appends text to the `usage:` line.

```cpp
a.footer("filename ...");
```

Result:

```
$ ./test
usage: ./test --host=string [options] ... filename ...
options:
  -h, --host    host name (string)
  ...
```

### Program name

The parser shows the program name in the usage message. By default it is taken from `argv[0]`;
`set_program_name()` overrides it with any string.

## Processing flags manually

`parse_check()` parses the command line arguments and handles both errors and the help flag for
you. You can do that work yourself instead: `bool parse()` parses the arguments and returns
whether they were valid. Check the result and do whatever you want with it.

```cpp
cmdline::parser p;
p.add("hoge", 'h', "hoge flag with no value");
p.add<int>("moge", 'm', "moge flag with int value", false, 123);
p.add("help", 0, "print help");

if (!p.parse(argc, argv) || p.exist("help")){
  std::cout << p.error_full() << p.usage();
  return 0;
}

if (p.exist("hoge"))
  std::cout << "there is hoge flag" << std::endl;

std::cout << "moge value: " << p.get<int>("moge") << std::endl;
```

(See [`test2.cpp`](test2.cpp) for more.)

## Custom readers

By default, flag values are read using `lexical_cast` (a home-grown equivalent of it). So
`add<int>(...)` reads its command line argument as a decimal integer and makes `parse` fail on a
malformed value. This behaviour is configurable: instead of `lexical_cast` you may pass an
arbitrary function object, which lets you express things like "read as hexadecimal" or "a value
within this range". For convenience, cmdline ships with ready-made readers for ranged values,
one-of-a-set values, and so on. The default behaviour is available as `cmdline::default_reader`.

```cpp
p.add<int>("hoge", 'h', "int value (100 - 999)", false, 100, cmdline::range(100, 999));
p.add<std::string>("moge", 'm', "one of abc, def, ghi", false, "abc",
                   cmdline::oneof<std::string>("abc", "def", "ghi"));
```

## API reference

### `void parser::add(const std::string &name, char short_name=0, const std::string &desc="")`

Defines a flag that carries no value. The arguments are, in order: the full name, the short
name, and a description. Passing `0` as the short name means the flag has no short form.

### `template <class T> void parser::add(const std::string &name, char short_name=0, const std::string &desc="", bool need=true, const T def=T())`

Defines a flag that carries a value. Required are the full name, the short name and the
description - along with the type, supplied as a template parameter. Two optional arguments
follow: whether the flag is mandatory (if it is, `parse` fails when the flag is absent from the
command line), and the default value used when no value is given.

### `template <class T, class F> void parser::add(const std::string &name, char short_name=0, const std::string &desc="", bool need=true, const T def=T(), F reader=F())`

Also defines a flag that carries a value, but lets you install a custom reader for it. The
method above uses `lexical_cast` as its default reader; here you may pass any function or
function object mapping a `std::string` to type `T`. If that function throws, the value is
treated as invalid and `parse()` fails.

### `bool parser::parse(int argc, const char * const argv[])`

Parses the command line arguments - pass the arguments of `main` straight through. Returns
`false` on failure and `true` on success. On failure, `error()` gives the specific message.
Overloads taking a single `std::string` or a `std::vector<std::string>` are also available.

### `void parser::parse_check(int argc, char *argv[])`

Parses, then handles errors and the help flag. If no `help` flag has been defined, it registers
one under `-?, --help`. On a parse error, or when help is requested, it prints the message and
terminates the program; it returns only when the arguments are valid.

### `bool parser::exist(const std::string &name) const`

Returns whether the given flag was present on the command line. Use after `parse()`.

### `template <class T> const T &parser::get(const std::string &name) const`

Returns the value of the given flag. Use after `parse()`. If the flag name is invalid (never
passed to `add`) or the type does not match the one used at `add` time, a `cmdline::cmdline_error`
exception is thrown.

### `const std::vector<std::string> &parser::rest() const`

Returns the arguments that were not recognised as options.

### `std::string parser::error() const`

Returns the first error message.

### `std::string parser::error_full() const`

Returns all error messages.

### `std::string parser::usage() const`

Returns the usage message.

### `void parser::footer(const std::string &f)`

Appends a string after the `usage:` line of the message returned by `usage()`.

### `void parser::set_program_name(const std::string &name)`

Sets the program name shown on the `usage:` line. When not called, `argv[0]` is used.

## Portability

In this fork improvements were made to be compatible with MSVC. Type names in the usage message are demangled through `<cxxabi.h>`, which ships with libstdc++ and libc++ but does not exist on MSVC. The include is guarded, and where the header is unavailable the raw `typeid` name is used instead, which on MSVC is already readable. Define `CMDLINE_NO_CXXABI` to force that.


## License

BSD 3-Clause. See [LICENSE](LICENSE).

Original code is Copyright (c) 2009 Hideyuki Tanaka. Modifications are Copyright (c) 2026 Ilnur
Sultanov.
