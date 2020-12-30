---
date: 2017-04-02T00:00:00-07:00
summary: Should you remove data from the database or simply mark it as deleted?
tags:
- programming
- python
- django
title: Delete or not to delete
---

Should you remove data from the database or simply mark it as deleted?
At Teem we have a lot of data that we need to manage and often "physically"
deleting the data from disk can be problematic. Either the users simply wants
to undelete something or the deletion would cause problems for a log. The
generic solution to this problem is to soft delete/archive the data by adding
a `deleted_at` timestamp field to the table and then filter all queries to
hide rows that have been marked as deleted.


<!-- more /-->

This simple approach can take you a long way but doesn't fully cover all the
required scenarios. More specifically, what do you do with related objects?
About a year ago, I realized that we really needed to address the
entire problem. There are several pieces of data that we never want to delete,
e.g. devices and visitors among others. At the time I couldn't find a really
good solution so I wrote my own:
[django-archive-mixin](https://github.com/LucasRoesler/django-archive-mixin).


As of today, there are at least two other options that cover this situation as
well and you should checkout too:

* [django-softdelete](https://github.com/scoursen/django-softdelete), and
* [django-safedelete](https://github.com/makinacorpus/django-safedelete).

The biggest distinction between our project and the ones above is that I tried
to stay as close to the delete logic used in the original Django source code as
possible.  In particular, this means  mimicking Django's `collector` code that
implements the ORM cascade delete.  It is interesting to note that Django does
not rely on the database's cascade functionality but instead manages the delete
process itself. The benefit of doing this is that you can then specify behavior
at delete time via the `on_delete` argument to the model field.  For example:

```
class Car(models.Model):
    manufacturer = models.ForeignKey(
        'production.Manufacturer',
        blank=True, null=True
        on_delete=models.SET_NULL,
    )
```
will cause the manufacturer field to be set to `None` when you delete the
related manufacture. The default behavior would be to delete the car instance
as well.

[Django provides 6 on_delete
options](https://docs.djangoproject.com/en/1.10/ref/models/fields/#django.db.models.ForeignKey.on_delete):
`CASCADE`, `PROTECT`, `SET_NULL`, `SET_DEFAULT`, `SET()`, and `DO_NOTHING`.
At delete, the Django `collector` crawls the relationships and
buckets each object found into different lists depending on the `on_delete`
configuration for that specific relationship.  `CASCADE` puts the object in a
bucket to be deleted, `PROTECT` will cause an exception to be thrown,
`SET_NULL`, `SET_DEFAULT`, and `SET()` each cause and update to that instance,
and `DO_NOTHING` is a no-op. Once I understood this process, I decided to
piggyback on the process and add an additional piece of logic to put more
objects into the update bucket. Essentially, I allow Django to do all of the
collection for me and then I go through the list of objects to inspect if it
should be archived instead, if it is, I move it to the update bucket and move
on.

```
def cascade_archive(inst_or_qs, using, keep_parents=False):
    """
    Return collector instance that has marked ArchiveMixin instances for
    archive (i.e. update) instead of actual delete.

    Arguments:
        inst_or_qs (models.Model or models.QuerySet): the instance(s) that
            are to be deleted.
        using (db connection/router): the db to delete from.
        keep_parents (bool): defaults to False.  Determine if cascade is true.

    Returns:
        models.deletion.Collector: this is a standard Collector instance but
            the ArchiveMixin instances are in the fields for update list.
    """
    from .mixins import ArchiveMixin

    if not isinstance(inst_or_qs, models.QuerySet):
        instances = [inst_or_qs]
    else:
        instances = inst_or_qs

    deleted_ts = timezone.now()

    # The collector will iteratively crawl the relationships and
    # create a list of models and instances that are connected to
    # this instance.
    collector = models.deletion.Collector(using=using)
    if StrictVersion(django.get_version()) < StrictVersion('1.9.0'):
        collector.collect(instances)
    else:
        collector.collect(instances, keep_parents=keep_parents)
    collector.sort()

    for model, instances in collector.data.iteritems():
        # remove archive mixin models from the delete list and put
        # them in the update list.  If we do this, we can just call
        # the collector.delete method.
        inst_list = list(instances)

        if issubclass(model, ArchiveMixin):
            deleted_on_field = get_field_by_name(model, 'deleted_on')
            collector.add_field_update(
                deleted_on_field, deleted_ts, inst_list)

            del collector.data[model]

    for i, qs in enumerate(collector.fast_deletes):
        # make sure that we do archive on fast deletable models as
        # well.
        model = qs.model

        if issubclass(model, ArchiveMixin):
            deleted_on_field = get_field_by_name(model, 'deleted_on')
            collector.add_field_update(deleted_on_field, deleted_ts, qs)

            collector.fast_deletes[i] = qs.none()

    return collector
```
What I love about this logic is that is is a fairly small change to how
deletion works while also being fairly low-level enough that it covers all of
the deletion cases that the Django ORM handles.

We have been using this mixin for about a year now with no hiccups. It works as
expected and hasn't really needed much attention. If you are using Django and
have been looking for a safe delete/archive utility
[check it out](https://github.com/LucasRoesler/django-archive-mixin) and let me
know what you think.
