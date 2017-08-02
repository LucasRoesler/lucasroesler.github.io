---
title: Berlin and Golang
summary: I am moving to Berlin and will soon start using Go as a daily language!
date: 2017-08-02T11:00:00-04:00
categories:
- programming
- golang
tags:
- golang
- berlin
keywords:
- golang
- berlin
comments: false
showPagination: false
autoThumbnailImage: true
coverSize: partial
# coverImage: https://farm4.staticflickr.com/3928/15343861037_dd63578267_c.jpg
# coverImage: https://farm8.staticflickr.com/7196/26861998346_007d0a234e_c.jpg
coverImage: https://farm1.staticflickr.com/217/522864770_f0eab013e7_b.jpg
thumbnailImagePosition: left
---

At the beginning of the month I left [Teem](https://teem.com/) I was Directory of Engineering and it was and still is a great company.  There are some amazing developers there, so if you are looking for a job in SLC, hit them up.  This weekend (Aug 5) I will be moving to Berlin to start a new role at [Contiamo](https://www.contiamo.com/) and will be working on the [Labs feature](https://docs.contiamo.com/en/labs/).

<!--more-->

While I will miss the developers at Teem, I am super excited about this move.  First, my wife and I honeymooned in Berlin (Munich and Vienna) and absolutely loved it. We have been talking about the possibility to move abroad for years and Berlin has long be at or near the top of our list of target cities. My initial Germany bucket list includes

1. Learn German
2. Drink a lot of beer
3. Hike in the Black Forest
4. Harz National Park
5. Neuschwanstein
6. Saxon Switzerland National Park
7. Visit the Chocolate Museum in Cologne
8. Of course, Oktoberfest

Second to simply being in Germany, I am really excited to deep dive into [Go](https://golang.org/).  I have of course worked with a typed language for little projects here or there, but most of my professional work has been Python and Javascript. Of course, most of that time was spent swearing at the language :cough: Java :cough:, just ask the developers at Teem.  One thing I have _very slowly_ been working on is a port of [octoeb](https://github.com/enderlabs/octoeb) to  Go. This is great because I wrote and designed the original, so I have a very clear picture and understanding of the design. I can focus on learning the particulars of Go. Additionally, I can add some needed improvements to the initial design. Among other things, I plan on adding an `init` command to simplify configuration and to redesign the internal logic to allow it to be more easily extended to other services beyond Github and Jira (Bitbucket, Github issues, etc).

This blog has so far focused on Python work I have done, naturally a large portion of the technical posts will begin focusing on Go.  But I don't expect to completely stop Python anytime soon.  I do expect most of my Go posts to pretty introductory, at least for the remainder of the year.

First noob lesson in go: variables defined in a package are exposed and defined in all files in that package.  For example, if you have two files `foo.go` and `bar.go`, this will produce a compile error.


```go
// file/foo.go
package foo
var (
    Foo string
    err error
)
```

```go
// file/bar.go
package foo
var (
    Bar string
    err error
)
```

This was quite a surprise from my python experience.  I am sure that I will run across many more lessons like this. As this goes on, please feel free to let me know of any tweaks or improvements or Go patterns that I am missing.

---

Thanks to [m.a.r.c](https://flic.kr/p/NcPtQ) for the CC BY-NC image that I used as the cover on this post.
