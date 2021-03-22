---
id: c874cfd9-6e98-431b-a86a-b60d0cbf3a69
title: Fixing Copy Status
desc: "How I plan to solve the problem of properly setting the status field of the BookInstance model."
updated: 1616441281091
created: 1616269010379
---

## Workflow

In determining how I want to solve this problem I should first break down the workflow I envision for our `BookInstance` and `BorrowedCopy` records.

- `BorrowedCopy` records represent the borrowing history of each copy but in certain situations repsent the current state of a copy.
  - When `date_checked_out` has a value but `date_returned` does not, this indicates the copy is `ON_LOAN`.
  - When both `date_checked_out` and `date_returned` are `NULL` this implies a reservation for a given copy.
  - All creations of a `BorrowedCopy` record will be reservations (i.e. no `date_checked_out`) if the copy status is not `AVAILABLE`.
- Library Patrons can create `BorrowedCopy` record for a given `BookInstance` that is in the `AVAILABLE`, `ON_LOAN`, or `RESERVED` status.
- Only copies with a status of `AVAILABLE` can be checked out but reservation records can be created for `ON_LOAN` and `RESERVED` copies.
- Only Library Staff can change a `BookInstance` status from `ON_LOAN` to any other status. In effect, they are responsible for returning a book to circulation.
- The presence or absence of a `BorrowedCopy` record without `date_checked_out` value for a given copy determines if it returns to an `AVAILABLE` or `RESERVED` status upon being returned to circulation.

## Controlling `BookInstance` Status

Writing out the workflow is a great exercise and typing the list above helped clarify a few things for me. Initially I was thinking I could create an annotation that evaluates any related `BorrowedCopy` records for a given `BookInstance` and return this as the default for the status. While this approach does have the benefit of always being the most up to date (since it is caclulated each time a record is acccessed) it also increases the complexity of the SQL generated to fetch our record. In this scenario the `status` field would cease to exist on the `BookInstance` model per se and only be returned as a dynamically caculated value.

However, this approach cannot statisfy one of the requirements of the above workflow. That is that only Library staff can return a copy to circulation. This rules out the pure annotation approach since some changes in status are not controlled by related record values.

### Static Fields Approach

While we must rely on a static model field to get the functionality we want, we can still utilize a similar approach that moves more of the operations to the database layer. This will help make sure that the application can better scale as the library gets more popular or patrons getting into reservation wars for the most popular books.

I'll use Django's nifty [Conditional Update](https://docs.djangoproject.com/en/3.1/ref/models/conditional-expressions/#conditional-update) approach. This allows me to specificy various criteria in When clauses that will be evaluated in the database and which value should be written to the status field depending on which clause returns `True`.

Now where to put this little bit of logic? Well the returning to circulation operaton happens directly with the BookInstance model so it would seem that we need a model method to represent that action wherein we will place this logic. _However_ we also need to revise the logic in the `save()` method for the `BorrowedCopy` model to accurately reflect the intended workflow.

In order to avoid having to maintain multiple copies of the same logic I will use one of my favorite features of Django annotations. That is they can be defined and assigned to a variable, outside of calling the actual query. This means I can _hopefully_ create a single definition of this business logic and reuse it in multiple model methods.

### On Second Thought...

We'll need two versions of the conditional update annotation.

- `status_patrons`: The one for library patrons will determine if it should set the copy's status to `ON_LOAN` (checking out), `RESERVED` (placing on hold), or `MAINTENANCE` (returned and awaiting library staff).
- `status_staff`: The one for staff will determine if a copy should be set to `RESERVED` (if `BorrowedCopy` records without a checkout date exist) or `AVAILABLE` (if not) when it is returned to circulation (or taken out of `MAINTENANCE` status).

The benefit of this approach is we don't need to require either staff or patrons to know what the appropriate status is to set for the copy. Of course the front-end will determine whether to present a 'Check Out' or 'Reserve' button to the user (or both) for the patrons. For the staff, I will do a little bit of [customizing the default Django admin interface](https://docs.djangoproject.com/en/3.1/ref/contrib/admin/actions/) to allow staff to 'Return to Circulation' for multiple copies at once.

In this case I get to play with a few of the nifty advanced ORM features of Django. Let's take a look!

- First I'm using the [`OuterRef` to nest subqueries](https://docs.djangoproject.com/en/3.1/ref/models/expressions/#subquery-expressions) within the overall `Case` statement. I will use these subqueries to define each of statuses.

```python
"""
  catalog/annotations.py
"""

from django.db.models import OuterRef
from catalog.models import BorrowedCopy
# When using the OuterRef function, this annotation must be called as
# part of a query on the related object, BookInstance in this case.

# Define condition in which a copy is checked out.
checked_out = BorrowedCopy.objects.filter(
  copy=OuterRef('pk'),
  date_checked_out__isnull=False,
  date_returned__isnull=True
)
# Define condition in which a copy is reserved
reserved = BorrowedCopy.objects.filter(
  copy=OuterRef('pk'),
  date_checked_out__isnull=True
)
```

- Next we roll up each of these subqueries inside a Case statement. To make this simple as well as provide some potential query optimization, I will be applying the [`Exists` subquery class](https://docs.djangoproject.com/en/3.1/ref/models/expressions/#exists-subqueries) along with the rest of the `Case..When` functionality that are part of Django ORM's [conditional expressions](https://docs.djangoproject.com/en/3.1/ref/models/conditional-expressions/#) toolbox.

```python
"""
  catalog/annotations.py
"""
...
from django.db.models import Value, CharField, Case, When
from catalog.models import BookInstance

status_patrons = Case(
  When(Exists(checked_out), then=Value(BookInstance.ON_LOAN)),
  When(Exists(reserved), then=Value(BookInstance.RESERVED)),
  default=Value(BookInstance.MAINTENANCE),
  output_field=CharField()
)

status_staff = Case(
  When(Exists(reserved), then=Value(BookInstance.RESERVED)),
  default=Value(BookInstance.AVAILABLE),
  output_field=CharField()
)
```

The `When` clause expects a dynamic value as the `then` keyword argument so to get this working as expected we wrap our static status values in the `Value` class. This ensures we write back "o" for `ON_LOAN` rather than go looking for a field called "o".

The `Case` statement will evaluate each `When` clause in order so even if a book being checked out has other future reservations, the `status_patrons` clause will still return the character value designating the status as "on loan". This is another example of why I like to declare each statically define choice to it's own static variable on the model (take a look at the BookInstance definition [[here|localibrary.models.copy_borrowed]] if you need a refresher). It means there is a single source of truth when it comes to static values in the application.

Alright, I think I've got my subqueries and annotations worked out. Now to plug them into the application workflow.
