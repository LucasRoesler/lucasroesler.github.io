---
title: "Go 1.18 Released"
description: "Generics, string utils, and workspaces, oh my!"
date: 2022-03-20T00:00:00+02:00
tags:
  - golang
  - programming
draft: false
# Photo by <a href="https://unsplash.com/@remyloz?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Remy_Loz</a> on <a href="https://unsplash.com/s/photos/workspace?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
coverImage: images/remy_loz-3S0INpfREQc-unsplash.jpg
---

Go 1.18 was [released last week (March 15)](https://twitter.com/golang/status/1503787326060875782?s=20&t=BJLVhJnsJKSloA1D0LPEHg) and I have finally had a chance to play with it. I have been excited for it in the month leading to the release, reading almost every preview that I saw on Twitter, and the release has not disappointed! I think the last release that got me this excited was 1.16, when they added [`embed`](https://pkg.go.dev/embed).

Today, I want to share a few of things that made me a bit more excited than usual for a Go release.

---

## Generics!

First, let's deal with the [elephant in the room](https://en.wikipedia.org/wiki/Elephant_in_the_room), [generics](https://go.dev/doc/go1.18#generics)! To say that this has been a [long time coming](https://go.dev/blog/generics-proposal) and controversial would be an understatement. As a result, this has been in the works for a long time. One of the first proposals for Generics was [back in 2010!](https://go.dev/design/15292/2010-06-type-functions).

I am one of those people that was actually looking forward to this change. Over the past year I have actually had a few moments where Generics would have made my life _much_ simpler. I am not going to start using it for _every_ function I write or anything like that, but I will not shy away. However, I am also not going to review how Go's Generics work, the Go team has already written a [great tutorial](https://go.dev/doc/tutorial/generics) and I recommend you start there.

## Strings!

Oddly, the feature I was most excited for was a new `Cut` method added to [`strings`](https://pkg.go.dev/strings#Cut) and [`bytes`](https://pkg.go.dev/bytes#Cut). When I first read about that addition, I yelled "oh yeah, awesome!", my wife can confirm. This is a small quality of life change and I love it. This is the perfect function to help you do things like parsing `key=value` pairs, which is where I have already used it. If you aren't sure about the usefulness of `Cut`, "why not use use `Index` and string slicing?", then checkout the [detailed explanation from Russ in the Pull Request](https://github.com/golang/go/issues/46336).

## Workspaces!

The next feature I didn't realize I _should_ have been excited for and will probably use almost daily is [workspaces](https://go.dev/doc/tutorial/workspaces). Workspace mode makes it easier to work with multiple modules. This is obviously a huge boon for teams/projects that use monorepos. I don't really prefer monorepos. Personally, the tooling for them are not that great (yet). However, I do like working on distributed systems and will probably use `workspaces` everyday.

So what is a workspace? From the docs, they say

> A workspace is a collection of modules on disk that are used as the root modules when running minimal version selection (MVS).

> A workspace can be declared in a go.work file that specifies relative paths to the module directories of each of the modules in the workspace. When no go.work file exists, the workspace consists of the single module containing the current directory.

Basically, this can be _any_ folder where the sub-folders are Go modules. Even if you don't use a monorepo, you will still want to create a workspace for any project with multiple components/repos. Creating a workspace will allow your editor/IDE to simultaneously lint and autocomplete all of the components in that workspace.

I like to organize my `code` folder so that components are nested together in sub-folders:

```
code
├── company
│   ├── componet-a
│   ├── componet-b
├── openfaas
│   ├── faas
│   ├── faas-cli
│   ├── ...
│   └── faas-netes
└── sandbox
    └── bug-repro
```

As a result, previously, when I opened `company` in VSCode, linting would just be broken. It would complain about detecting multiple modules and just mark everything in red. It was not a workable dev environment until switched to a subdirectory for a single component.

With Go 1.18, I can create a `go.work` workspace for my `company` folder like this

```sh
cd code/company
go work init .
go work use component-a
go work use component-b
```

and just open the `company` folder and everything works! This really cuts down on the number of editors I need open at the same time (or switching the active project in my current editor window).

You probably don't want to create a workspace for your _entire_ code folder. I suspect there would be performance issues and it would be annoying to manager that `go.work` file .

### Workspaces for OpenFaas?

One area where I do use a "monorepo" is when I am building serverless projects that are composed of multiple functions.

OpenFaaS already supports Go modules. But now adding a workspace to the project root (like above) it is really easy to edit all of the Go functions in a single project from the same editor.

However, none of the official templates have any native support for workspaces yet, in fact, adding a workspace to a function itself would likely break it or be ignored (at best).

## Next steps

In the next weeks I am going to explore what proper support of workspaces or maybe use workspaces to simplify some of the build code we use for OpenFaaS templates. I really want to see what can be made of the [`copy common folders`](https://github.com/openfaas/faas-cli/issues/322) feature in combination with workspaces. I originally built the copy feature to make sharing code between python and nodejs functions easier, but there was no great way to do this before. I think the combination of workspace and a replace statement might allow the same code share for Go functions.

---

Cover Photo by [Remy_Loz](https://unsplash.com/@remyloz?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/workspace?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
