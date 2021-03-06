---
title: "Using Tox and PyTest with OpenFaaS"
description: "How to Unit Test Python functions in OpenFaaS"
date: 2021-03-06T00:00:00+02:00
tags:
  - programming
  - openfaas
  - serverless
  - python
  - pytest
  - tox
  - ci/cd
  - github actions
  - testing
  - unit test
draft: false
# <span>Photo by <a href="https://unsplash.com/@alex_andrews?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Alexander Andrews</a> on <a href="https://unsplash.com/?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
coverImage: images/cover2-alexander-andrews.jpg
---

I think it is an uncontroversial statement that to say testing is important in software development. Writing tests may not always be fun, but nothing is a sweet as that moment when a unit test catches a bug before your commit.

In OpenFaaS we have tons of tests in each project, even the [`certifier`](https://github.com/openfaas/certifier) itself that runs a short suite of end-to-end tests. _But_, not all of our function templates have first class testing support. For Go, Node, and Java, this has already been added. For Go it is easy because there is a standard test command `go test`. For Node and Java we (or the community) have picked and optional default test command. Soon, we are going to do the same for Python.

But what do you do in the mean time? Or, what if you have your own Python template and you want to add testing to it?  In this post I am going to show you how to use  [`tox`](https://tox.readthedocs.io/en/latest/) and [`pytest`](https://docs.pytest.org/en/stable/) with any Python OpenFaaS template.

<!-- more -->
## Intro

For this demo we are going to create a tiny calculator function. This gives us just enough code to demonstrate interesting but very common test cases:

1. is the request correct, .e.g did the request specify a valid math operation and two numbers,
2. is the response correct, i.e. the calculation.

Almost all functions will want these kind of tests and, as we will see, they are easy to write.

If you want to skip to the final result, checkout the sample repo here: https://github.com/LucasRoesler/pytest-openfaas-sample

### Start the project
```sh
$ mkdir pytest-sample
$ cd pytest-sanmple
$ faas-cli template store pull python3-flask
$ mv calc.yml stack.yml
```

**Note** Don't forget to add this section to your stack file so that the `faas-cli` will know how to automatically pull the template

```yaml
configuration:
  templates:
    - name: python3-flask
      source: https://github.com/openfaas/python-flask-template

```

### Setup the local python environment
I use [conda](https://docs.conda.io/projects/conda/en/latest/) for my local virtual environments, but you can of course use [`virtualenv`](https://virtualenv.readthedocs.org/en/latest/) or [`venv`](https://docs.python.org/3/library/venv.html) (see also [RealPython's primer on virtual environments](https://realpython.com/python-virtual-environments-a-primer/)).

In short this creates an isolated and repeatable development environment which you can delete and recreate if you need.

```sh
$ conda create -n pytest-sample tox
$ conda activate pytest-sample
$ cat <<EOF >> requirements.txt
pydantic==1.7.3
flask==1.1.2
EOF
$ cat <<EOF >> dev.txt
tox
pytest
black
pylint
EOF
$ conda install --yes --file requirements.txt
$ conda install --yes --file dev.txt
```

Now we are ready to develop the calculator.

### Add the first tests
Let's start with some simple tests:

```sh
$ touch calc/handler_test.py
```

then add the following test cases for a couple of our happy paths

```py
import calc.handler as h


class TestParsing:
    def test_operation_addition(self):
        req = '{"op": "+", "var1": "1.0", "var2": 0}'
        resp, code = h.handle(req)
        assert code == 200
        assert resp["value"] == 1.0

    def test_operation_multiplication(self):
        req = '{"op": "*", "var1": "100.01", "var2": 1}'
        resp, code = h.handle(req)
        assert code == 200
        assert resp["value"] == 100.01
```

At this point, we have followed normal conventions for pytest, by default, pytest will look for files that match the pattern `*_test.py` and then the run the functions that  match `test_*` (including matching methods if the class is named `Test*`, which is what we did above).

Now, we can run our test and see all of the errors in their wonderful glory

```sh
$ pytest
========================================= test session starts =========================================
platform linux -- Python 3.8.8, pytest-6.2.2, py-1.10.0, pluggy-0.13.1
rootdir: /home/lucas/code/openfaas/sandbox/pytest-sample
collected 2 items

calc/handler_test.py FF                                                                         [100%]

============================================== FAILURES ===============================================
_________________________________ TestParsing.test_operation_addition _________________________________

self = <calc.handler_test.TestParsing object at 0x7fc8029ad820>

    def test_operation_addition(self):
        req = '{"op": "+", "var1": "1.0", "var2": 0}'
>       resp, code = h.handle(req)
E       ValueError: too many values to unpack (expected 2)

calc/handler_test.py:49: ValueError
______________________________ TestParsing.test_operation_multiplication ______________________________

self = <calc.handler_test.TestParsing object at 0x7fc8029add30>

    def test_operation_multiplication(self):
        req = '{"op": "^", "var1": "2", "var2": -2}'
>       resp, code = h.handle(req)
E       ValueError: too many values to unpack (expected 2)

calc/handler_test.py:73: ValueError
======================================= short test summary info =======================================
FAILED calc/handler_test.py::TestParsing::test_operation_addition - ValueError: too many values to u...
FAILED calc/handler_test.py::TestParsing::test_operation_multiplication - ValueError: too many value...
========================================== 2 failed in 0.05s ==========================================
```

In this case we get a value error because our handler function only returns a string, not a dictionary and a status code like the test expects.  These test cases  are expecting that the handler returns something that looks like this

```py
return {"value": 1.1}, 200
```

This return value works with Flask, it will json serialize the dictionary and set the status code to 200 for us. Like the above snippet and run pytest again, the we get a new error, a value error because we got the wrong calculation result

```sh
$ pytest
========================================= test session starts =========================================
platform linux -- Python 3.8.8, pytest-6.2.2, py-1.10.0, pluggy-0.13.1
rootdir: /home/lucas/code/openfaas/sandbox/pytest-sample
collected 2 items

calc/handler_test.py .F                                                                         [100%]

============================================== FAILURES ===============================================
______________________________ TestParsing.test_operation_multiplication ______________________________

self = <calc.handler_test.TestParsing object at 0x7f9f30ecbd30>

    def test_operation_multiplication(self):
        req = '{"op": "^", "var1": "2", "var2": -2}'
        resp, code = h.handle(req)
        assert code == 200
>       assert resp["value"] == 0.25
E       assert 1 == 0.25

calc/handler_test.py:75: AssertionError
======================================= short test summary info =======================================
FAILED calc/handler_test.py::TestParsing::test_operation_multiplication - assert 1 == 0.25
===================================== 1 failed, 1 passed in 0.06s =====================================
```

### Adding `tox`

Before we fix the test, we want to setup one more thing: [`tox`](https://tox.readthedocs.io/en/latest/index.html): `tox`provides a unified set of tooling that will run `pytest` for us. This provides a standardized interface that we will use later in CI. In fact, it ensures we are running exactly the same thing in CI and our local dev environment, hopefully reducing the numer of "Actually fix CI" commit messages.  Later, if you decide that you prefer `nose` or some other testing tool or if you application is even more complex and requires pre-test configuration -- we can manage that through `tox`.

Create a `calc/tox.ini` file that looks like

```ini
# calc/tox.ini
[tox]
# list the testenv to run by default
envlist = lint,test
skipsdist = true

[testenv:test]
deps =
  pytest
  -rrequirements.txt
commands =
  # run unit tests
  pytest

[testenv:lint]
deps =
  flake8
commands =
  # stop the build if there are Python syntax errors or undefined names
  flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics --extend-exclude template
  # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
  flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics --extend-exclude template
```

This configures `tox` to create two environments and run our tests and flake8.  Now we can test and lint our `calc` function using

```sh
cd calc
tox
```
or just run the tests using

```sh
$ cd calc
$ tox -q -e test
======================================== test session starts ========================================
platform linux -- Python 3.8.8, pytest-6.2.2, py-1.10.0, pluggy-0.13.1
cachedir: .tox/test/.pytest_cache
rootdir: /home/lucas/code/openfaas/sandbox/pytest-sample/calc
collected 2 items

handler_test.py FF                                                                            [100%]

============================================= FAILURES ==============================================
________________________________ TestParsing.test_operation_addition ________________________________

self = <calc.handler_test.TestParsing object at 0x7ff1aa19a550>

    def test_operation_addition(self):
        req = '{"op": "+", "var1": "1.0", "var2": 0}'
>       resp, code = h.handle(req)
E       ValueError: too many values to unpack (expected 2)

handler_test.py:6: ValueError
_____________________________ TestParsing.test_operation_multiplication _____________________________

self = <calc.handler_test.TestParsing object at 0x7ff1aa19afd0>

    def test_operation_multiplication(self):
        req = '{"op": "*", "var1": "100.01", "var2": 1}'
>       resp, code = h.handle(req)
E       ValueError: too many values to unpack (expected 2)

handler_test.py:12: ValueError
====================================== short test summary info ======================================
FAILED handler_test.py::TestParsing::test_operation_addition - ValueError: too many values to unpa...
FAILED handler_test.py::TestParsing::test_operation_multiplication - ValueError: too many values t...
========================================= 2 failed in 0.04s =========================================
ERROR: InvocationError for command /home/lucas/code/openfaas/sandbox/pytest-sample/calc/.tox/test/bin/pytest (exited with code 1)
______________________________________________ summary ______________________________________________
ERROR:   test: commands failed
```

### The working implementation
Jumping forward a bit, here is the final implementation. I have used the `enum` package and `pydantic` to help simplify the validation and parsing of requests into my internal `Calculation` model

```py
from pydantic import BaseModel, ValidationError
from enum import Enum, unique
from typing import Callable


@unique
class OperationType(Enum):
    ADD = "+"
    SUBTRACT = "-"
    MULTIPLY = "*"
    DIVIDE = "/"
    POWER = "^"


class Caclucation(BaseModel):
    op: OperationType
    var1: float
    var2: float

    def execute(self) -> float:
        if self.op is OperationType.ADD:
            return self.var1 + self.var2

        if self.op is OperationType.SUBTRACT:
            return self.var1 - self.var2

        if self.op is OperationType.MULTIPLY:
            return self.var1 * self.var2

        if self.op is OperationType.DIVIDE:
            return self.var1 / self.var2

        if self.op is OperationType.POWER:
            return self.var1 ** self.var2

        raise ValueError("unknown operation")



def handle(req) -> (dict, int):
    """handle a request to the function
    Args:
        req (str): request body
    """

    try:
        c = Caclucation.parse_raw(req)
    except ValidationError as e:
        return {"message": e.errors()}, 422
    except Exception as e:
        return {"message": e}, 500

    return {"value": c.execute()}, 200
```

Now our tests will pass

```sh
$ tox -q -e test
======================================== test session starts ========================================
platform linux -- Python 3.8.8, pytest-6.2.2, py-1.10.0, pluggy-0.13.1
cachedir: .tox/test/.pytest_cache
rootdir: /home/lucas/code/openfaas/sandbox/pytest-sample/calc
collected 2 items

calc/handler_test.py ..                                                                         [100%]

========================================== 2 passed in 0.04s ==========================================
```


### Testing error cases
We can even extend our test cases to cover validation errors. Because we using `pydantic` and we are returning `{"message": e.errors()}`, these tests look like this

```py
def test_operation_parsing_error_on_empty_obj(self):
    req = '{}'

    resp, code = h.handle(req)
    assert code == 422
    # should be a list of error
    errors = resp.get("message", [])
    assert len(errors) == 3
    assert errors[0].get("loc") == ('op', )
    assert errors[0].get("msg") == "field required"

    assert errors[1].get("loc") == ('var1', )
    assert errors[1].get("msg") == "field required"

    assert errors[2].get("loc") == ('var2', )
    assert errors[2].get("msg") == "field required"

def test_operation_parsing_error_on_unknown_operation(self):
    req = '{"op": "foo", "var1": "1.0", "var2": 0}'
    resp, code = h.handle(req)
    assert code == 422

    # should be a list of error
    errors = resp.get("message", [])
    assert len(errors) == 1
    assert errors[0].get("loc") == ('op', )
    assert errors[0].get("msg") == (
        "value is not a valid enumeration member; permitted: "
        "'+', '-', '*', '/', '^'"
    )
```

Checkout the repo for the other example tests https://github.com/LucasRoesler/pytest-openfaas-sample/blob/main/calc/handler_test.py

### Running this in CI

I've added a Github Action Workflow to my sample repo. It is based on the sample testing workflow that [Github provides for Python](https://docs.github.com/en/actions/guides/building-and-testing-python#starting-with-the-python-workflow-template) and the function workflow from [Serverless for Everyone](https://gumroad.com/l/serverless-for-everyone-else)

This workflow will test and build your function in parallel. The resulting Docker image is pushed to Docker Container Registry.

**NOTE** you need to configure the `DOCKER_PASSWORD` secret for your Github repo for this workflow to work correctly.


## Next steps

This is just a tiny step to more stable functions. Over the next couple weeks we are going to look into how to build the `tox` runner into the Python functions. Which means, you can just add a `tox.ini` to each function and `faas-cli` will run any tests you can configured  and then build the function. If you have more examples of interesting `tox` configurations, let me know on [Twitter](https://twitter.com/theaxer) or stop by the [OpenFaas Slack](https://docs.openfaas.com/community/#slack-workspace) and share it with the community.