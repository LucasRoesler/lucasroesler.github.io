---
date: 2017-02-06T20:00:00Z
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

How do we build a fast API against database models with foreign keys and many-
to-many relationships?  If you do nothing you get what I call the "waterfall of
doom". At some point in the past someone told me or I read that "joins are
effectively free in Postgres".  While this might be somewhat true when you are
writing all of the SQL and can control every part of your query; I have recently
found that when the database gets big enough and you are using the Django ORM,
joins aren't free and less can be more!

Warning, I am not a DBA and mileage may vary.

<!-- more /-->

At [Teem](https://teem.com) we deal with a lot of calendar data.  For various
reasons this means we have database tables that looks something like this

```
+----------------------+    +----------------------+
|Calendar              |    |Event                 |
|========              |    |=====                 |
|- id                  |    |- id                  |
|- source              |    |- organization_id     |
|- organization_id     |    |- calendar_id         |
|                      |    |- organizer_id        |
+----------------------+    |- title               |
                            |- start_at            |
                            |- end_at              |
                            +----------------------+
+----------------------+    +----------------------+
|Participant           |    |Organizer             |
|===========           |    |=========             |
|- id                  |    |- id                  |
|- organization_id     |    |- organization_id     |
|- event_id            |    |- event_id            |
|- email               |    |- name                |
|- name                |    |- email               |
+----------------------+    +----------------------+
```
(_Note this is not how it is actually setup in production._)

Now, you can be happy or upset with the design for various reasons
but my biggest issue is how Django queries these tables for our API.  The
simplest most direct way to serialize an Event including its calendar,
organizer, and the participants cause what I call the "waterfall of doom",
a.k.a. the N+1 problem. This topic is covered in many places throughout the
internet, for example:

* [Performance: N+1 Query Problem](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/)
* [What is the n+1 selects problem?](http://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem)

so I won't focus on it here; but basically, every request for an Event results
in 4 more requests (_the waterfall_) for Organization, Calendar, Organizer, and
the Participants respectively.  Even worse, if I request a list of Events the
simplest code for the API will do 4 database queries for each Event in the list
(_the waterfall of doom_). For a single event this isn't noticeable, but if I
want to serialize all events in a calendar, it is a big performance problem.

We use [Django Rest Framework](http://www.django-rest-framework.org/), so the
simple (and bad) Event serialization described above looks something like this:

```python
class EventSerializer(ModelSerializer):
    organizer = OrganizerSerializer(read_only=True)
    calendar =  CalendarSerializer(read_only=True)
    organization = OrganizationSerializer(read_only=True)
    participants = ParticipantSerializer(
        source='participant_set', many=True, read_only=True)

    class Meta:
        fields = (
            'id', 'start_at', 'end_at', 'title',
            'organizer', 'calendar', 'organization', 'participants',
        )

class EventViewSet(ReadOnlyModelViewSet):
    queryset = Event.objects.all()
    serializer_class = EventSerializer
```
It is worth noting that this is clearly not how our production API is currently
designed; but, it demonstrates the core performance issue. Fortunately,
[Django provides a couple tools](https://docs.djangoproject.com/en/1.10/topics/db/optimization/#retrieve-everything-at-once-if-you-know-you-will-need-it)
out of the box to help us solve this performance issue: `prefetch_related`
and  `select_related`.

If you are familiar with Django, then this is not news to you.  Given the setup
above you would likely jump on `select_related` and call it a day, e.g.

```python
class EventViewSet(ReadOnlyModelViewSet):
    queryset = Event.objects.select_related(
        'organizer', 'calendar', 'participant_set').all()
    serializer_class = EventSerializer
```

In fact, for a short time, this optimization worked for us; but a problem pops
up when the database starts getting big.  The above ViewSet will generate SQL
like this

```sql
SELECT COUNT(*) FROM (
    SELECT DISTINCT ON (
        e.start_at, e.end_at, e.title
        )
        e.*,
        o.*
    FROM event e
        LEFT OUTER JOIN participant p
            ON ( e.id = p.event_id )
        LEFT OUTER JOIN organizer o
            ON ( e.id = o.event_id )
    WHERE (
        e.calendar_id =  5051
        AND e.start_at <= '2016-12-02 00:00:00+00:00'
        AND e.end_at >= '2016-12-01 00:00:00+00:00'
        AND (
            UPPER(p.email::text) = UPPER('name@example.com')
            OR UPPER(o.email::text) = UPPER('name@example.com')
        )
    )
    ORDER BY
        e.start_at ASC,
        e.end_at ASC,
        e.title ASC,
) sub;
```

Which looks good, but here is the problem

![Example Performance with select_related](https://lh3.googleusercontent.com/42eujc1S4_syWt2a64akZ28YxvBPofo2vHeJLWE_OwwSyruc_xLZvoe2RPIsRHHM_hTqHgjT6tH1-v9U76juQ2Ik7qInTADCbMpqFJC-hIswhJt8u36Y4dXdla9ffOy1GkI2OFxO_OXsSlDpjJJEUJs3cew54TEiLOT7zupFmzyA2LVGf8aYExjzKpYIQLdctzPPk59ecdNlhtscFZNl6Q47vHKyyq04CYOaHHop-KrOJMVAsorJbb8EYeVnrcanmM6Iowy5hkfRa2NuUMEvzOnd3hoZPxSVxNRTZukULPTsgCmTw5-cbUQi_9T3zrIRpypqEzjH1WaCaBeTtzBJlncNEC6MYM1GFP7e68qBMZpuAIzO-i4i43dTY5n3IfpY--ESYXtOp6xhKEWvOIDUtYGxXNow7HOWuluiOC0rWffxdDUWgDgv_CAAmVBCIWMvUXPOinz2N18V28-SeMsLG7Lsi6CiMu4ZfJGA44m6vY1CAi_3pQTXdB7xr1YU05Bzu4qyJiV0hRThIrC3GIj4nPo0wxI1PYneHY-AnAXmxVpMerQkKdtYpglv3DuJQraoxJg9Iu_nxoQO73fi6C7ZXisrVj-wCZE6_HsRYlsm5Yp_kH365gsX=w1440-h476-no)

21 seconds querying the events table in our real life production database! Why
oh why is this happening?  A quick `EXPLAIN ANALYZE` shows that Postgres is
doing sequential scan on `participant` and `organizer`.  Now, I am not a DBA so
I can't fully explain why this is happening, but I do have some ideas about how
to avoid the scan.  Here is the fastest query I could make in SQL

```sql
SELECT COUNT(*) FROM (
    SELECT DISTINCT ON (
        e.start, e.end, e.title,
        )
        e.*,
        o.*
    FROM event e
        LEFT OUTER JOIN (
            SELECT * FROM participant
            WHERE
                participant.organization_id = 9776
                AND UPPER(participant.email::text) = UPPER('name@example.com')
        )
        p ON ( e.id = p.event_id )
        LEFT OUTER JOIN (
            SELECT * FROM organizer
            WHERE UPPER(organizer.email::text) = UPPER('name@example.com')
        ) o ON ( e.id = o.event_id )
    WHERE (
        e.organization_id =  9776
        AND e.start_at <= '2016-12-02 00:00:00+00:00'
        AND e.end_at >= '2016-12-01 00:00:00+00:00'
        AND (p.id IS NOT NULL OR o.id IS NOT NULL) -- User filter applied in subselect joins
    )
    ORDER BY
        e.start_at ASC,
        e.end_at ASC,
        e.title ASC
) sub;
```

This query is fast in production, but ... it can not be produced using the
Django ORM. Here, I explicitly made subqueries on indexed columns to speed it
up. In a future post, I promise to discuss how and when we use raw SQL so speed
up some of our requests. But, for the general CRUD API endpoint, I want to use
the ORM so that I can leverage the great filter framework provided by Django
Rest Framework. After a lot of tinkering, the best ORM compatible query I could
design is this

```sql
SELECT COUNT(*) FROM (
    SELECT DISTINCT ON (
        e.start, e.end, e.title
        )
        e.*,
        o.*
    FROM event e
        LEFT OUTER JOIN participant p
            ON ( e.id = p.event_id )
        LEFT OUTER JOIN organizer o
            ON ( e.id = o.event_id )
    WHERE (
        e.organization_id =  9776
        AND e.start_at <= '2016-12-02 00:00:00+00:00'
        AND e.end_at >= '2016-12-01 00:00:00+00:00'
        AND (
            UPPER(p.email::text) = UPPER('name@example.com')
            OR UPPER(o.email::text) = UPPER('name@example.com')
        )
        AND (p.organization_id = 9776 OR p.organization_id IS NULL)
    )
    ORDER BY
        e.start_at ASC,
        e.end_at ASC,
        e.title ASC
) sub;
```

For specific API queries, we actually using a similar query in production. But,
I couldn't find a good general solution until I realized that trying to do
everything at once might actually be too much to ask for, especially since the
API enforces relatively small pages. While reading through the documentation to
hunt down another bug, I [re-read the docs](https://docs.djangoproject.com/en/1.10/ref/models/querysets/#prefetch-related)
son `prefetch_related` and it occurred to me that I could guarantee
exactly 4 fast queries to the db on every API call instead of one sporadically
slow query.  With the following small change to our viewset definition, our
slowest API call is an order of magnitude faster than the previous call with the
`select_related` (although there is room for improvement)

![Performance with prefetch_related](https://lh3.googleusercontent.com/pUhp4eFa0T-hdtkHBUiowhLMhz9VTbWjt7L7iorDjvGpvybFl32Y-hK70ndpgmiCm33cHhsKqFSaj99IERwt28YHe-Ndb9jutVHWHCoVq0hjiCnws7HhgqGh5z1v40yZbihCgxDTf8Wf-dAXwyAYQ0RttclPPNIE4sLMUioLnkosHWYJ_if9VVjJ6xOPZACDGC70MijYqOG2TfYyKNBGmExl6mFf-F2v-vkNsXHFQzs9Wm12yM0TJ7Hrx7LOrHIOcVjb3OxBveoedc_qXoG89rTb3h9xvRRDigG0bEFcEYVRi3WjNsXt3gpuXb05tOepcgpLAHG8Nfpjx0cW0axwInwVPdrxLcfEbH6SnnnFNpTQQIX3UtEDV3z0AKvpFQwfvz8lfdc2Jr3FCXJqG0AtxXV1PwbWMZ4wudtcgLogUYkxXsfzTTRlDO5p8bPZZQSdfCDCrJdp_3lj1FUK7aNZsTVN-7-lVKdCQbYXEOit01zA_Jrv3fWM5znx1OPe_kMNWQqnpOk6uRT97zAi2Boopu4fnyoLn-wDfuiw10wpfaQEFIC910tSOeD1RBn9AF5lOlbop_vYrTU5Z5VBN3izI-ug8ucJKk6YgIb9bv2epzGTDUaehdIheMYEWtoAnmcSFE3XEFR--hy1m_w-1TZAIS4ex_A0O-cQ5DgaFzKTfg=w2286-h620-no)

In fact, the careful observer will notice that the called components in this
trace are a bit different.  Well, the `prefetch_related` was so successful,
that the same exact query no longer qualifies for being traced in New Relic and
the average dropped by half!

![average reservations api call](https://lh3.googleusercontent.com/IGNUFurt9JEPG5c_zHAO8RkML0joObxq66RPZ7ga3xrXYGpnKiNjUPMDAsGCGuEZ4nyMJz7crE5a5UuBQuMFSTl6biLTsahB6MPbbCLehVaBGhaJTQvrwvWao0049ksDScEb6l5KgqPr8kXZ75L1k3V1B2QGpzUTeV0jN3d2SWGtuHFrW6K2wNdnjGifnBP5bXcB_3MoAvj3ZAXmvmI08_o_n_UjSN0fSMAupq2tR013pB4HGYr1HXIN9-UZRc9oqjMqtu2rslGQi0FTxrOHY5xfn6lX-0Hzy8d0qKSkhrkW8GYkk0rZwRcoQonrivtCOMeXkFB2d5l45FZlW0oM2Yo0W9Zc_jFKeRnLSyX0cdbutBCoR1Q4KFxRKz0dFrPJFCvUA4gngJ_IYdkriL6uAIq1p3lHPze_1XI2reHp2qM689-ggmppmwrlyT6QVVMy5jA6xKG7hklVkmtgsm-aj8nw5tD3OkCbJLRlAJJabIxusEKHTTI5rDtXKaqkBAidWmAF1PMuEuj2wcNQvLCN93Y2Sy_zL-aY4JDpRFbmbYh2PfJarmHhibCUF3pWK79kI_uqLwmPb1a6g1UkuFQb1zA2FmXaoKD_T8wUBOUWfQ6NhKg6gCV_SYngrtMcvI3-nDmcvIALEn7xOaM32TGOAPx5igkG7QDkM2-dKHKjNA=w1260-h298-no)

I would like to caution that I believe this works so well in this case because we
limit the API page size, so, the data and query sizes in the related queries are
relatively small per request. If you need to load a lot of data at once, I can
foresee this solution being much worse than `select_related`.

TL;DR: Monitor and analyze your production queries and `prefetch_related` can
be a great solution if you can keep the number of queries and the page size
small.