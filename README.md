[![Build Status](https://github.com/hukkin/tomli/actions/workflows/tests.yaml/badge.svg?branch=master)](https://github.com/hukkin/tomli/actions?query=workflow%3ATests+branch%3Amaster+event%3Apush)
[![codecov.io](https://codecov.io/gh/hukkin/tomli/branch/master/graph/badge.svg)](https://codecov.io/gh/hukkin/tomli)
[![PyPI version](https://img.shields.io/pypi/v/tomli)](https://pypi.org/project/tomli)

# Tomli

> A lil' TOML parser

**Table of Contents** *generated with [mdformat-toc](https://github.com/hukkin/mdformat-toc)*

<!-- mdformat-toc start --slug=github --maxlevel=6 --minlevel=2 -->

- [Intro](#intro)
- [Installation](#installation)
- [Usage](#usage)
  - [Parse a TOML string](#parse-a-toml-string)
  - [Parse a TOML file](#parse-a-toml-file)
  - [Handle invalid TOML](#handle-invalid-toml)
  - [Construct `decimal.Decimal`s from TOML floats](#construct-decimaldecimals-from-toml-floats)
  - [Building a `tomli`/`tomllib` compatibility layer](#building-a-tomlitomllib-compatibility-layer)
- [FAQ](#faq)
  - [Why this parser?](#why-this-parser)
  - [Is comment preserving round-trip parsing supported?](#is-comment-preserving-round-trip-parsing-supported)
  - [Is there a `dumps`, `write` or `encode` function?](#is-there-a-dumps-write-or-encode-function)
  - [How do TOML types map into Python types?](#how-do-toml-types-map-into-python-types)
- [Performance](#performance)
  - [Pure Python](#pure-python)
  - [Mypyc generated wheel](#mypyc-generated-wheel)

<!-- mdformat-toc end -->

## Intro<a name="intro"></a>

Tomli is a Python library for parsing [TOML](https://toml.io).
It is fully compatible with [TOML v1.0.0](https://toml.io/en/v1.0.0).

A version of Tomli, the `tomllib` module,
was added to the standard library in Python 3.11
via [PEP 680](https://www.python.org/dev/peps/pep-0680/).
Tomli continues to provide a backport on PyPI for Python versions
where the standard library module is not available
and that have not yet reached their end-of-life.

Tomli uses [mypyc](https://github.com/mypyc/mypyc)
to generate binary wheels for most of the widely used platforms,
so Python 3.11+ users may prefer it over `tomllib` for improved performance.
Pure Python wheels are available on any platform and should perform the same as `tomllib`.

## Installation<a name="installation"></a>

```bash
pip install tomli
```

## Usage<a name="usage"></a>

### Parse a TOML string<a name="parse-a-toml-string"></a>

```python
import tomli

toml_str = """
[[players]]
name = "Lehtinen"
number = 26

[[players]]
name = "Numminen"
number = 27
"""

toml_dict = tomli.loads(toml_str)
assert toml_dict == {
    "players": [{"name": "Lehtinen", "number": 26}, {"name": "Numminen", "number": 27}]
}
```

### Parse a TOML file<a name="parse-a-toml-file"></a>

```python
import tomli

with open("path_to_file/conf.toml", "rb") as f:
    toml_dict = tomli.load(f)
```

The file must be opened in binary mode (with the `"rb"` flag).
Binary mode will enforce decoding the file as UTF-8 with universal newlines disabled,
both of which are required to correctly parse TOML.

### Handle invalid TOML<a name="handle-invalid-toml"></a>

```python
import tomli

try:
    toml_dict = tomli.loads("]] this is invalid TOML [[")
except tomli.TOMLDecodeError:
    print("Yep, definitely not valid.")
```

Note that error messages are considered informational only.
They should not be assumed to stay constant across Tomli versions.

### Construct `decimal.Decimal`s from TOML floats<a name="construct-decimaldecimals-from-toml-floats"></a>

```python
from decimal import Decimal
import tomli

toml_dict = tomli.loads("precision-matters = 0.982492", parse_float=Decimal)
assert isinstance(toml_dict["precision-matters"], Decimal)
assert toml_dict["precision-matters"] == Decimal("0.982492")
```

Note that `decimal.Decimal` can be replaced with another callable that converts a TOML float from string to a Python type.
The `decimal.Decimal` is, however, a practical choice for use cases where float inaccuracies can not be tolerated.

Illegal types are `dict` and `list`, and their subtypes.
A `ValueError` will be raised if `parse_float` produces illegal types.

### Building a `tomli`/`tomllib` compatibility layer<a name="building-a-tomlitomllib-compatibility-layer"></a>

Python versions 3.11+ ship with a version of Tomli:
the `tomllib` standard library module.
To build code that uses the standard library if available,
but still works seamlessly with Python 3.6+,
do the following.

Instead of a hard Tomli dependency, use the following
[dependency specifier](https://packaging.python.org/en/latest/specifications/dependency-specifiers/)
to only require Tomli when the standard library module is not available:

```
tomli >= 1.1.0 ; python_version < "3.11"
```

Then, in your code, import a TOML parser using the following fallback mechanism:

```python
import sys

if sys.version_info >= (3, 11):
    import tomllib
else:
    import tomli as tomllib

tomllib.loads("['This parses fine with Python 3.6+']")
```

## FAQ<a name="faq"></a>

### Why this parser?<a name="why-this-parser"></a>

- it's lil'
- pure Python with zero dependencies
- the fastest pure Python parser [\*](#pure-python):
  18x as fast as [tomlkit](https://pypi.org/project/tomlkit/),
  2.1x as fast as [toml](https://pypi.org/project/toml/)
- outputs [basic data types](#how-do-toml-types-map-into-python-types) only
- 100% spec compliant: passes all tests in
  [BurntSushi/toml-test](https://github.com/BurntSushi/toml-test)
  test suite
- thoroughly tested: 100% branch coverage

### Is comment preserving round-trip parsing supported?<a name="is-comment-preserving-round-trip-parsing-supported"></a>

No.

The `tomli.loads` function returns a plain `dict` that is populated with builtin types and types from the standard library only.
Preserving comments requires a custom type to be returned so will not be supported,
at least not by the `tomli.loads` and `tomli.load` functions.

Look into [TOML Kit](https://github.com/sdispater/tomlkit) if preservation of style is what you need.

### Is there a `dumps`, `write` or `encode` function?<a name="is-there-a-dumps-write-or-encode-function"></a>

[Tomli-W](https://github.com/hukkin/tomli-w) is the write-only counterpart of Tomli, providing `dump` and `dumps` functions.

The core library does not include write capability, as most TOML use cases are read-only, and Tomli intends to be minimal.

### How do TOML types map into Python types?<a name="how-do-toml-types-map-into-python-types"></a>

| TOML type        | Python type         | Details                                                      |
| ---------------- | ------------------- | ------------------------------------------------------------ |
| Document Root    | `dict`              |                                                              |
| Key              | `str`               |                                                              |
| String           | `str`               |                                                              |
| Integer          | `int`               |                                                              |
| Float            | `float`             |                                                              |
| Boolean          | `bool`              |                                                              |
| Offset Date-Time | `datetime.datetime` | `tzinfo` attribute set to an instance of `datetime.timezone` |
| Local Date-Time  | `datetime.datetime` | `tzinfo` attribute set to `None`                             |
| Local Date       | `datetime.date`     |                                                              |
| Local Time       | `datetime.time`     |                                                              |
| Array            | `list`              |                                                              |
| Table            | `dict`              |                                                              |
| Inline Table     | `dict`              |                                                              |

## Performance<a name="performance"></a>

The `benchmark/` folder in this repository contains a performance benchmark for comparing the various Python TOML parsers.

Below are the results for commit [0724e2a](https://github.com/hukkin/tomli/tree/0724e2ab1858da7f5e05a9bffdb24c33589d951c).

### Pure Python<a name="pure-python"></a>

```console
foo@bar:~/dev/tomli$ python --version
Python 3.12.7
foo@bar:~/dev/tomli$ pip freeze
attrs==21.4.0
click==8.1.7
pytomlpp==1.0.13
qtoml==0.3.1
rtoml==0.11.0
toml==0.10.2
tomli @ file:///home/foo/dev/tomli
tomlkit==0.13.2
foo@bar:~/dev/tomli$ python benchmark/run.py
Parsing data.toml 5000 times:
------------------------------------------------------
    parser |  exec time | performance (more is better)
-----------+------------+-----------------------------
     rtoml |    0.647 s | baseline (100%)
  pytomlpp |    0.891 s | 72.62%
     tomli |     3.14 s | 20.56%
      toml |     6.69 s | 9.67%
     qtoml |     8.27 s | 7.82%
   tomlkit |     56.1 s | 1.15%
```

### Mypyc generated wheel<a name="mypyc-generated-wheel"></a>

```console
foo@bar:~/dev/tomli$ python benchmark/run.py
Parsing data.toml 5000 times:
------------------------------------------------------
    parser |  exec time | performance (more is better)
-----------+------------+-----------------------------
     rtoml |    0.668 s | baseline (100%)
  pytomlpp |    0.893 s | 74.81%
     tomli |     1.96 s | 34.18%
      toml |     6.64 s | 10.07%
     qtoml |     8.26 s | 8.09%
   tomlkit |     52.9 s | 1.26%
```
