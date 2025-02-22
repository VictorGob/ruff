---
title: "Known Deviations from Black"
hide:
  - navigation
---

This document enumerates the known, intentional differences in code style between Black and Ruff's
formatter.

For a list of unintentional deviations, see [issue tracker](https://github.com/astral-sh/ruff/issues?q=is%3Aopen+is%3Aissue+label%3Aformatter).

### Trailing end-of-line comments

Black's priority is to fit an entire statement on a line, even if it contains end-of-line comments.
In such cases, Black collapses the statement, and moves the comment to the end of the collapsed
statement:

```python
# Input
while (
    cond1  # almost always true
    and cond2  # almost never true
):
    print("Do something")

# Black
while cond1 and cond2:  # almost always true  # almost never true
    print("Do something")
```

Ruff, like [Prettier](https://prettier.io/), expands any statement that contains trailing
end-of-line comments. For example, Ruff would avoid collapsing the `while` test in the snippet
above. This ensures that the comments remain close to their original position and retain their
original intent, at the cost of retaining additional vertical space.

This deviation only impacts unformatted code, in that Ruff's output should not deviate for code that
has already been formatted by Black.

### Pragma comments are ignored when computing line width

Pragma comments (`# type`, `# noqa`, `# pyright`, `# pylint`, etc.) are ignored when computing the width of a line.
This prevents Ruff from moving pragma comments around, thereby modifying their meaning and behavior:

See Ruff's [pragma comment handling proposal](https://github.com/astral-sh/ruff/discussions/6670)
for details.

This is similar to [Pyink](https://github.com/google/pyink) but a deviation from Black. Black avoids
splitting any lines that contain a `# type` comment ([#997](https://github.com/psf/black/issues/997)),
but otherwise avoids special-casing pragma comments.

As Ruff expands trailing end-of-line comments, Ruff will also avoid moving pragma comments in cases
like the following, where moving the `# noqa` to the end of the line causes it to suppress errors
on both `first()` and `second()`:

```python
# Input
[
    first(),  # noqa
    second()
]

# Black
[first(), second()]  # noqa

# Ruff
[
    first(),  # noqa
    second(),
]
```

### Line width vs. line length

Ruff uses the Unicode width of a line to determine if a line fits. Black's stable style uses
character width, while Black's preview style uses Unicode width for strings ([#3445](https://github.com/psf/black/pull/3445)),
and character width for all other tokens. Ruff's behavior is closer to Black's preview style than
Black's stable style, although Ruff _also_ uses Unicode width for identifiers and comments.

### Walruses in slice expressions

Black avoids inserting space around `:=` operators within slices. For example, the following adheres
to Black stable style:

```python
# Input
x[y:=1]

# Black
x[y:=1]
```

Ruff will instead add space around the `:=` operator:

```python
# Input
x[y:=1]

# Ruff
x[y := 1]
```

This will likely be incorporated into Black's preview style ([#3823](https://github.com/psf/black/pull/3823)).

### `global` and `nonlocal` names are broken across multiple lines by continuations

If a `global` or `nonlocal` statement includes multiple names, and exceeds the configured line
width, Ruff will break them across multiple lines using continuations:

```python
# Input
global analyze_featuremap_layer, analyze_featuremapcompression_layer, analyze_latencies_post, analyze_motions_layer, analyze_size_model

# Ruff
global \
    analyze_featuremap_layer, \
    analyze_featuremapcompression_layer, \
    analyze_latencies_post, \
    analyze_motions_layer, \
    analyze_size_model
```

### Newlines are inserted after all class docstrings

Black typically enforces a single newline after a class docstring. However, it does not apply such
formatting if the docstring is single-quoted rather than triple-quoted, while Ruff enforces a
single newline in both cases:

```python
# Input
class IntFromGeom(GEOSFuncFactory):
    "Argument is a geometry, return type is an integer."
    argtypes = [GEOM_PTR]
    restype = c_int
    errcheck = staticmethod(check_minus_one)

# Black
class IntFromGeom(GEOSFuncFactory):
    "Argument is a geometry, return type is an integer."
    argtypes = [GEOM_PTR]
    restype = c_int
    errcheck = staticmethod(check_minus_one)

# Ruff
class IntFromGeom(GEOSFuncFactory):
    "Argument is a geometry, return type is an integer."

    argtypes = [GEOM_PTR]
    restype = c_int
    errcheck = staticmethod(check_minus_one)
```

### Trailing own-line comments on imports are not moved to the next line

Black enforces a single empty line between an import and a trailing own-line comment. Ruff leaves
such comments in-place:

```python
# Input
import os
# comment

import sys

# Black
import os

# comment

import sys

# Ruff
import os
# comment

import sys
```

### Parentheses around awaited collections are not preserved

Black preserves parentheses around awaited collections:

```python
await ([1, 2, 3])
```

Ruff will instead remove them:

```python
await [1, 2, 3]
```

This is more consistent to the formatting of other awaited expressions: Ruff and Black both
remove parentheses around, e.g., `await (1)`, only retaining them when syntactically required,
as in, e.g., `await (x := 1)`.

### Implicit string concatenations in attribute accesses

Given the following unformatted code:

```python
print("aaaaaaaaaaaaaaaa" "aaaaaaaaaaaaaaaa".format(bbbbbbbbbbbbbbbbbb + bbbbbbbbbbbbbbbbbb))
```

Internally, Black's logic will first expand the outermost `print` call:

```python
print(
    "aaaaaaaaaaaaaaaa" "aaaaaaaaaaaaaaaa".format(bbbbbbbbbbbbbbbbbb + bbbbbbbbbbbbbbbbbb)
)
```

Since the argument is _still_ too long, Black will then split on the operator with the highest split
precedence. In this case, Black splits on the implicit string concatenation, to produce the
following Black-formatted code:

```python
print(
    "aaaaaaaaaaaaaaaa"
    "aaaaaaaaaaaaaaaa".format(bbbbbbbbbbbbbbbbbb + bbbbbbbbbbbbbbbbbb)
)
```

Ruff gives implicit concatenations a "lower" priority when breaking lines. As a result, Ruff
would instead format the above as:

```python
print(
    "aaaaaaaaaaaaaaaa" "aaaaaaaaaaaaaaaa".format(
        bbbbbbbbbbbbbbbbbb + bbbbbbbbbbbbbbbbbb
    )
)
```

In general, Black splits implicit string concatenations over multiple lines more often than Ruff,
even if those concatenations _can_ fit on a single line. Ruff instead avoids splitting such
concatenations unless doing so is necessary to fit within the configured line width.

### Own-line comments on expressions don't cause the expression to expand

Given an expression like:

```python
(
    # A comment in the middle
    some_example_var and some_example_var not in some_example_var
)
```

Black associates the comment with `some_example_var`, thus splitting it over two lines:

```python
(
    # A comment in the middle
    some_example_var
    and some_example_var not in some_example_var
)
```

Ruff will instead associate the comment with the entire boolean expression, thus preserving the
initial formatting:

```python
(
    # A comment in the middle
    some_example_var and some_example_var not in some_example_var
)
```

### Tuples are parenthesized when expanded

Ruff tends towards parenthesizing tuples (with a few exceptions), while Black tends to remove tuple
parentheses more often.

In particular, Ruff will always insert parentheses around tuples that expand over multiple lines:

```python
# Input
(a, b), (c, d,)

# Black
(a, b), (
    c,
    d,
)

# Ruff
(
    (a, b),
    (
        c,
        d,
    ),
)
```

There's one exception here. In `for` loops, both Ruff and Black will avoid inserting unnecessary
parentheses:

```python
# Input
for a, f(b,) in c:
    pass

# Black
for a, f(
    b,
) in c:
    pass

# Ruff
for a, f(
    b,
) in c:
    pass
```

### Single-element tuples are always parenthesized

Ruff always inserts parentheses around single-element tuples, while Black will omit them in some
cases:

```python
# Input
(a, b),

# Black
(a, b),

# Ruff
((a, b),)
```

Adding parentheses around single-element tuples adds visual distinction and helps avoid "accidental"
tuples created by extraneous trailing commas (see, e.g., [#17181](https://github.com/django/django/pull/17181)).

### Trailing commas are inserted when expanding a function definition with a single argument

When a function definition with a single argument is expanded over multiple lines, Black
will add a trailing comma in some cases, depending on whether the argument includes a type
annotation and/or a default value.

For example, Black will add a trailing comma to the first and second function definitions below,
but not the third:

```python
def func(
    aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa,
) -> None:
    ...


def func(
    aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa=1,
) -> None:
    ...


def func(
    aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa: Argument(
        "network_messages.pickle",
        help="The path of the pickle file that will contain the network messages",
    ) = 1
) -> None:
    ...
```

Ruff will instead insert a trailing comma in all such cases for consistency.

### Parentheses around call-chain assignment values are not preserved

Given:

```python
def update_emission_strength():
    (
        get_rgbw_emission_node_tree(self)
        .nodes["Emission"]
        .inputs["Strength"]
        .default_value
    ) = (self.emission_strength * 2)
```

Black will preserve the parentheses in `(self.emission_strength * 2)`, whereas Ruff will remove
them.

Both Black and Ruff remove such parentheses in simpler assignments, like:

```python
# Input
def update_emission_strength():
    value = (self.emission_strength * 2)

# Black
def update_emission_strength():
    value = self.emission_strength * 2

# Ruff
def update_emission_strength():
    value = self.emission_strength * 2
```

### Call chain calls break differently in some cases

Black occasionally breaks call chains differently than Ruff; in particular, Black occasionally
expands the arguments for the last call in the chain, as in:

```python
# Input
df.drop(
    columns=["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"]
).drop_duplicates().rename(
    columns={
        "a": "a",
    }
).to_csv(path / "aaaaaa.csv", index=False)

# Black
df.drop(
    columns=["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"]
).drop_duplicates().rename(
    columns={
        "a": "a",
    }
).to_csv(
    path / "aaaaaa.csv", index=False
)

# Ruff
df.drop(
    columns=["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"]
).drop_duplicates().rename(
    columns={
        "a": "a",
    }
).to_csv(path / "aaaaaa.csv", index=False)
```

Ruff will only expand the arguments if doing so is necessary to fit within the configured line
width.

Note that Black does not apply this last-call argument breaking universally. For example, both
Black and Ruff will format the following identically:

```python
# Input
df.drop(
    columns=["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"]
).drop_duplicates(a).rename(
    columns={
        "a": "a",
    }
).to_csv(
    path / "aaaaaa.csv", index=False
).other(a)

# Black
df.drop(columns=["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"]).drop_duplicates(a).rename(
    columns={
        "a": "a",
    }
).to_csv(path / "aaaaaa.csv", index=False).other(a)

# Ruff
df.drop(columns=["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"]).drop_duplicates(a).rename(
    columns={
        "a": "a",
    }
).to_csv(path / "aaaaaa.csv", index=False).other(a)
```

Similarly, in some cases, Ruff will collapse composite binary expressions more aggressively than
Black, if doing so allows the expression to fit within the configured line width:

```python
# Black
assert AAAAAAAAAAAAAAAAAAAAAA.bbbbbb.fooo(
    aaaaaaaaaaaa=aaaaaaaaaaaa
).ccccc() == (len(aaaaaaaaaa) + 1) * fooooooooooo * (
    foooooo + 1
) * foooooo * len(
    list(foo(bar(4, foo), foo))
)

# Ruff
assert AAAAAAAAAAAAAAAAAAAAAA.bbbbbb.fooo(
    aaaaaaaaaaaa=aaaaaaaaaaaa
).ccccc() == (len(aaaaaaaaaa) + 1) * fooooooooooo * (
    foooooo + 1
) * foooooo * len(list(foo(bar(4, foo), foo)))
```

### Expressions with (non-pragma) trailing comments are split more often

Both Ruff and Black will break the following expression over multiple lines, since it then allows
the expression to fit within the configured line width:

```python
# Input
some_long_variable_name = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

# Black
some_long_variable_name = (
    "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
)

# Ruff
some_long_variable_name = (
    "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
)
```

However, if the expression ends in a trailing comment, Black will avoid wrapping the expression
in some cases, while Ruff will wrap as long as it allows the expanded lines to fit within the line
length limit:

```python
# Input
some_long_variable_name = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # a trailing comment

# Black
some_long_variable_name = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  # a trailing comment

# Ruff
some_long_variable_name = (
    "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
)  # a trailing comment
```

Doing so leads to fewer overlong lines while retaining the comment's intent. As pragma comments
(like `# noqa` and `# type: ignore`) are ignored when computing line width, this behavior only
applies to non-pragma comments.

### The last context manager in a `with` statement may be collapsed onto a single line

When using a `with` statement with multiple unparenthesized context managers, Ruff may collapse the
last context manager onto a single line, if doing so allows the `with` statement to fit within the
configured line width.

Black, meanwhile, tends to break the last context manager slightly differently, as in the following
example:

```python
# Black
with tempfile.TemporaryDirectory() as d1:
    symlink_path = Path(d1).joinpath("testsymlink")
    with tempfile.TemporaryDirectory(dir=d1) as d2, tempfile.TemporaryDirectory(
        dir=d1
    ) as d4, tempfile.TemporaryDirectory(dir=d2) as d3, tempfile.NamedTemporaryFile(
        dir=d4
    ) as source_file, tempfile.NamedTemporaryFile(
        dir=d3
    ) as lock_file:
        pass

# Ruff
with tempfile.TemporaryDirectory() as d1:
    symlink_path = Path(d1).joinpath("testsymlink")
    with tempfile.TemporaryDirectory(dir=d1) as d2, tempfile.TemporaryDirectory(
        dir=d1
    ) as d4, tempfile.TemporaryDirectory(dir=d2) as d3, tempfile.NamedTemporaryFile(
        dir=d4
    ) as source_file, tempfile.NamedTemporaryFile(dir=d3) as lock_file:
        pass
```

In future versions of Ruff, and in Black's preview style, parentheses will be inserted around the
context managers to allow for clearer breaks across multiple lines, as in:

```python
with tempfile.TemporaryDirectory() as d1:
    symlink_path = Path(d1).joinpath("testsymlink")
    with (
        tempfile.TemporaryDirectory(dir=d1) as d2,
        tempfile.TemporaryDirectory(dir=d1) as d4,
        tempfile.TemporaryDirectory(dir=d2) as d3,
        tempfile.NamedTemporaryFile(dir=d4) as source_file,
        tempfile.NamedTemporaryFile(dir=d3) as lock_file,
    ):
        pass
```
