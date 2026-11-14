# Resilex

**A regular expression engine written in Python, with its own parser, compiler, and matching engines.**

Resilex is a standalone regex engine that behaves like Python's re where the syntax overlaps, plus a few extensions and tools for exploring how regex engines work. The installed runtime never imports re or hands matching off to it.

Regular patterns run on an ordered Thompson NFA, so you get linear time for a fixed pattern and no catastrophic backtracking on patterns like `(a+)+b`. Patterns with backreferences or variable width lookbehind use a separate bounded backtracking matcher instead, and those can run into the work limit.

> Quick heads up: this is for learning, experimenting, and keeping execution bounded. It's not fast, and it's not meant to replace Python's built in re.

## Installation

From the project directory:

```bash
python -m pip install -e .
```

You need Python 3.10 or newer and nothing else at runtime. Import it as resilex. You also get two command line tools: `resilex-visualize` and `resilex-benchmark`.

## Quick start

```python
from resilex import compile

pattern = compile(r"\d+")
match = pattern.search("item-42")

if match:
    print(match.group())  # 42

print(pattern.explain("item-42"))
```

## 200k tests

You need `hypothesis` installed for this one.

```bash
python3 tests/twohundredktests.py <number of workers>
```

## Features

**Syntax:** character classes and escapes (`\d`, `\w`, `\s`, `\b`, `\A`, `\Z`, `\u`, `\N{...}`), Unicode properties like `\p{Script=Greek}`, anchors, capturing, non capturing and named groups, backreferences, alternation, greedy, lazy and possessive quantifiers, atomic groups, conditionals, lookahead and lookbehind, inline flags, and comments.

**Flags:** IGNORECASE (I), MULTILINE (M), DOTALL (S), VERBOSE (X), ASCII (A), UNICODE (U), LOCALE (L), DEBUG and NOFLAG. LOCALE is for bytes patterns and goes through ctypes to the platform's C locale functions. Case insensitive matching handles odd Unicode pairs like İ/i and the Kelvin sign/K, backreferences included.

**API:** it mirrors Python's, with compile, search, match, fullmatch, finditer, findall, sub, subn, split, escape and purge. pos and endpos work, and match objects give you group, groups, groupdict, start, end, span, expand, lastindex, lastgroup and regs, plus `match[1]`. It handles str, bytes, bytearray and memoryview, and compiled patterns can be pickled and copied. Call `.explain(text)` to see the actual search path.

## Limits

Backtracking patterns have no linear guarantee, and you'll see a `RegexMatchLimitError` once the work limit is reached.

`RegexLimits` lets you change the defaults:

| Limit | Default |
|---|---:|
| Pattern length | 65,536 |
| Group nesting depth | 100 |
| Counted repeat bound | 49,999 |
| NFA states | 100,000 |
| DFA states | 10,000 |
| DFA alphabet size | 256 |
| Backtracking steps | 1,000,000 |

Go over a limit and you get a `RegexComplexityError` or `RegexMatchLimitError`.

```python
from resilex import RegexLimits, compile

limits = RegexLimits(backtrack_steps=100_000)

pattern = compile(
    r"(a+)+b",
    limits=limits
)

print(pattern.search("a" * 1000))
```

## Tools

`resilex-visualize` makes Graphviz DOT output for compiled NFAs:

```bash
resilex-visualize '\d+' --output pattern.dot
```

Render it with Graphviz:

```bash
dot -Tsvg pattern.dot -o pattern.svg
```

It can also output a DFA for a finite alphabet:

```bash
resilex-visualize 'a|b' --format dfa --alphabet ab --output pattern.dot
```

DFA conversion supports subset construction, Hopcroft minimization, explicit alphabets, anchors and single character lookarounds. It can't do backreferences or conditionals, since backreferences can describe non regular languages.

`resilex-benchmark` throws difficult patterns like `(a+)+b` at Resilex across different input sizes:

```bash
resilex-benchmark --help
```

## Testing

```bash
python -m pytest -q
```

There are 64 tests, and the last run passed all of them plus 10,282 subtests.

A comparison script checks recorded expected results from CPython v3.10.13, and 403/403 cases passed. To reproduce it:

```bash
curl -fsSLo /tmp/re_tests.py \
  https://raw.githubusercontent.com/python/cpython/v3.10.13/Lib/test/re_tests.py

python tests/check_cpython_conformance.py /tmp/re_tests.py
```

It's a fixture based check, not a live comparison, and it doesn't run the full `test_re.py` suite.

Other checks: tests that confirm re is never imported (`python -S` too), random searches, locale byte tests across all 256 values, and concurrency checks. Tested against Python 3.12.3 so far.

## Clean wheel verification

This builds both distribution formats and tests the installed wheel in a fresh virtual environment (Python 3.10+).

```bash
python3.10 -m pip install build

python3.10 -m build \
  --sdist \
  --wheel \
  --outdir /tmp/resilex-dist

python3.10 -m venv /tmp/resilex-clean-venv

/tmp/resilex-clean-venv/bin/python -m pip install --no-deps \
  /tmp/resilex-dist/resilex-1.0.0-py3-none-any.whl

mkdir -p /tmp/resilex-validation

cp tests/test_regex_engine.py tests/check_cpython_conformance.py \
  /tmp/resilex-validation/

cd /tmp/resilex-validation
```

Check the installed package:

```bash
/tmp/resilex-clean-venv/bin/python -c \
'import sys, resilex; p=resilex.__file__; assert p.startswith(sys.prefix); assert "re" not in sys.modules; assert resilex.compile(r"\p{Script=Greek}+").fullmatch("Αβ"); print(p)'
```

Run the tests:

```bash
/tmp/resilex-clean-venv/bin/python -m unittest -v
```

Run the CPython corpus check:

```bash
curl -fsSLo re_tests.py \
  https://raw.githubusercontent.com/python/cpython/v3.10.13/Lib/test/re_tests.py

/tmp/resilex-clean-venv/bin/python \
  check_cpython_conformance.py re_tests.py
```

Look inside the archives:

```bash
python3.10 -m zipfile -l \
  /tmp/resilex-dist/resilex-1.0.0-py3-none-any.whl

tar -tzf /tmp/resilex-dist/resilex-1.0.0.tar.gz
```

Both should contain LICENSE, `LICENSE-UNICODE.txt`, `NOTICE`, `resilex_unicode_data/LICENSE.txt`, and all three bundled Unicode data files. Results only reflect machines actually tested.

## Limitations

- It follows Python re where the syntax overlaps, but it's not a full replacement.
- Only Unicode 15.0 data is bundled, so newer scripts aren't included.
- Variable width lookbehind is a Resilex extension that Python's re doesn't accept.
- `RegexSyntaxError` doesn't inherit from Python's re.error.
- It's a lot slower than the C based re. On one dev machine, `\d+` over 1 MB took about 1.8 seconds and a case insensitive literal search took 6. Numbers vary by machine, and the default limits may be stricter than Python's re.

## Unicode data attribution

Resilex bundles `Scripts.txt`, `ScriptExtensions.txt` and `PropertyValueAliases.txt` from Unicode Character Database 15.0.0 (https://www.unicode.org/Public/15.0.0/ucd/), Copyright © 2022 Unicode, Inc., under the Unicode License V3. See `LICENSE-UNICODE.txt` and `NOTICE`. Resilex's own code is under the MIT License, see LICENSE.

## Who is this for?

It's good for learning, studying automata, exploring NFA and DFA structures, and running supported untrusted patterns with bounded execution. It's not a good fit for high volume text processing, performance critical production work, or full compatibility with Python's re.

## Development status

Still in active development. The test suite is extensive, but bugs and compatibility differences may still be hiding. More testing, fuzzing and fixes are coming, and contributions, bug reports and feedback are welcome.

## Attribution

Resilex was originally created by Minecrafterify.

You're free to use, modify, distribute, and sell software based on Resilex under the MIT License. This applies to all versions of the project, including earlier commits in the repository's Git history.

If you use or build on Resilex, please credit Minecrafterify as the original creator and link back to the original repo:

https://github.com/Minecrafterify/Resilex

For more on attribution, see `ATTRIBUTION.md`.
