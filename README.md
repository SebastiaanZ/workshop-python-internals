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

### 1.1. The new operator

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

### 1.2. Setting up your development environment

Please make sure that you can compile CPython from source in your
development environment. Refer to
[the "Setup and Building" page][devguide-setup-building] in the Python
Developer's Guide or [this article][real-python-guide-cpython-source]
published by Real Python for more information on how to set up your
system.

A good test would be to try and compile the version of Python
(`v3.10.7`) that is used in this workshop.

### 1.3. Preparation

- Create a branch from the `v3.10.7` tag and switch to it:
  - If you created a fork:
    - `git remote add upstream <url-to-original-cpython-repo>`
    - `git fetch upstream refs/tags/v3.10.7`
    - `git checkout v3.10.7 -b cpython-v3.10.7`
  - If you cloned the cpython repo directly:
    - `git fetch origin refs/tags/v3.10.7`
    - `git checkout v3.10.7 -b cpython-v3.10.7`
  - In both cases, the name of your branch is `cpython-v3.10.7`.

- Windows: Make sure that you run the `PCBuild\get_externals.bat` that
  is included in the latest commit of this new branch.

- Compile Python for the first time from this "clean" state.
  - See section 1.2 for resources on how to compile Python.

---

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

### 2.1. A new token for the tokenizer

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

#### 2.1.1. Exploring the tokenizer

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

#### 2.1.2. A missing token

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

#### 2.1.3. Add a token for the operator

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

### 2.2. Adding AST support for the operator

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
keeping track of the operand to the left of the operator, the operand
to the right of the operator, as well as the operator that combines the
two. The problem is that the node class `CallPipe` does not exist yet,
so we can't actually make an AST yet that uses it.

#### 2.2.1. Adding a binary operator node

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

### 2.3. Adding a grammar rule the operator

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

#### 2.3.1. A short and simplified introduction to Python's Grammar

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

### 2.3.2. Writing the grammar rule

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

### 2.3.3. Grammar Actions

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

However, to know the left and the right operand, we actually need to
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

### 2.5. Solutions for this part

This section contains the changes that you had to make to create support
for the pipe operator in Python's tokenizer, grammar, and AST.

#### 2.5.1. File `Grammar/Tokens`
Add the following line:
```
VBARGREATER             '|>'
```
(Picking another name for the token is fine.)

#### 2.5.2. File `Parser/python.asdl`

Add `CallPipe` to the `operator` list on lines 99-100:
```
    operator = Add | Sub | Mult | MatMult | Div | Mod | Pow | LShift
                 | RShift | BitOr | BitXor | BitAnd | FloorDiv | CallPipe
```

#### 2.5.3. File `Grammar/python.gram`

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

---

## 3. From Abstract Syntax Tree to Bytecode

At this point, Python's parser can take an expression containing the new
pipe operator and turn it into the relevant Abstract Syntax Tree nodes.
Now we are going to add support to the compiler so that it can compile
the generated AST into an intermediate language called
[bytecode][term-bytecode]. This is the language that actually gets
interpreted by the evaluation loop.

### 3.1. Bytecode

Bytecode is an intermediate language that in its core is a long sequence
of instructions or"operation codes ("opcodes") for the interpreter. Each
instruction tells the interpreter to do "something", like getting the
value that a name was assigned to (`LOAD_NAME` or `LOAD_FAST`) or adding
two value together (`BINARY_ADD`).

Each instruction takes up two bytes in the bytecode, which means that
they can, very conveniently, be represented as an unsigned short
integer, although we often refer to them by their human-readable name.
If you've completed a few puzzles of the
[Advent of Code 2019][aoc-2019], this should feel very familiar to you:
Bytecode is very similar to the input for the `intcode` computer/virtual
machine that you were asked to code there.

### :joystick: Exercise 5 ###

We're going to look at the disassembled bytecode for a "compiled"
object, like a function, class, method, or generator, using the `dis`
module from Python's standard library.

- Open a Python REPL/Interactive shell and type in the following:

```python-repl
>>> import dis
>>> dis.dis(lambda x, y: 2 * x + y)
```

- Try to understand the disassembled bytecode instructions for the
  _body_ of the function created with the `lambda`-expression. To help
  you, this would be the equivalent function definition (minus the
  function name):

```python
def anonymous_lambda_function(x, y):
    return 2 * x + y
```

### 3.2. Adding an opcode for the operator

The compiler needs to be able to compile the AST node for the new
operator into a bytecode instruction or opcode. There's just one tiny
problem: The opcode for our operator doesn't exist yet[^2], so we'll
have to define it before we can modify the compiler.

[^2]: In reality, we do already have an opcode for calling a function,
which is what our operator does. However, for educational purposes, we
are going to ignore that and add our own.

### :joystick: Exercise 6 ###

- Open the file `Lib/opcode.py`, which is the only Python-file that we
  are going to modify today, and scroll through the contents.

As you might have noticed, the interface for defining an opcode is very
simple: You simply add a function call to `def_op` and pass in the
human-readable name of the opcode and its integer value. **There is one
catch:** The constant on line 131, `HAVE_ARGUMENT`, is important, as it
specifies that all the opcodes with that integer value and higher have
an additional argument. The opcode that we are going to define for our
operator does **not** take an argument, so we have to take this constant
into account.

### :joystick: Exercise 7 ###

- Add an opcode for our operator using the name `"BINARY_CALL_PIPE"`
  - Make sure to give it an integer value below `HAVE_ARGUMENT`. Note
    that blank lines correspond to available opcodes.

- Regenerate Python's opcode tables:
  - Windows: `PCbuild\build.bat --regen`
  - Mac/Linux: `make regen-opcode`

### 3.3. Python's compiler

Python's compiler compiles the Abstract Syntax Tree by visiting all the
nodes in the tree structure. For each node type, a dedicated
`compiler_visit` function gets called that defines how the compiler
will handle that node. This function may ask the compiler to visit child
nodes of the current node, like the left or right operand of a binary
operation.

A lot of these `compiler_visit_*` functions contain a `switch` block, 
like this simplified version of the `compiler_visit_expr1` function
below. This function eventually gets called when the compiler visits an
expression type node, including expressions with a binary operator like
our pipe operator. 

```C
// File: Python/compile.c, starting at line 5180.
static int compiler_visit_expr1(struct compiler *c, expr_ty e)
{
    switch (e->kind) {
    // Other cases removed for brevity
    case BinOp_kind:
        VISIT(c, expr, e->v.BinOp.left);
        VISIT(c, expr, e->v.BinOp.right);
        ADDOP(c, binop(e->v.BinOp.op));
        break;
    }
    // Other cases removed for brevity
}
```

The `switch` block here has a `case` for expressions of the `BinOp_kind`
that instructs the compiler what it should do when it encounters a
binary expression.

It's important to note that the first thing the compiler is instructed
to do is to VISIT the left-hand side of the operator so it can write the
bytecode instructions to resolve that value, then it is instructed to
VISIT the right-hand side to do the same, and only then is it instructed
to write the opcode for the binary operator with `ADDOP`.

This makes sense if you think about it: If we actually want to perform
the operation, we already need to know the two values that the operator
is applied to. The order in which we visit these operands, left before
right, will be important later.

### 3.3.1. Compiling CallPipe operators

Now that we've identified the part of the compiler that handles binary
operators, we can add support for our `CallPipe` AST node. As you can
see above, we will add an opcode with `ADDOP` and determine which opcode
to write by calling the function `binop`

This `binop` function, starting at line 3607 in `Python/compile.c`,
consists of a large `switch` block that has a `case` for all types of
operator nodes. Except for our `CallPipe` node, that is.

### :joystick: Exercise 8 ###

- Open the file `Python/compiler.c` and scroll to the function `binop`
  at line 3607. 

- Read through the cases and note that there's a case for each AST
  operator node defined in `Parser/Python.asdl` (see also exercise 2)
  that maps that operator to an opcode.

- Add a case for the `CallPipe` AST node and make it return the opcode
  you defined in exercise 6.

### 3.3.2. Defining the stack effect of our operator

There's one additional thing that we need to modify in the compiler to
make it work: The effect that an opcode has on the **value stack**. The
value stack is a stack that the evaluation loop will use to keep track
of the currently evaluated values that it is working with. Performing an
operation often modifies this stack, either by removing one or more
values, pushing one or more values on the stack, or both. 

Our binary operation will also mutate the stack: It will "process" the
`left` and `right` operands, removing them from the stack, and push the
result of the operation, the function call, on the stack. This means
that the net stack effect of pipe operator is that it removes 1 value
from the value stack. We need to specify this "stack effect" of the
opcode in the compiler.

### :joystick: Exercise 9 ###

- Open the file `Python/compiler.c` and scroll to the function
  `stack_effect` at line 955. 

- Skim through this function and notice that most binary operators have
  a stack effect of `-1`:

```C
        /* Binary operators */
        case BINARY_POWER:
        case BINARY_MULTIPLY:
        case BINARY_MATRIX_MULTIPLY:
        case BINARY_MODULO:
        case BINARY_ADD:
        case BINARY_SUBTRACT:
        case BINARY_SUBSCR:
        case BINARY_FLOOR_DIVIDE:
        case BINARY_TRUE_DIVIDE:
            return -1;
```

- Add a line with `case BINARY_CALL_PIPE:` to this list to indicate that
  our binary operation also has a stack effect of `-1`.

- Recompile your version of Python to check your changes. It should
  compile without errors. Actually trying to run an expression with the
  pipe operator will still fail, as we haven't told the evaluation loop
  what it should actually do when it evaluates the opcode yet...

### 3.4. Changing the `MAGIC_NUMBER` (optional)

Running the compiler takes a bit of time. That's why Python caches the
compiled bytecode for a module in `.pyc` files (typically located in
a `__pycache__` directory). This works fine as long as the bytecode
doesn't change. Since we've made a change to the bytecode, by adding a
new opcode, we need to tell Python that old `.pyc`-files are stale.[^3]

[^3]: Since you probably added an opcode integer in a "gap" without
changing existing opcode integers, trying to use an old `.pyc` file may
still work. Still, we did create a new version of the bytecode.

You can do that by incrementing the `MAGIC_NUMBER`, which acts as a 
version number for bytecode. If Python encounters a `.pyc`-file, it will
compare the `MAGIC_NUMBER` stored in the `.pyc` file with the current
`MAGIC_NUMBER`. If they don't match, it will consider the file to be
stale, which means that Python will recompile the original file.

### :joystick: Exercise 10 (optional) ###

- Increment the value of `MAGIC_NUMBER` in
  `Lib/importlib/_bootstrap_external.py`

- Regenerate importlib to make it aware of the new `MAGIG_NUMBER`
  - Mac/Linux: `make regen-importlib`
  - Windows: `PCbuild\build.bat --regen` (although this will also happen
    automatically when you compile Python)

- Recompile Python using your preferred method

---
## 4. From bytecode to execution

The compiler will now generate bytecode, a sequence of instructions or
opcodes, that will be executed, but what will actually execute these
instructions? This is where the **evaluation loop** comes in: This is
the loop that will interpret the bytecode. You can find the
implementation of the evaluation loop in `Python/ceval.c`. 

### :joystick: Exercise 11 ###

- Open the file `Python/ceval.c` and scroll down to line 1739. This is
  the start of Python's main loop that will interpret the bytecode.

- Now scroll down to line 1848. This is the start of a huge `switch`
  block within the main loop that has a `case` for each opcode that
  the evaluation loop can interpret. Scroll through a few cases to get
  feel for the structure of a `case`.

### 4.1. Interpreting a binary operator

As our operator is a binary operator, we are going to look at the `case`
for an existing binary opcode, `BINARY_SUBTRACT`. A binary subtract is
the operation of subtracting one value from another value using the `-`
operator, as in `4 - 3`. This is the case that handles the opcode
`BINARY_SUBTRACT`:

```C
        case TARGET(BINARY_SUBTRACT): {
            PyObject *right = POP();
            PyObject *left = TOP();
            PyObject *diff = PyNumber_Subtract(left, right);
            Py_DECREF(right);
            Py_DECREF(left);
            SET_TOP(diff);
            if (diff == NULL)
                goto error;
            DISPATCH();
        }
```

### 4.1.1 Getting the operand values

A binary subtract, like `a - b`, performs an operation on two operands,
the value to the left of the operator and the value to the right of the
operator. Remember from earlier in the workshop that the compiler will
first write the instructions to evaluate the left operand, then the
right operand, and only then will it write the opcode to perform the 
operation.

This means that once the evaluation loop gets to the opcode for the
operation, the operand values have already been evaluated. While this is
handy, it leaves us with a problem: How do we get those values to
perform the operation?

This is where the value stack comes in. When the interpreter is done
evaluating a (sub-)expression, it will push the value onto the value
stack to keep track of it. In our case, this means that after evaluating
the `left` operand, the value for `left` will now be available on the
value stack. Then the evaluation loop will evaluate the `right` operand
and push that value **on top of** the value for the `left` operand onto
the value stack.

After evaluating the `left` and `right` operands, the value stack will
look like this:

![Valuestack after left/right](/img/value-stack-bg.png "Value stack after evaluating the left and right operand")

Since the values are available on the value stack, the `case` for
`BINARY_SUBTRACT` can get these values from the stack. This is what
happens in the first two lines within the `case` block:

```C
            PyObject *right = POP();
            PyObject *left = TOP();
```

The `POP()` macro will pop the last value from the value stack, which is
the **right** operand (since it was evaluated last). The `TOP()` macro
will "peek" at the top of the stack to get the value of the **left**
operand, which is now at the top of the value stack after popping the
**left** operand value. This `TOP()` macro will not remove the value
from the stack, so the number `4` is still the top value on the stack.

Now that we have the `right` and `left` values, we can calculate the
difference by calling a cAPI function called `PyNumber_Subtract`:

```C
            PyObject *diff = PyNumber_Subtract(left, right);
```

This returns the computed difference, which we call `diff`.

At this point, we're done with the `right` and `left` operand, which
means that we'll decrease the reference counts:

```C
            Py_DECREF(right);
            Py_DECREF(left);
```

However, what about the result of our operation? How do we make it
available as the result of this binary expression? We make sure it's
available on top of the stack! However, instead of just pushing it,
we are going to replace the left operand, `4`, that was still lingering
on top with the `SET_TOP` macro:

```C
            SET_TOP(diff);
```

All that's left to do now is to do some error handling, which you may
ignore in this workshop:

```C
            if (diff == NULL)
                goto error;
```

At the end of the `case`-block, we tell the evaluation loop to continue
to the next opcode in the bytecode by calling the `DISPATCH` macro:

```C
            DISPATCH();
```

### 4.2. Implementing a case for the pipe operator

As our binary operation is very similar in structure to a binary
subtract, we can copy the `BINARY_SUBTRACT` case and modify it for our
use. Obviously, we can't use `PyNumber_Subtract` anymore, but, luckily
for us, there's a very convenient function we can use instead:
`PyObject_CallOneArg(function, argument)`.

### :joystick: Exercise 12 ###

- Copy-paste the `case` for `TARGET(BINARY_SUBTRACT)` in the file
  `Python/ceval.c` to create a duplicate. You can find it on line 2094.

- Change the `TARGET` of the `case` to our opcode.

- Thing about how you can modify this case to fit our needs.
  - Hint `PyObject_CallOneArg` is a convencience function to call a
    function with one argument.

- After you're done, recompile Python.

- Open your new version of Python and try it out!

---

### 5. Conclusion

You should now have a working version of Python that includes a pipe
operator. There are a few important remarks, though:

- The location I picked to insert our grammar rule was convenient, as we
  only had to change a few rule references. However, it also means that
  precedence of grammar rules isn't ideal: If you want to use a lambda
  expression to define a function within a pipe expression, you have to
  surround it with parentheses.

- Adding a new opcode that just calls a function with one argument isn't
  ideal. An opcode for calling functions already exists,
  `CALL_FUNCTION`. An actual implementation should have used this one
  instead of defining a redundant one.

In conclusion, the version we've implemented is not suitable for =
production and only serves an educational purpose.

## 6. Solutions to part 3 & 4

- Modify the `binop` function in `Python/compile.c`:
```C
static int
binop(operator_ty op)
{
    switch (op) {
    // Other cases removed for brevity
    case CallPipe:
        return BINARY_CALL_PIPE;
    default:
        PyErr_Format(PyExc_SystemError,
            "binary op %d should not be possible", op);
        return 0;
    }
}
```

- Modify the `stack_effect` function in `Python/compile.c`:
```C
static int
stack_effect(int opcode, int oparg, int jump)
{
    switch (opcode) {
        // Other cases removed

        /* Binary operators */
        case BINARY_POWER:
        case BINARY_MULTIPLY:
        case BINARY_MATRIX_MULTIPLY:
        case BINARY_MODULO:
        case BINARY_ADD:
        case BINARY_SUBTRACT:
        case BINARY_SUBSCR:
        case BINARY_FLOOR_DIVIDE:
        case BINARY_TRUE_DIVIDE:
        case BINARY_CALL_PIPE:  // <- this line was inserted
            return -1;
        
        // Other cases removed
        
        default:
            return PY_INVALID_STACK_EFFECT;
    }
    return PY_INVALID_STACK_EFFECT; /* not reachable */
}
```

- Define a `BINARY_CALL_PIPE` opcode. I added it to a group with a few
  other binary opcodes, but other integers work, too, as long as you
  picked one that was not already in use.
```python
def_op('BINARY_CALL_PIPE', 35)
```

- Add a case for the new opcode to the evaluation loop in
  `Python/ceval.c`. I've added mine just below the `case` for
  `TARGET(BINARY_SUBTRACT)`, but, in principle, it can be anywhere
  within the loop.
```C
        case TARGET(BINARY_CALL_PIPE): {
            PyObject *right = POP();
            PyObject *left = TOP();
            PyObject *diff = PyObject_CallOneArg(right, left);
            Py_DECREF(right);
            Py_DECREF(left);
            SET_TOP(diff);
            if (diff == NULL)
                goto error;
            DISPATCH();
        }
```



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
[term-bytecode]: https://docs.python.org/3/glossary.html#term-bytecode
[aoc-2019]: https://adventofcode.com/2019
[devguide-magic-number]: https://devguide.python.org/internals/compiler/#introducing-new-bytecode
