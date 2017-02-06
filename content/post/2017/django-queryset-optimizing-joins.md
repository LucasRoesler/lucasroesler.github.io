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
Django provides a couple of options to avoid this.  Recently, I have found that
when the database gets big enough sometimes the answer is to avoid the join altogether!

Warning, I am not a DBA and mileage may vary.

<!-- more /-->

At [Teem](https://teem.com) we deal with a lot of calendar data.  For various
reasons this means we have database tables that looks sort of like this

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

Now, you can be happy or upset with design for various reasons
but my biggest issue is how Django queries these tables for our API.  The
simplest most direct way to serialize an Event including its calendar,
organizer, and the participants cause what I call the "waterfall of doom",
a.k.a. the N+1 problem. This topic is covered in many places throughout the
internet, for example:

* [Performance: N+1 Query Problem](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/)
* [What is the n+1 selects problem?](http://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem)

so I won't focus on it here; but basically, every request for an Event results in 4 more requests
(Organization, Calendar, Organizer, and the Participants).  Even worst if I request a list of Events,
the simplest code for the API will do 4 database queries for each Event in the list.
For a single event this isn't noticeable, but if I want to serialize all events in a calendar,
it is a big performance problem.

We use [Django Rest Framework](http://www.django-rest-framework.org/), so this bad serializer
looks something like this:

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
It is worth noting that this is clearly now how our production API is currently designed; but,
it demonstrates the core performance issue. Fortunately,
[Django provides a couple tools](https://docs.djangoproject.com/en/1.10/topics/db/optimization/#retrieve-everything-at-once-if-you-know-you-will-need-it)
out of the box to help us solve this performance issue: `prefetch_related`
and  `select_related`.

If you are familiar with Django, then this is not news to you.  Given the setup above
you would likely jump on `select_related` and call it a day

```python
class EventViewSet(ReadOnlyModelViewSet):
    queryset = Event.objects.select_related(
        'organizer', 'calendar', 'participant_set').all()
    serializer_class = EventSerializer
```

In fact, for a short time, this optimization worked for us; but a problem pops up
when the database starts getting big.  The above ViewSet will generate SQL that

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
up. Now this is not exactly a problem and I promise to discuss how and when we
use raw SQL so speed up some of our requests. But, for the general CRUD API
endpoint, I want to use the ORM so that I can leverage the great filter
framework provided by Django Rest Framework. After a lot of tinkering, the best
ORM compatible query I could design is this

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

For specific API queries, we actually using a similar query in product. But, I
couldn't find a good general solution until I realized that trying to do
everything at once might actually be too much to ask for.  While reading through
the documentation to hunt down another bug, I [re-read the docs](https://docs.djangoproject.com/en/1.10/ref/models/querysets/#prefetch-related)
on `prefetch_related` and it occurred to me that I could guarantee
exactly 4 fast queries to the db on every API call instead of one sporadically
slow query.  With the following small change to our viewset definition, our
slowest API call is an order of magnitude faster than the previous call with the
`select_related` (although there is room for improvement)

![Performance with prefetch_related](https://lh3.googleusercontent.com/ddIvjV00sukF2T1vHTCCbtHhHeuh1kcin-k2xbanLhO0wIVfdHuWOxB40W6Odw7a2h9rSPbNUm8Y7BG6K5cN-wwO1hD22Y2-A-USsE2-qTYBjzp1l7hy5zMO9Xp6kOAKoDJapzXb1bDOMjgblFuaiHYad9QVrLvA6XWQ0w5-FIfYfpVAdFYiAis3T9OhhC8spHpP2vUb8RkUw29W0eOg5Ju6zuYbfBDd1EsMfQySMTrtkplCB8an7Ji3iL_j8zrjdjlgzmw2N4HYNFE0iiGNBPVoYN0ko8HcOhoJv3FZw1r5ircNVv2LflzreXKmFijGs3pR4LzIMjyn2bJUbBAelEl15jhG8Nvy9VWUnW2kVo1EqqSU5OBTZSllWocrwvViiAz39LkoR9X29_WchPgE1WcHLs41KEK3dPP-FDzH9rfMUF7xmaMTz945LgNzx2XCcMz4NEmW_86gHmBGuuJcZs0t1ePVFBFfak4MJjqU3iTBiBjH_6dKDPrnwDJ9JGsWoKKtUuR8gyGHhmbKnjvPBJav1WTgYGG33dIDhTxeEfHGd3j-OJrOWvKv0GNdhsArZ9EJCMvC4PNkkVI2juKeRneMJaEJFNLu0c9Gk-y7_LHR-_ecDR3Q=w2286-h620-no)

In fact, the careful observer will notice that the called components in this
trace are a bit different.  Well, the `prefetch_related` was so successful,
that the same exact query no longer qualifies for being traced in New Relic and
the average dropped by half!

![average reservations api call](https://lh3.googleusercontent.com/J5GRb-47wTPwZ6NVYGMjtsnZpeoiNGqcVdVD-m620gns5aPdRYxYIUJLVHJ_QfHmf20twhlHOAWXwoFldLUYToQSx6QGPuOWR7XVwQOL4OeJG4HoQ_yI2FmeOse9TipkCf0rc80cM06jVTaXGaX9HMPZpsZUs6_dZKTjazAEGxND0iBxvRi2snmdrw6XT8z-4RPVV51PCODPqboquFKl7dfs-Gna3O3al8f_oguVBW0C_pioe7OGE-kuSCmF5gxC2fGdlDKxLm4Zv96b0j81kYhLzo20VUkjgisyc5kaLJrUpMrK9ySEbUbVaELnrpnZAF7XNeHeDutP5C0oR-eSJ5z6DReTOyhH4e52i5dD3vpb3M-Q1VBMJi3I7LTlhiG7fmIZno06bLsjSp8V0te6lWdcGmvBqKTLA1HFf_Kf3_9CZAp5g0JPtetMdwFt9LdTxzZcoykHHa1lDGsKOzMEYZN7kPm6uGIlsL-pVzag6c_mTq0DN3ARYzgA7KCAyWngsiNqpcjit0Q4XReuF3wfNpRFkZLMpuylsqzryQYl0yw354NaND8lRP3X8YT40hY-xKsnLWoTVncYLY3dkP0o0IiygXjAHDeW82N-fi_GrFXmaPSpju8k=w1260-h298-no)

I would like to caution that I believe this works so well in this case because we
limit the API page size, so, the data and query sizes in the related queries are
relatively small per request. If you need to load a lot of data at once, I can
foresee this solution being much worse than `select_related`.

TL;DR: Monitor and analyze your production queries and `prefetch_related` can
be a great solution if you can keep the number of queries small.