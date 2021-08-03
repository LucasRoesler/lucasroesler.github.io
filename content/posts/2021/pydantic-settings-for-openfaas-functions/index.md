---
title: "Easy settings management for Python functions with Pydantic"
date: 2021-08-03T00:00:00+02:00
tags:
  - programming
  - openfaas
  - python
  - secrets
  - pydantic
draft: false
---

File this under quick tips and tricks for Python functions in OpenFaas.

I just learned about [Pydantic's](https://pydantic-docs.helpmanual.io/) support for [reading settings from secret files](https://pydantic-docs.helpmanual.io/usage/settings/#secret-support) and it fits perfectly with secrets in OpenFaaS. If you love Python or you are writing Python functions in OpenFaas, this is a great way to simplify your configuration parsing.

<!-- more -->

I am going to keep this short and sweet. Let's say you have an OpenFaas function that needs to access a database. Typically, this requires the `db_host`, `db_user`, and `db_password` to be configured. In OpenFaas, we would expect the `db_host` and `db_user` as environment variables, but the `db_password` is sensitive and should be supplied as a secret, see [the docs for a quick intro to secrets](https://docs.openfaas.com/reference/secrets/).

Your function `stack.yaml` might look like this

```yaml
  functions:
    cool_function:
      lang: python3
      image: functions/cool_function:latest
      env:
        db_host: cool.rds.aws.com
        db_user: super_cool_person
      secrets:
      - db_password
```

If you use `pydantic`, then the only code you need for to configure your function looks like this

```py
from pydantic import BaseSettings
class Settings(BaseSettings):
    db_password: str
    db_host: str
    db_user: str

    class Config:
        secrets_dir = '/var/openfaas/secrets'

settings = Settings()
```

When `Settings` is instantiated, it will automatically read the values from the environment variables _and_ from our secrets, because we set `secrets_dir`. Given our `stack.yaml` above, this means

* `settings.db_host == cool.rds.aws.com`,
* `db_user == super_cool_person`, and
* `db_password` will be the value from `/var/openfaas/secrets/db_password`,

because OpenFaaS puts all of the secrets into `/var/openfaas/secrets`.

That's it, 10 lines of code and you get simple type safe configuration.

## Next steps

I am a big fan of type hinting in Python. Once, you are using Pydantic, then the next step is to configure [`mypy`](http://mypy-lang.org/) to get static type checking before you deploy your functions. But that is a blog post for another day.