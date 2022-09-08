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

This workshop uses **[`CPython 3.10.7`][cpython-v3.10.7]**.

### 1.1 The new operator

The operator that we are going to implement is a
[pipe operator][elixer-pipe-operator]. This new binary operator will
allow us to pass a single argument to a callable in a different way:
Instead of calling the callable directly, like `function(argument)`, the
operator allows us to call the callable like `argument |> function`.

This would allow us to build a
pipeline of function calls:

```python
def double(number: int) -> int:
   return number * 2

1 |> double |> double |> double |> print  # prints 8
```

### 1.2 Setting up your development environment

Please make sure that you can compile CPython from source in your
development environment. Refer to
[the "Setup and Building" page][devguide-setup-building] in the Python
Developer's Guide or [this article][real-python-guide-cpython-source]
published by Real Python for more information on how to set up your
system.

A good test would be to try and compile the version of Python
(`v3.10.7`) that is used in this workshop.

### 1.3 Preparation

- Create a branch from the `v3.10.7` tag and switch to it.

  - Windows: Make sure that you run the `PCBuild\get_externals.bat` that
    is included in the latest commit of this new branch.

- Compile Python for the first time from this "clean" state.
  - See section 1.2 for resources on how to compile Python.

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

![Text to tokens](/img/text-to-tokens-bg.png "From text to tokens")

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

Create a python module, say `my_module.py`, in the root of the
repository that contains the following Python code:

```python
def double(number):
    return number * 2


doubled_number = double(10)
```

Open a terminal and navigate to the folder containing the temporary
module. Now run the command `python -m tokenize -e my_module.py` (
substitute the `python` command with the path to Python executable that
you compiled earlier[^1]). This should show you the output of the tokenizer,
a stream of tokens parsed from your module.

Note: The `-e` option of the `tokenize` module ensures that we get exact
token names instead of more general token names.

[^1]: On Windows, a convenience entrypoint for your compiled Python
executable will be placed in the root of the repository named
`python.bat`. On Mac/Linux, an executable file called `python` will be
placed in the root directory of the repository after running `make`.

#### 2.1.2 A missing token

Now change the line `doubled_number = double(10)` to
`doubled_number = 10 |> double` and run the tokenizer again. You should
get another stream of tokens, but, critically, the operator `|>`
was not recognized as a single token. Rather, the individual characters
that make up the operator were recognized as separate tokens, a `VBAR`
token for `|` and a `GREATER` token for `>`. 

This is not what we want: Our operator should be an atom, a single
irreducible unit of Python code and the character sequence `|>` should 
get translated to a single token.  That's why the first step in
implementing the operator is adding a token for the operator.

#### 2.1.3 Add a token for the operator

Take a look at the `Grammar\Tokens` file in the repository. This file
contains the definitions for various tokens. Note that most of the token
names are merely descriptions of the characters that make up the tokens
rather than "meaningful" names like "division". This is because these
are merely tokens without syntax and semantics.

### :joystick: Exercise 1 ###
- Try defining a token for the new operator by following the
example of existing tokens.
- After defining your token, regenerate the tokenizer:
  - Windows: `PCbuild\build.bat --regen` (may take a while the first
    time).
  - Linux/Mac: `make regen-token`
- Test your new token by running the tokenizer again (see previous
  sections). The character sequence `|>` should now be recognized as a
  single toke.

### 2.2 Adding AST support for the operator

Python's PEG parser will take the stream of tokens produced by the
tokenizer and turn it into an Abstract Syntax Tree. However, before we
can implement and compile a new grammar rule for the new operator, we
need to add support for the new operator to the AST.

More specifically, we need to create an AST node class that can
represent our operator in a binary expression (represented by a `BinOp`
node in the AST). The binary expression `10 |> double` should eventually
be translated into a tree structure like this:

![AST CallPipe Node](/img/ast-node-for-callpipe-bg.png "AST Node for our operator")

As you can see, this `BinOp` node represents the binary expression by
keeping track of the operant to the left of the operator, the operant
to the right of the operator, as well as the operator that combines the
two. The problem is that the node class `CallPipe` does not exist yet,
so we can't actually make an AST yet that uses it.

#### 2.2.1 Adding a binary operator node

Luckily, adding such a node class is easy: We don't actually have to
write any code. All the necessary AST types are generated from a
definition file written in the
[Zephyr Abstract Syntax Definition Language][zephyr-asdl]. We are going
to modify this file to define a new operator node.

### :joystick: Exercise 2 ###

- Open the file `Parser/Python.asdl` and look at lines 99-100:

```
    operator = Add | Sub | Mult | MatMult | Div | Mod | Pow | LShift
                | RShift | BitOr | BitXor | BitAnd | FloorDiv
```

- Try modifying this definition in such a way that the generator will
  generate a `CallPipe` node, in addition to the operator nodes already
  defined here.

- Regenerate the AST:
  - Windows: `PCbuild\build.bat --regen`
  - Mac/Linux: `make regen-ast`

- Recompile Python for your platform.
  - You may get a warning that `CallPipe` is not handled in a switch
    statement. You can ignore this for now, as we'll take care of it
    later by modifying Python's compiler.

- Run your newly compiled version of Python and check if it worked:

```python-repl
>>> import ast
>>> ast.CallPipe
<class 'ast.CallPipe'>
```

### 2.3 Adding a grammar rule the operator

Now that we've defined a token for the operator and added support for
the operator in the AST, we need to write a grammar rule for the PEG
parser. The grammar rule will tell the parser when to add a `BinOp` node
that includes a `CallPipe` operator node to the Abstract Syntax Tree
its creating.

Obviously, to add a grammar rule, we need to know a bit about how
Python's Parsing Expression Grammar (PEG) parser works and how you
define rules for it. Since this is quite an extensive topic, this
workshop will focus on the grammar constructs that we need to implement
the grammar rule for the new operator.

If you want to know more about Python's PEG parser, please refer to the
[excellent devguide page about the parser][peg-parser] written by Pablo
Galindo Salgado.

#### 2.3.1 A short and simplified introduction to Python's Grammar

**Note:** This introduction is extremely simplified, takes shortcuts,
suffers from blatant omissions, and focuses only on those parts that
we need to write the grammar rule for our new operator.


Python's grammar consists of rules that follow the form:
```
rule_name: grammar_expression
```

The grammar expression syntax is very flexible but also fairly simple:
There is [a limited number of syntactic rules][grammar-expressions] that
you can combine to create complex grammar rules. For the new operator,
we will need to use a few of those rules.


##### `e1 e2`
The first rule is simple: You can combine subexpressions with a space
like this:

```
some_rule: first_expression second_expression
```

This means that to match `some_rule`, the incoming stream of tokens
first has to match `e1` and then `e2`. Note that this is not _just_ an
"AND"-rule, as the order is critical as well.

##### `e1 | e2`
You can also match alternatives:

```
some_rule: first_alternative | second_alternative
```

This rule will first try to match `first_alternative` and if that works,
we've matched this rule. If and only if the `first_alternative` fails to
match will the parser consider `second_alternative`.

You'll see a lot of rules that have alternatives, like this simplified
`sum` rule:
```
sum:
    | number '+' number
    | number '-' number
    | number
```

In this rule, the `'+'` and `'-'` match a literal `+` and `-` character
respectively. Also note that within each alternative, the first rule is
used to ensure that, say, we first match a `number`, then match a `+`,
and finally match another `number` for the first alternative.

##### Rules may reference other rules

Another thing you'll notice is that most rules reference other rules
in one or more of their alternatives. This is essential: Once the parser
starts matching a rule, it will only consider the alternatives you
defined there. If a rule isn't referenced by any other rule, it will
never be considered by the parser (except the rules that serve as
entrypoints for the parser).

In fact, if you look at the `sum` rule above, it already references
another rule, `number`, which could potentially be defined as:

```
number:
    | NUMBER
    | text
```

In this example, `NUMBER` would match a `NUMBER` token in the stream
of token received by the parser from the tokenizer.

If the current token being parsed is not a `NUMBER` token, the second
alternative kicks in, which makes the parser consider the `text` rule
instead. This creates a natural flow or descent down the grammar rules,
but also defines which grammar rules are considered first, which 
influences things like operator precedence or the need for grouping in
your code.

##### Putting it all together

While this is just a small selection of syntax rules of Python's
grammar, we can already build powerful grammar rules with it.

Let's consider a very simple grammar with just two rules:

```
sum:
    | atom '+' atom
    | atom

atom:
    | NUMBER
```

If we assume that `NUMBER` refers to the `NUMBER` token, it is fairly
trivial to see that these rules can match the following stream of
tokens:

```
1
```

The parser would first consider the first alternative of `sum`, which is
`atom '+' atom`. This rule will fail, as there is no literal `+`
character in the stream of tokens. Once this fails, the parser will try
the second alternative, which is just the `atom` rule on its one and
that will lead to a match, as the `atom` rule is just a `NUMBER` token
on its own. Likewise, no one will be surprised that the rules can match
`1 + 1`, as this perfectly fits the first alternative of the `sum` rule.

Now consider the following stream of tokens:

```
1 + 2 + 3
```

Are the rules able to match this expression? Do think about it for a
moment before reading on.

The answer is "no". The problem is that the parser will start matching
the stream of tokens just fine, as the first part, `1 + 2`, will match
the first alternative of the `sum` rule just fine. However, after that
rule is matched, those tokens are consumed, which leaves us with `+ 3`.
As there is no rule that starts with a `'+'`, the grammar can't match
this part. So what should we do to support this?

We could add another alternative to the sum rule:

```
sum:
    | atom '+' atom '+' atom
    | atom '+' atom
    | atom

atom:
    | NUMBER
```

Now the expression `1 + 2 + 3` can be matched, but what about
`1 + 2 + 3 + 4`? Or `1 + 2 + 3 + 4 + 5`? What if we want to match an
expression with an arbitrary number of pluses? Adding an infinite number
of alternatives is not going to be fun. Clearly, we're missing something
to help us here.

As always, the answer here is recursion. The only thing we need to tweak
is the very first part of the first alternative of the `sum` rule:

```
sum:
    | sum '+' atom
    | atom

atom:
    | NUMBER
```

Now, the first alternative of the `sum` rule _may contain itself_. This
means that we can match `1 + 2`, as `1` is a valid alternative of the
`sum` rule (the second one), but also `1 + 2 + 3`, as `1 + 2` we've just
established that `1 + 2` is a valid alternative of the sum rule`, so 
`some_matched_sum + 3` is also a valid alternative of the sum rule.

And if you think about it, this is precisely the kind of structure that
we'd need to be able to match for our pipe operator, as our "pipelines"
can become arbitrarily long:

```
1 |> double |> double |> double |> triple |> print
```

### 2.3.2 Writing the grammar rule

Now it's time to write the grammar rule for the pipe operator. You can
find Python's grammar rules in the file `Grammar/python.gram`. Take a
moment to scroll through the file, but don't feel bad if you feel
overwhelmed; Python's grammar is huge and has grown considerably in size
in the last few versions.

To simplify the matter, we are going to add our grammar rule between the
`shift_expr` rule (lines 646-649) and `sum` (lines 651-554). Compared to
the grammar rules above, you'll notice that these rules have a few extra
bits, like `[expr_ty]` and `{ _PyAST_BinOp(a, Add, b, EXTRA) }`. The
former just means that matching this rule will return an expression type
and the latter is something we'll see later. You may ignore those bits
for now.

### :joystick: Exercise 3 ###

- As a start, add this rule template between the two rules:

```
shift_expr[expr_ty]:
    | a=shift_expr '<<' b=sum { _PyAST_BinOp(a, LShift, b, EXTRA) }
    | a=shift_expr '>>' b=sum { _PyAST_BinOp(a, RShift, b, EXTRA) }
    | sum

pipe[expr_ty]:
    |
    | sum
    
sum[expr_ty]:
    | a=sum '+' b=term { _PyAST_BinOp(a, Add, b, EXTRA) }
    | a=sum '-' b=term { _PyAST_BinOp(a, Sub, b, EXTRA) }
    | term
```

- Now try writing a grammar rule that would match valid expressions that
  use the pipe operator. Some things to consider:
  - You can use `'|>'` to match the operator itself, just like the `sum`
    rule uses `'+'`.
  - Think about how the grammar rules cascade down: The `sum` rule is
    only ever considered because it's referenced by the `shift_expr`
    rule. Try to modify the `shift_expr` rule to make the parser
    consider the `pipe` rule which, in turn, should make the parser
    consider the `sum` rule.
  - How could you use recrusion to match an arbitrary number of pipe
    operators in a "pipeline"?
  - Don't worry about things like `{ _PyAST_BinOp(a, Add, b, EXTRA) }`
    in this exercise.

### 2.3.3 Grammar Actions

After writing your grammar rule, the parser will be able to match an
expression that includes a pipe operator. Still, it doesn't yet know how
to turn what it has matched into an Abstract Syntax Tree node. This is
where [Grammar Actions][grammar-actions] come to the rescue. Grammar
actions are just small snippets of C-code that tell the parser what it
should do with the things it has matched.

As you have probably guessed by now, this is where the curly braces come
into play: They allow you to embed snippets of C that get executed to do
the work of creating the necessary AST nodes (and so on).

If you look at the first alternative of the `sum` rule, it defines the
grammar action `{ _PyAST_BinOp(a, Add, b, EXTRA) }` that gets executed
whenever that alternative is matched. This grammar action just calls a
special C function, `_PyAST_BinOp` that will create the actual `BinOp`
AST node that we need. 

Obviously, that function can only create the `BinOp` node if it knows
which operator was used and what was to the left/right of the operator.
As we know that this grammar action gets executed if we matched the
first alternative of the `sum` rule, we know that the appropriate
operator AST node is `Add` (which is part of the `operator` list you've
seen in exercise 2).

However, to know the left and the right operant, we actually need to
know what was matched. This is what the `a=` and `b=` in the rule do:
They assign names to whatever was matched on the left-hand side and the
right-hand side of the operator respectively. Once we've assigned those
names, we can simply pass them in as arguments within the grammar
action.

The `EXTRA` argument in the `PyAST_BinOp` is a special macro that passes
parser context into the function, things like line number and column
number, but we're not going to dive into that in this workshop.

### :joystick: Exercise 4 ###

- Copy the grammar action of the first `sum` alternative and paste it
  as the grammar action of the first alternative of the `pipe` rule.

- Make sure that you change the operator, `Add`, to the operator we've
  defined in exercise 2.

- Assign the names `a` and `b` to the left-hand side and the right-hand
  side of the operator in the grammar expression respectively.

- Regenerate Python's PEG parser including the new rule:
  - Windows: `PCbuild\build.bat --regen`
  - Mac/Linux: `make regen-pegen`

- Compile Python to make it include the new parser.

- Start your newly compiled Python and test if it works:

```python-repl
>>> import ast
>>> tree = ast.parse("10 |> double")
>>> tree.body[0].value
<ast.BinOp object at 0x0000026C86862DA0>
>>> tree.body[0].value.op
<ast.CallPipe object at 0x0000026C86BE6580>
```

What doesn't work is actually trying to use the expression:
```python-repl
>>> def double(number): return 2 * number
... 
>>> 10 |> double
SystemError: binary op 14 should not be possible
```

This is because there is no actual implementation for this binary
operator yet.

### 2.5 Solutions for this part

This section contains the changes that you had to make to create support
for the pipe operator in Python's tokenizer, grammar, and AST.

#### 2.5.1 File `Grammar/Tokens`
Add the following line:
```
VBARGREATER             '|>'
```
(Picking another name for the token is fine.)

#### 2.5.2 File `Parser/python.asdl`

Add `CallPipe` to the `operator` list on lines 99-100:
```
    operator = Add | Sub | Mult | MatMult | Div | Mod | Pow | LShift
                 | RShift | BitOr | BitXor | BitAnd | FloorDiv | CallPipe
```

#### 2.5.3 File `Grammar/python.gram`

Add the grammar rule `pipe` and make sure `shift_expr` (lines 656-659) 
refers to it in ALL its alternatives:
```
shift_expr[expr_ty]:
    | a=shift_expr '<<' b=pipe { _PyAST_BinOp(a, LShift, b, EXTRA) }
    | a=shift_expr '>>' b=pipe { _PyAST_BinOp(a, RShift, b, EXTRA) }
    | pipe

pipe[expr_ty]:
    | a=pipe '|>' b=sum { _PyAST_BinOp(a, CallPipe, b, EXTRA) }
    | sum

sum[expr_ty]:
    | a=sum '+' b=term { _PyAST_BinOp(a, Add, b, EXTRA) }
    | a=sum '-' b=term { _PyAST_BinOp(a, Sub, b, EXTRA) }
    | term
```

Note: the `sum` rule is unchanged but was included to help you find the
right location in the file to add the `pipe` rule.

## 2. From Abstract Syntax Tree to Bytecode


[cpython-v3.10.7]: https://github.com/python/cpython/tree/v3.10.7
[elixer-pipe-operator]: https://elixirschool.com/en/lessons/basics/pipe_operator
[devguide-setup-building]: https://devguide.python.org/getting-started/setup-building/
[real-python-guide-cpython-source]: https://realpython.com/cpython-source-code-guide/
[wikipedia-ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[py-glossary-bytecode]: https://docs.python.org/3/glossary.html#term-bytecode
[ellipsis]: https://docs.python.org/3/library/constants.html#Ellipsis
[zephyr-asdl]: http://asdl.sourceforge.net/
[peg-parser]: https://devguide.python.org/internals/parser/
[grammar-expressions]: https://devguide.python.org/internals/parser/#syntax
[grammar-actions]: https://devguide.python.org/internals/parser/#grammar-actions
