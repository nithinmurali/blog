---
layout: post
title: Django weekday catch!
---

Just a catch in django weekday, its incompalitable with standrad python weekday numbering. You can get weekday in django using `Sale.objects.get(sale_date__week_day=2)` or `ExtractWeekday`.

* `date.weekday` considers a weekday from 0 (Monday) to 6 (Sunday)
* `date.isoweekday()` considers a weekday from 1 (Monday) to 7 (Sunday)
* __Django__ interprets the integer as 1 (Sunday) to 7 (Saturday)
* MySQL and Oracle are identical to Django’s weekday value - 1 (Sunday) to 7 (Saturday)
* PostgreSQL - 0 (Sunday) to 6 (Saturday)
* SQLite is identical to Python’s isoweekday() representation - 1 (Monday) to 7 (Sunday)


A sample fuction to get django weekday from date :
```
def get_django_weekday(date):
    return (date.isoweekday() % 7) + 1
```

source: https://ana-balica.github.io/2017/08/14/django-week-day-field-lookup/
