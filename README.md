# Workshop Python Internals
- **Date:** 2022-09-09
- **Author:** Sebastiaan Zeeff (Ordina Pythoneers)
- **Event:** Pythoneersday of the Ordina Pythoneers

---

## 1. Introduction

In this workshop, we are going to explore the internals of Python by
implementing a new operator. This implementation is going to take us on
a journey through the steps that Python takes to execute your
applications, from parsing your source code to evaluating bytecode.

Do note that the quality of the implementation is not in itself all that
important in this workshop, as the main goal is the get a better
understanding of the Python internals. This means that we're going to
take some shortcuts in our implementation just so we can make the entire
journey.

This workshop uses **CPython 3.9.14**.

### 1.1 The new operator

The operator that we are going to implement is a
[pipe operator][elixer-pipe-operator]. This new binary operator will
allow us to pass a single argument to a callable in a different way:
Instead of calling the callable directly, like `function(argument)`, the
operator allows us to call the callable like `argument |> function`.

This would allow us to build a
pipeline of function calls:

```python-repl
>>> def double(number: int) -> int:
...     return number * 2

>>> 1 |> double |> double |> double |> print
8
```

### 1.2 Prerequisites

Please make sure that you can compile CPython from source in your
development environment. Refer to
[the "setup building" page][devguide-setup-building] in the Python
Developer's Guide for more information on how to set up your system. A
good test would be to try and compile the version of Python (`v3.9.14`)
that is included in the `cpython/v3.9.14` branch in this repository.

## 2. From Source Code to Abstract Syntax Tree

![Code to AST](/img/code-to-ast-bg.png "From Source Code to Abstract Syntax Tree")

In this part of the workshop, which should take you roughly half of the
scheduled time, our goal is to get from "raw source code" to something
called an [Abstract Syntax Tree (AST)][wikipedia-ast]. An AST is a tree
that represents your source code in a way that's much easier to work
with for the compiler that compiles your code to
[bytecode][py-glossary-bytecode].

To add support for the new operator, we need to make the following
changes to CPython:

- Add a token for the operator to the tokenizer
- Add a grammar rule for the operator to the PEG parser
- Add a node class for the operator to the AST types

### 2.1 A new token for the tokenizer

When you type your source code into your editor or IDE, you are just
typing text. This seems trivial, but this means that when Python starts
reading your source file, it initially just reads a stream of
characters that it needs to make sense of.

The first step Python takes is turning your source file into a stream 
of _tokens_, the smallest atoms that still have a meaning in the language. As
an example, if you type in `...` (the literal for the built-in constant
[Ellipsis][ellipsis]), the tokenizer will recognize this as a single
token. If you were to take away just a single dot, it would no longer be
an Ellipsis.

#### 2.1.1 Exploring the tokenizer

Create a temporary python module, say `my_module.py`, that contains the
following Python code:

```python
def double(number: int) -> int:
    return number * 2


doubled_number = double(10)
print(doubled_number)
```

Open a terminal and navigate to the folder containing the temporary
module. Now run the command `python -m tokenize -e my_module.py` (
substitute the `python` command by the command you use to run Python
3.9 on your system). This should show you the output of the tokenizer,
a stream of tokens parsed from your module.

Now change the line `doubled_number = double(10)` to
`doubled_number = 10 |> double` and run the tokenizer again. You should
get another stream of tokens, but, critically, the operator `|>`
was not recognized as a single token. Rather, the individual characters
that make up the operator were recognized as separate tokens, a `VBAR`
token for `|` and a `GREATER` token for `>`. 



[elixer-pipe-operator]: https://elixirschool.com/en/lessons/basics/pipe_operator
[devguide-setup-building]: https://devguide.python.org/getting-started/setup-building/
[wikipedia-ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[py-glossary-bytecode]: https://docs.python.org/3/glossary.html#term-bytecode
[ellipsis]: https://docs.python.org/3/library/constants.html#Ellipsis
