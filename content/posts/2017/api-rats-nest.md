---
categories:
- programming
- rest
comments: false
date: 2017-06-05T21:22:00-06:00
keywords:
- api
- rest
showPagination: false
summary: Please, do not use nested routes. Remember the Unix way, do one thing and do it well.
tags:
- api
- rest
title: Your API's rat nest
autoThumbnailImage: true
coverSize: partial
coverImage: https://c1.staticflickr.com/9/8196/8445504396_4fd22ef25e_c.jpg
thumbnailImagePosition: left
---

Recently, I have seen several articles talking about RESTful API design.  Of
course this is also a common topic of discussion for the engineers at Teem.  I
want to use (and write) APIs that are easy to understand and explain and the
fastest way to complicate your API is nested routes. Just don't do it! Do not
create nested routes in your API.  Let's keep our APIs simple and create one
endpoint per resource and if filters are needs, use GET parameters. This is
simpler to document and simpler to maintain and ultimately, easier to use.

<!--more-->
In a RESTful API urls identify resources and identifiers should change
as infrequently as possible [[Fielding pg. 110](https://www.ics.uci.edu/
~fielding/pubs/dissertation/fielding_dissertation.pdf)].  The natural
consequence is that urls should contain the minimal information needed
to access a resource. For example, in an API with Users and Companies,
`/users/:userId` is better than `/companies/:companyId/users/:userId`.
If your user changes companies, then the identifier must change.  This is also
more complicated than it needs to be. The user's `id` should be enough to
access it from the database.

Some people will argue that you solve the above problem by implementing
_both_ endpoints.  However, if you pair this with the Unix philosophy
["Do One Thing and Do It Well"](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well),
then you simply shouldn't use nested routes. It is simpler to understand
an API that has only one way to access any given resource. Additionally,
any request that you can do with nested routes you can do with a non-nested
route and query parameters. This also reduces the complexity of
managing and testing endpoints because there is only one code path that
users/developers can use.  Basically, the non-nested endpoint is works
it works well, so why create another endpoint to clutter up the API?

At the end of the day, APIs without nested routes are easier to build, easier
to maintain, and easier to consume.  This leads to happy devs and ultimately
happy users.

---

Thank you to
<a data-flickr-embed="true"  href="https://www.flickr.com/photos/dookington/8445504396/in/photolist-dSiqdL-HoMKRm-a5BWuE-cQJcpj-5aQHDt-61PKYd-8kEUCJ-9psndx-2MYpnV-J8kzFb-dvmSjb-RCLYho-FiCvSu-6CMKpd-5xAM4H-3XTBB6-CGS8y3-CuucdJ-RfW4Pp-So3x7N-s1Vh8-m7cCmc-o92Zw4-b3SSGX-528Fwg-a3DqaD-8j9o5t-fy12R9-7xYWoU-dBJ3Gj-prjyoB-bY6Cdy-68bcuk-6Qkn9E-dSPirj-7Gyme1-fFTbHv-3cJ73y-bsKUze-svnwP6-7uH8S-RrmYp1-o29b6m-TcXH28-tfbf-4orsJt-ehk6kN-4pYNoQ-5Sw1iK-cxkJs7" title="Wired">Tom Ducat-White</a> for the CC BY-NC-ND image I used as the cover on this post.