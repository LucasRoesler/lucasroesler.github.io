---
date: 2017-01-15T00:00:00Z
draft: true
title: Postgres Joins and Django Querysets
summary: Dealing with inefficient joins in Django's ORM
keywords:
- tech
- python
- django
tags:
- django
- python
- postgres
categories:
- programming
showPagination: false
---

How do we build a fast API against database models with foreign keys and
many-to-many relationships?  If you do nothing you get the "waterfall of doom", but,
Django provides a couple of options to avoid this.

<!-- more /-->