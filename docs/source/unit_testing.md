# Unit Testing with pytest

Writing tests might feel like extra work on top of the code you actually want to write. In practice it is the opposite: tests let you change code confidently, catch bugs before they reach others, and document what your code is supposed to do. This page introduces **pytest**, the testing framework used in this project.

## What is a Unit Test?

A unit test checks that a single, isolated piece of code — a function or a class method — does what it is supposed to do. You give it known inputs and assert that the output matches what you expect.

```python
def add(a: int, b: int) -> int:
    return a + b

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
```

If `add` ever breaks, `test_add` will fail and tell you exactly where the problem is before it causes something harder to debug downstream.

## Installing pytest

pytest is already in the project's dev dependencies. After running `uv sync`, it is available immediately:

```bash
uv run pytest tests/
```

## Writing Tests

### File and function naming

pytest discovers tests automatically based on naming conventions:

- Test files must be named `test_*.py` or `*_test.py`
- Test functions must start with `test_`
- Test classes must start with `Test`

The project already has the right structure:

```
tests/
    __init__.py
    test_data.py     <- tests for src/ema_beers/data.py
    test_model.py    <- tests for src/ema_beers/model.py
    test_api.py      <- tests for src/ema_beers/api.py
```

### A basic test

```python
# tests/test_data.py
from ema_beers.data import load_dataset

def test_load_dataset_returns_dataframe():
    df = load_dataset("data/raw/train.csv")
    assert df is not None
    assert len(df) > 0

def test_load_dataset_has_expected_columns():
    df = load_dataset("data/raw/train.csv")
    assert "label" in df.columns
    assert "feature_1" in df.columns
```

### Testing for exceptions

Sometimes the correct behaviour is to raise an error. Use `pytest.raises` to assert that:

```python
import pytest
from ema_beers.data import load_dataset

def test_load_dataset_raises_on_missing_file():
    with pytest.raises(FileNotFoundError):
        load_dataset("data/raw/does_not_exist.csv")
```

### Parametrize: testing multiple cases cleanly

Instead of writing one test function per input combination, use `@pytest.mark.parametrize`:

```python
import pytest
from ema_beers.model import normalize

@pytest.mark.parametrize("value, expected", [
    (0.0, 0.0),
    (1.0, 1.0),
    (0.5, 0.5),
    (-1.0, 0.0),   # clamped to valid range
])
def test_normalize(value, expected):
    assert normalize(value) == expected
```

This runs four separate tests from a single function, each clearly labelled in the output.

### Fixtures: reusable setup

If multiple tests need the same setup — loading a dataset, creating a model — use a **fixture** instead of repeating the setup in each test:

```python
import pytest
import pandas as pd
from ema_beers.model import Model

@pytest.fixture
def sample_data() -> pd.DataFrame:
    return pd.DataFrame({"feature_1": [1.0, 2.0, 3.0], "label": [0, 1, 0]})

@pytest.fixture
def trained_model(sample_data: pd.DataFrame) -> Model:
    model = Model()
    model.fit(sample_data)
    return model

def test_model_predicts_correct_shape(trained_model, sample_data):
    predictions = trained_model.predict(sample_data)
    assert len(predictions) == len(sample_data)

def test_model_output_is_binary(trained_model, sample_data):
    predictions = trained_model.predict(sample_data)
    assert set(predictions).issubset({0, 1})
```

Fixtures can also be placed in a `conftest.py` file in the `tests/` directory, which makes them automatically available to all test files without importing.

## Running Tests

Run all tests:

```bash
uv run pytest tests/
```

Run a specific file:

```bash
uv run pytest tests/test_model.py
```

Run a specific test function:

```bash
uv run pytest tests/test_model.py::test_model_predicts_correct_shape
```

Show detailed output with `-v`:

```bash
uv run pytest tests/ -v
```

Stop on the first failure with `-x`:

```bash
uv run pytest tests/ -x
```

## Test Coverage

**Coverage** measures which lines of your code are actually executed by your tests. The project uses the `coverage` package, which is already in the dev dependencies.

Run tests with coverage:

```bash
uv run coverage run -m pytest tests/
uv run coverage report
```

This prints a summary showing which files and lines are not covered:

```
Name                        Stmts   Miss  Cover
-----------------------------------------------
src/ema_beers/data.py          24      4    83%
src/ema_beers/model.py         31      2    94%
src/ema_beers/api.py           18      8    56%
-----------------------------------------------
TOTAL                          73     14    81%
```

For an HTML report you can browse in detail:

```bash
uv run coverage html
```

This creates a `htmlcov/` directory. Open `htmlcov/index.html` in a browser to see exactly which lines are hit and which are missed.

!!! note
    The project's `pyproject.toml` already configures coverage to exclude the tests directory itself:
    ```toml
    [tool.coverage.run]
    omit = ["tests/*"]
    ```

## Exercise

!!! example "Exercise"
    1. Open `tests/test_data.py` and write a test for a function in `src/ema_beers/data.py`. If the function doesn't exist yet, write a simple one (e.g. a function that reads a CSV and returns a DataFrame).

    2. Run your test and confirm it passes:

        ```bash
        uv run pytest tests/test_data.py -v
        ```

    3. Deliberately break the function and run the test again — observe how pytest reports the failure.

    4. Fix the function, then add a second test using `@pytest.mark.parametrize` to check multiple inputs.

    5. Run the full test suite with coverage and check the report:

        ```bash
        uv run coverage run -m pytest tests/
        uv run coverage report
        ```

    6. Identify an uncovered line and write a test that covers it.
