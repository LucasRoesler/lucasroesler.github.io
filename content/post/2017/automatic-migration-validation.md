---
categories:
- programming
comments: false
date: 2017-02-20T00:00:00-07:00
draft: true
keywords:
- python
- django
- automation
- postgres
showPagination: false
summary: How we automate the validation of our Django migrations
tags:
- python
- django
- automation
- postgres
title: "Zero downtime deploys: A tale of Django migrations" 
---

At Teem, we aim for zero down-time deploys; so, the one of the most 
important things we must validate is that things will not break mid-deploy!

The most sensitive step of the deploy process is the changes to our database.
Prior to the automation I am about to describe, validation of the database
migrations required specialized knowledge about Postgres, the changes to the
applicaiton model, load on the database for that model, and a bit of general
experience. This obviously slows down reviews and subsequently deploys. Worst,
it was simply too easy to miss problem migrations when depending on only peer
reviews. To make our lives easier we created a series of validation checks to
ensure that each database migration will be backwards compatible. 

<!--more-->
## The What
The checks I am going to describe are simply a sequence of regex
that we run on the migrations in the changelog. The process looks, roughly,
like this

1. Using git, generate a list of new migrations in this release,
1. Using Django's `sqlmigrate` manage command, generate the SQL for each
   migration,
1. Run a sequence of regex on each SQL command,
1. Report the issues,
1. Profit!

We do this in a python script we internally call `octoeb` which uses
[Click](http://click.pocoo.org/5/) to create a commandline
interface.  So, I can get a changelog along with an audit of the migrations
in my current branch using:

```
$ ocotoeb changelog
```
I won't describe the specific python code, instead I will give you equivalent bash commands
that you can run in your CLI and a simple description of the regex that we
are using. This will give you all the pieces you need to build a similar script
in your favorite language.

## The why
The basic goal is to ensure that any applied migrations are backwards
compatible with the model definitions in the currently deployed release. This
is a requirement because our current deployment process looks like

1. pull the new release code to a single server,
1. run the migrations,
1. restart the application,
1. check the application status,
1. slowly roll the code to the rest of the servers.

As a result, during a deploy we have a mix of old model definitions and new
model definitions running simultaneously.  This means that the database must
except both the old and the new for a short period of time and that any change we make
to the database should not lock up an entire table.

Probably the most common change we often want to make is simply adding a new
column to an existing model.  This can present several issues.  First, your new 
column should not set a default.  In postgres, adding a column with a default will
lock the table while it rewrites the existing rows with the default.  This can
easily be avoided by adding the column first without the default, then adding
the default, followed by a future backfill on the existing rows.  This will
create the column and all future rows will have the default.

Implicit in the above recommendation is that all new columns must be nullable.
You can not add a column without a default unless you allow null. Additionally,
while the old models are running against the new table definitions, the app
will set a value for that column, so it must either have a default or allow
null otherwise Postgres will throw an error.

Finally, the other change that we need to watch for is removing columns. This
is a multi-step process. If you drop a column while the old models are still
active you will have two possible errors (1) when Django tries to select on
that column (which it will because it always explicitly names the columns
selected) or (2) attempting to insert data to a column that doesn't exist
anymore. To actually handle this type of model change you must deploy the model
change prior to running the migration.  In our process, that means you must
commit the model change in a release separate from the database migration.

There are certainly other cases to consider, but we have found these 3 cases to
cover the vast majority of our migration concerns. Having put these checks into
place, we rarely have any issues with database migrations during deploy.

## The how 
### Getting your list of migrations
To find the new migrations you can run the following command

```
$ git log --name-status master.. | grep -e  "^[MA].*migrations.*"
```

Breaking this down, `git log --name-status master..` will print a log of the
commits and the file changes in each commit between `master` and the current
`HEAD`.  The `grep` returns only those lines that start with `A` or `M` and
also contains the work `migrations`.  These are all of the new or modified
migration files.  It will return something like this

```
A	apps/accounts/migrations/0019_auto_20170126_1830.py
```


### Getting your SQL
Once you have the list of migration files that you need to check, we need to
get the actual SQL that is going to be run by Django.  Fortunately, Django
provides a simple command to get this SQL,
[`sqlmigrate`](https://docs.djangoproject.com/en/1.10/ref/django-admin/#sqlmigrate).
For example,

```
django-admin sqlmigrate account 0002
```

will print the sql for the second migration in the `accounts` app. In the
pervious section the ouput contains all of the information that we need.
Specifically, with a command like this

```
$ git log --name-status master.. | grep -e  "^[AM].*migrations.*" | cut -d / -f 2,4 | cut -d . -f 1
```

we would get back exactly the list of apps and the migration name for each
migration that we need to check

```
accounts/0017_auto_20170126_1342
```

This still isn't quite what we need.  At the end of the day the following
command will generate the SQL for you

```
$ git log --name-status master.. | grep -e  "^[AM].*migrations.*" | cut -d / -f 2,4 | cut -d . -f 1 | awk -F"/" '{ print $1,$2}' | xargs -t -L 1 django-admin sqlmigrate
```

We are using python for our scripting, so my script is actually a bit
different, using the regex `apps/((.*)/migrations/(\d+[0-9a-z_]*))\.py` and a
combination of a for loop and subprocess to generate the SQL.

### Regex magic
Now that we have the actual SQL that needs to be tested, it is simply a matter
of running a few regex tests. We have 3 core tests that we run:

- `check_for_not_null` which we test using `/NOT NULL/`
- `check_for_dropped_columns` which we test using `/DROP COLUMN/`,
  and
- `check_for_add_with_default` which we test using `/ADD COLUMN .* DEFAULT/`.

For each migration, we test those 3 regex and alert if they have any matches.
As I mentioned earlier, there are certainly other cases that could be
considered. Let me know if there are some additional checks I should add.

