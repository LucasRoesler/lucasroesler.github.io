---
categories:
- programming
comments: false
date: 2017-01-15T16:37:52-07:00
draft: true
keywords:
- python
- django
- automation
- continuous integration
showPagination: false
summary: How we automate the validation of our Django migrations
tags:
- python
- django
- automation
- continuous integration
title: Automatic database migration validation
---

At Teem, we are slowly moving closer and closer to a continuous integration workflow.  To get there we need to trust that the incoming changes are not going to break the deploy.  One of the areas that are hardest to ensure we are safe are database migrations.

<!--more-->