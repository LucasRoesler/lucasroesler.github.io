---
title: "Python in OpenFaaS with submodules"
summary: "Tidy python in an OpenFaaS world: or, what happens when you want to split your Python function into multiple modules?"
permalink: "https://google.com/test"
date: 2219-01-20T00:00:00+02:00
categories:
  - programming
  - openfaas
  - serverless
  - http
  - examples
tags:
  - openfaas
  - python
  - serverless
  - examples
keywords:
  - openfaas
  - python
  - serverless
comments: false
showPagination: false
autoThumbnailImage: true
draft: true
---

As a core contributor to [OpenFaaS][openfaas-hopepage], I hangout in the OpenFaaS Slack a lot, you can [join us here][openfaas-slack-singnup]. Most recently that meant trying to answer this question

> [How can I] write a function in python that uses multiple source files so that the main `handler.py` is able to import helper functions from separate source files.

Today, I want to walk though creating exactly this kind of function by creating a word count function.

<!--more-->

**TL;DR**: Use explicit relative imports to split your python code into several files. If you want to quickly jump to the finished code, I have create a [public repo here][project-repo].

I will assume some familiarity with OpenFaaS and that you may have even started/completed the [OpenFaaS Workshop][workshop-repo], so I won't go into a lot of detail about the specific OpenFaaS setup instructions below. Instead, I want to focus on how to comfortably develop a Python function as it goes from a small "one-liner" function to a larger multi-module project.

## Setting up a python environment

This project uses Python 3 and I like to use Conda for my development environments:

```sh
$ conda create --name faaswordcount  python=3.7.2 black autopep8 flake8 pylint mypy flask gevent
$ conda source activate faaswordcount
```

Or from my example repo: `conda env create -f environment.yml`.

Creating a local environment like this will allow your editor/ide to lint/hint/autocomplete as needed.

## Starting the function

Next, to start a new Python 3 function, I like to use the flask template because I am going to load a static list of "stop words" into memory when the function starts. Using the flask templates allows the function to do this only once instead of on each invocation.

```sh
$ faas-cli template store pull python3-flask
Fetch templates from repository: https://github.com/openfaas-incubator/python-flask-template at master
2019/01/19 10:55:34 Attempting to expand templates from https://github.com/openfaas-incubator/python-flask-template
2019/01/19 10:55:35 Fetched 3 template(s) : [python27-flask python3-flask python3-flask-armhf] from https://github.com/openfaas-incubator/python-flask-template
$ faas-cli new wordcount --lang python3-flask
```

The project should now look like

```txt
.
├── stack.yml
├── template
│   ├── python27-flask
│   │   ├ ...
│   ├── python3-flask
│   │   ├ ...
│   └── python3-flask-armhf
│       ├ ...
└── wordcount
    ├── __init__.py
    ├── handler.py
    └── requirements.txt
```

## The word count implementation using relative imports

We are going to add two more files to our function:

```sh
touch wordcount/stopwords
touch wordcount/wordcount.py
```

The project now looks like

```txt
.
├── stack.yml
├── template
│   ├── ...
└── wordcount
    ├── __init__.py
    ├── handler.py
    ├── requirements.txt
    ├── stopwords
    └── wordcount.py
```

The `stopwords` is a plain text file of words that will be excluded in the wordcount. These are short common words that you would not want to include in a wordcount visualization such as: `a`, `an`, `him`, `her`, and so on. This list of words will depend on your use case, adjust as you see fit.

All of the real fun happens in `wordcount.py`:

```py
# modified wordcount.py
import unicodedata
import os
from typing import Dict, List

from operator import itemgetter
from collections import defaultdict

FILE = os.path.dirname(__file__)
STOPWORDS = set(
    map(str.lower,
        map(str.strip,
            open(os.path.join(FILE, 'stopwords'), 'r').readlines()
            )
        )
)


def process_text(text: bytes) -> Dict[str, int]:
    """Splits a long text into words a count of interesting words in the text.

    process_text will eliminate any of the stopwords, punctuation, and normalize
    the text to merge cases and plurals into a single value.
    """
    words = text.decode("utf-8").split()
    # remove stopwords
    # remove 's
    words = [
        word[:-2] if word.lower().endswith(("'s",))
        else word
        for word in words
    ]
    # remove numbers
    words = [word for word in words if not word.isdigit()]
    words = [strip_punctuation(word) for word in words]
    words = [word for word in words if word.lower() not in STOPWORDS]

    return process_tokens(words)

def process_tokens(words: List[str]) -> Dict[str, int]:
    """Normalize cases and remove plurals.
    """
    # ...

def strip_punctuation(text: str) -> str:
    # ...
```

Note that processing the `STOPWORDS` at the start of the file means it will be loaded into memory once the package is imported. In this case, once OpenFaaS deploys the function.

Using an explicit relative import, we can very easily use the `process_text` method to create a very simple `handler.py`:

```py
# handler.py
import json
from .wordcount import process_text


def handle(req):
    """handle a request to the function
    Args:
        req (str): request body
    """

    return json.dumps(process_text(req))
```

This style of relative imports will work for any file or sub-package you include inside of your function folder. Additionally, your IDE and linter will be able to resolve the imported code correctly!

## Setup OpenFaaS

OpenFaaS on Kubeneretes is my go-to deployment because I can use the built in k8s cluster in Docker-for-Mac.

```sh
$ kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
namespace "openfaas" created
namespace "openfaas-fn" created

$ helm repo add openfaas https://openfaas.github.io/faas-netes/ && helm repo update
"openfaas" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "contiamo" chart repository
...Successfully got an update from the "openfaas" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈

$ helm upgrade openfaas --install openfaas/openfaas \
    --namespace openfaas  \
    --set basic_auth=false \
    --set faasnetesd.imagePullPolicy='IfNotPresent' \
    --set functionNamespace=openfaas-fn
```

This gives me an OpenFaaS installation with auth disabled and it will use my locally built images for my functions.

## Deploy and test the function

```sh
$ faas-cli up -f wordcount.yml --skip-push
# Docker build output ...
Deploying: wordcount.

Deployed. 202 Accepted.
URL: http://127.0.0.1:31112/function/wordcount

$ echo 'This is some example text that we want to see a frequency response for.  It has text like apple, apples, apple tree, etc' | faas-cli -f wordcount.yml invoke wordcount
{"example": 1, "text": 2, "want": 1, "see": 1, "frequency": 1, "response": 1, "apple": 3, "tree": 1, "etc": 1}
```

## Wrapping up

Using relative imports allows the creation of Python functions that are split between several files. We could take this further and import from sub-folders or sub-folders of sub-folders even. This has the added benefit that the code is valid in both your local environment and the final docker container. Try the [completed code example in this repo.][project-repo]

[openfaas-homepage]: https://openfaas.com
[openfaas-slack-singnup]: https://docs.openfaas.com/community/#slack-workspace
[project-repo]: https://github.com/LucasRoesler/openfaas-multifile-example
[workshop-repo]: https://github.com/openfaas/workshop
