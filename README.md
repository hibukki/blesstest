# blesstest

A gold (snapshot) testing framework for Python based on pytest.

TL;DR: Define scenarios that exercise your logic, run them with blessed, then look at the outputs and "bless" them. Next time the tests run, if the result differs from blessed output, it will fail and you will need to re-bless it.

## Installation

```bash
pip install git+https://github.com/SonOfLilit/blesstest@main
```

## Basic usage

### Define a function to test

```python
def add(a: int, b: int) -> int:
    return a + b
```

### Define the harness

In `conftest.py`:

```python
from blesstest import harness, pytest_collect_file # noqa

import pydantic
from .functions import add

class AddInput(pydantic.BaseModel):
    a: int
    b: int

class AddOutput(pydantic.BaseModel):
    result: int

@harness
def addition_harness(test_input: AddInput) -> AddOutput:
    result = add(test_input.a, test_input.b)
    return AddOutput(result=result)
```

### Create a test definition file

e.g., `tests/test_addition.blesstest.jsonc`:

```jsonc
{
  "add_simple": {
    // This will test the case `sum(a=1, b=2)`
    "harness": "addition_harness",
    "params": {
      "a": 1,
      "b": 2
    }
  },
  "add_large_numbers": {
    "harness": "addition_harness",
    "params": {
      "a": 1000000,
      "b": 2000000
    }
  },
  "with_inheritance": {
    "base": "add_simple",
    "params": {
      "b": 5
    }
  },
  "with_variations": {
    // This will run 2 tests: `sum(a=10, b=1)` and `sum(a=10, b=2)`
    "harness": "addition_harness",
    "params": {
      "a": 10
    },
    "variations": [
      {
        "params": {
          "b": 1
        }
      },
      {
        "params": {
          "b": 2
        }
      }
    ]
  }
}
```

### Run Tests

```bash
pytest
```

### Check the results

#### Check what files changed

```bash
$ git status blessed/
[..]
Untracked files:
(use "git add <file>..." to include in what will be committed)
        blessed/add_simple.json
        blessed/add_large_numbers.json
        blessed/with_inheritance.json
        blessed/with_variations__b_1.json
        blessed/with_variations__b_2.json
```

#### Do the changes look good?

```bash
$ cat blessed/add_simple.json
```

```
{
  "harness": "my_harness",
  "params": {
    "a": 1,
    "b": 2
  },
  "result": {
    "result": 3
  }
}
```

#### Add the golden (snapshot) files to git

```bash
$ git add blessed/add_simple.json
```

Congrats, you've blessed your first test!

Now `pytest` will show you that the test passed. If the results will ever differ from those snapshots, the tests will fail again.
