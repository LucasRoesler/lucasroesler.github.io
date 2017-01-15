---
categories:
- programming
comments: false
date: 2017-01-15T13:13:10-07:00
draft: true
keywords:
- python
- django
summary: Should you remove data from the database or simply mark it as deleted?
tags:
- python
- django
showPagination: false
title: Delete or not to delete
---

Should you remove data from the database or simply mark it as deleted?

<!-- more /-->

At Teem we have a lot of data that we need to manage and often "physically"
deleting the data can be problematic, e.g. the LobbyConnect visitor list should
never be physically deleted.