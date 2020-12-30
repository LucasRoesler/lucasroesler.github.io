---
date: 2017-02-06T20:00:00Z
title: Postgres Joins and Django Querysets
summary: Dealing with inefficient joins in Django's ORM
tags:
- programming
- django
- python
- postgres
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

{{<image src="select_related_performance.png" title="Example Performance with select_related">}}

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

{{<image src="prefetch_related_performance.png" title="Performance with prefetch_related">}}

In fact, the careful observer will notice that the called components in this
trace are a bit different.  Well, the `prefetch_related` was so successful,
that the same exact query no longer qualifies for being traced in New Relic and
the average dropped by half!

{{<image src="average_speed_up.png" title="Average reservations api call">}}

I would like to caution that I believe this works so well in this case because we
limit the API page size, so, the data and query sizes in the related queries are
relatively small per request. If you need to load a lot of data at once, I can
foresee this solution being much worse than `select_related`.

TL;DR: Monitor and analyze your production queries and `prefetch_related` can
be a great solution if you can keep the number of queries and the page size
small.
