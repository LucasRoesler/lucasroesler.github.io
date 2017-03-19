---
categories:
- programming
comments: false
date: 2017-03-19T00:00:00-06:00
keywords:
- git
showPagination: false
summary: A little primer and tips from how I use Git.
draft:true
tags:
- git
title: How I Git
---

Perhaps the one piece of ubiquitous technology that you will find at any new
tech company is `git`.  There are a couple of other tech that you will likely
find like AWS, but `git` is the only one I expect is truly ubiquitous. It is
also, surprisingly, many developers number one frienemy.  I want to share some
of my favorite tips and tweaks that I have used over the years to make it all
friend and never my enemy.
<!--more-->

First, let's get this out of the way.  I like `git`, a lot. I am one of the odd
people that think it makes sense. I can't defend how complex it can sometimes
get, but I think that by and large the design and philosophy works for me and 
I can do a lot with it. 

# Best Practices

## Branching
Use 2 permanent branches `master` and `develop` with ephemeral branches for
`features` and `hotfix`es.  Many people will advocate for other branching
strategies but I think that this model is easy to convey and easy to mentally
track. All new code is merged into `develop` and when you are ready to release
something you merge `develop` into `master` and then tag the HEAD of `master`.

Need to fix a bug in production? Create a `hotfix` branch off of `master`,
merge that into `master`, tag it, deploy, and then merge master back into
`develop`.

Want to build the next cool feature? Create a `feature` branch off of
`develop` and then merge that back into `develop` when you are done and move on
to the next hot thing.

I like this pattern because it is simple and gives you quite a bit of control
over what happens when. I have seen many people advocate for a single `master`
branch that everything goes into. But, unless you have really good test
coverage and are doing true continuous integration where every commit is
deployable; the single branch policy will eventually breakdown on a team of any
reasonable size. It simply requires a discipline that I haven't seen larger
teams maintain. 

At Teem, we use one more semi-permanent branch we lovingly refer to as the
`release` branch.  For each release (we release weekly) we branch off of
develop into a branch named `release-<version_num>`.  We then deploy
this to a staging server for QA to validate. Having this additional branch
allows development of new features to keep going without causing a lot of
headache or confusion about what is in the release and if QA finds an issue,
how to merge the fix for that release.  Everything for that release is branched
from and merged back into the `release` branch. Finally, when it is time to
release, we merge the `release` branch into `master` and tag the HEAD. Backport
`master` into `develop`, rinse and repeat.

## Branch names
Please, for the love of all that is good use descriptive branch names. I
suggest the following naming patterns

```
feature-<ISSUE_ID>-short-summary-of-branch
hotfix-<ISSUE_ID>-short-summary-of-branch
releasefix-<ISSUE_ID>-short-summary-of-branch
```

Doing this allows anyone that is reviewing that branch to have some idea of
where that merge should be going, e.g. `feature` branches merge into `develop`
not `master`.  Having the issue id (we use JIRA, but this could be the id from
any ticket tracker) allows people to reference what the branch should be
addressing, be that a bug report or a user story.  And finally, a short summary
makes the branch descriptive and easier to use. I also recommend this pattern
because it then becomes easy to create a changelog from the git log of merges.

## Commit Messages
Let's get the less contentious piece of advice out of the way, commit messages
should be informative and well structured.  I have generally followed the
advice of
[tpope](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)
and the 
[`git` handbook](https://www.git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project#Commit-Guidelines) 
but with a slight tweak. Here is a example commit message

```
Capitalized, short (50 chars or less) summary

**What**
- Bullet pointed list of changes you made. 
- Each line should be no longer than 72 characters.
- For example:
- Switch from o365 beta API to the v1 API

**Why**
- Again a bulleted list of reasons you made that change.
- Again, no longer than 72 characters.
- For example:
- The v1 API is now stable and the beta API contains breaking changes with the
v1 api.

**Notes**
- Any additional notes for peer reviewers or to add additional context. 
```

The most important thing to remember, this message is supposed to inform people
about what is happening in the project without them needing to read every file.
For peer review, it is important that they can recognize what is intended to be
in the change set and what should not be there. One-line messages help no one.

To help ensure that you write a half-decent commit message, `git` has a feature
where called `.gitmessage`.  Create a file called `.gitmessage` in your home
folder and put this in it

```
Captialized, short (50 chars or less) summary

**What**
-

**Why**
-

**Notes**
```
`git` will now use that as the template/initial text in all of your commit
messages.  

Note, it will not have any impact on specifying the commit message when using 
`git -m`.

## Commits
Let's start with something that is probably not so contentious: I believe in 
committing frequently.  And now something slightly more contentious: I believe 
in using rebase to create a sensible history that makes something like 
`cherry-pick` simple to use. I generally believe that you should work on small
chunks of code that can reasonable be described in a single commit with one or
two comments in the "What" section of my commit message. I also believe that
you should break coding style fixes, e.g PEP8 fixes, into separate commits so
that they can be reviewed separately. To actually make all of these ideas play
nicely together I use rebase frequently. I squash my frequent small commit into
larger (but still fairly small) semantic pieces of history, so my git log will 
go from something like this

```
sha1 - "Add new contact method"
sha2 - "Fix typo in contact method"
sha3 - "Add new unit test"
sha4 - "Fix bug found by unit test"
sha5 - "Fix PEP8 issues with imports"
sha6 - "Fix PEP8 issues with line lengths"
```

to something like this

```
sha1' - "Add new contact method and unit test"
sha2' - "Fix PEP8 issues"
```

This history makes it easy to cherry-pick or revert the new feature and it
makes it easier for peer reviewers to review the logic change in `sah1'`
independently of the potentially noisy and distracting PEP8 changes in `sha2'`.
To do this I make heavy use `git rebase -i` to selectively squash commits.  I
have also created an aliases called `git fixup` that will simply squash my
staged changes into my previous commit. More on aliases later. 

## On rebasing
I do not intend to give a full defense of rebasing here. I will say this, if
you are not comfortable with `git`, then rebasing may not be for you.  Almost
everything else you do in `git` is fairly safe, there is a way to recover from
what you are doing, this is why I like and trust `git`.  However, `rebase` is
one of those commands that is not always recoverable and you often have to
simply live with the end results. With that said, if you are using rebase
especially if you are using `pull -r`, which is a standard pull that uses
rebase instead of merge, then you must enable and configure the `rerere`
feature. Quick, update your `~/.gitconfig` to have
```
[rerere]
    enabled = 1
    autoupdate = 1
```
you will thank me. From the [git
book](https://git-scm.com/blog/2010/03/08/rerere.html):

>  The name stands for "reuse recorded resolution" and as the name implies, it 
>  allows you to ask Git to remember how you've resolved a hunk conflict so 
>  that the next time it sees the same conflict, Git can automatically resolve 
>  it for you. 

Basically, while you are rebasing, if you have a conflict that you resolve, git
will now remember that resolution and automatically apply it again in the
future.  This is huge for `git pull -r`, which will often replay the same
section of your history and therefore run into the same conflict over and over.
Honestly, I don't think rebase is usable without enabling `rerere`.

# Aliases
Any post on `git` can not be complete without a list of handy-dandy aliases and
commands.  So, here are mine.  I have roughly 3 categories of aliases: audit
and cleanup, historical logging, and commit helpers.

## Audit and cleanup
The following commands help me clean up old branches
```
audit = !git branch --merged | grep -v '\*\|master\|develop\|release-'
clean-audit = !git branch --merged | grep -v '\*\|master\|develop\|release-' | xargs -n 1 git branch -d
b = !git for-each-ref --sort='-authordate' --format='%(authordate)%09%(objectname:short)%09%(refname)' refs/heads | sed -e 's-refs/heads/--'
trim = !git reflog expire --expire=now --all && git gc --prune=now
```

I use `audit` and `clean-audit` the most frequently.  `audit` simply lists my
local branches that have already been merged (except for `master`, `develop`,
and the `release` branches) and hence are not needed anymore. `clean-audit`
simply extends that command to delete the listed branches. 

The `b` alias prints a summary of all local branches, it looks likes this

```
Fri Mar 17 16:19:24 2017 -0600	4d2a351	master
Fri Mar 17 15:08:31 2017 -0600	d9ce23e	release-17.12.01
Fri Mar 17 13:21:34 2017 -0600	a5d5f73	develop
```

I use `trim` pretty sparingly. It simply cleans up old branch pointers that are
not being used as of "now".  This can be useful to reclaim some space after you
do `clean-audit`.  I only recommend this command if you have OCD.

## Historical Logging
The following commands will print your `git log` in fun, potentially
informative, ways:

```
graph = log --graph --oneline --decorate --all
l = log --pretty=format:%C(yellow)%h\ %ad%Cred%d\ %Creset%s%Cblue\ [%cn] --decorate --date=short
ll = log --pretty=format:%C(yellow)%h%Cred%d\ %Creset%s%Cblue\ [%cn] --decorate --numstat
```

Assuming that you and your team are writing good summary lines in your commit
messages, these commands can be used to quickly find when a certain commit
happened.  I don't use these frequently, but they have been useful when trying
to find when something happened in the repo.  `git l` is great a really short
summary of the history, e.g. 

```
af1f3ec 2017-03-05 (HEAD -> source-hugo, origin/source-hugo) Site rebuild Sun Mar  5 13:53:22 MST 2017 [Lucas Roesler]
ddfdd9d 2017-03-05 Post new spring hefe beer post [Lucas Roesler]
e5bd906 2017-02-25 Site rebuild Sat Feb 25 13:33:12 MST 2017 [Lucas Roesler]
bd21f27 2017-02-25 Add a reading list page [Lucas Roesler]
dad0fbb 2017-02-18 Site rebuild Sat Feb 18 20:20:50 MST 2017 [Lucas Roesler]
```

If you need a little more detail, `git ll` will show you the change stats as
well as the summary that you get in `git l`, e.g. 

```
af1f3ec (HEAD -> source-hugo, origin/source-hugo) Site rebuild Sun Mar  5 13:53:22 MST 2017 [Lucas Roesler]
21      1       public/2017/01/hello/index.html
21      1       public/2017/01/my-management-philosophy/index.html
21      1       public/2017/01/spicy-winter-porter/index.html
```

## Commit helpers
I only have one alias related to commits `fixup`

```
fixup=!git commit --amend
```

This command will take your staged changes and immediately squash them into the
previous commit.  This is great for fixing small typos and simply reduces the
amount of time I need to spend in `rebase`. This is by far my most frequently
used alias. 

## Adding aliases
Adding these aliases to your system is pretty simple. In your `~/.gitconfig`
file, add or update the `[alias]` section with the snippets I shared above. My
config file looks like

```
[alias]
    fixup=!git commit --amend

    # cleanup old branches
    audit = "!git branch --merged | grep -v '\\*\\|master\\|develop\\|release-'"
    clean-audit = "!git branch --merged | grep -v '\\*\\|master\\|develop\\|release-' | xargs -n 1 git branch -d"
 ```

