---
date: 2017-01-15T13:15:43-07:00
draft: true
summary: "The Django ORM is great but it can not always efficiently write the
queries that you need, sometimes you need to write SQL! I want to share an idea
about how to mix Django and SQL at the same time."
tags:
- programming
- python
- django
title: Writing raw sql and the Django ORM, a short tale.
---

The Django ORM is great, it allows us to quickly get applications built
but it isn't always the the fastest solution.  If you really need speed
you need to get comfortable writing SQL, but how do we leverage Django
while doing this?  This is espeically important if you are using Django Rest
Framework and all of the powerful tools and concepts it brings to the table.

<!-- more /-->

