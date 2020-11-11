# A sample django project to confirm `force_insert` option of `save()`

## Steps

### 1. Clone this app

```
$ git clone ...
```

### 2. Run migration

```
$ python manage.py migrate
```

### 3. Modify table definition by hand

Because Django's model doesn't allow multi-column primary key, it is necessary to confirm what SQL does `save()` without `force_insert` option execute.

```
$ sqlite3 db.sqlite
SQLite version 3.28.0 2019-04-15 14:49:49
Enter ".help" for usage hints.
sqlite> BEGIN;
sqlite> DROP TABLE "polls_choice";
sqlite> CREATE TABLE "polls_choice" ("question_id" integer NOT NULL, "choice_text" varchar(200) NOT NULL, "votes" integer NOT NULL, PRIMARY KEY ("question_id", "votes"));
sqlite> COMMIT;
sqlite>
```

### 4. Run django shell

```
$ python manage.py shell
```

### 5. Save data (1): without `force_insert` option

- `Choice` model is defined to have multi-column primary key: `question_id` and `votes`.
- All the object `c` in for loop have same `question_id` value.
- Without `force_insert=True`, `save()` executes `UPDATE` first, then executes `INSERT` if `UPDATE` doesn't update any rows.
- As you can see the console log below, there are 3 `UPDATE` and only 1 `INSERT`, while I expected 3 `INSERT`.

```
❯ python manage.py shell
Python 3.6.10 (default, Nov  2 2020, 21:46:18)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.16.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from polls.models import *

In [3]: from django.utils import timezone

In [4]: from datetime import datetime

In [5]: q = Question(question_text='hi', pub_date=datetime.now(tz=timezone.utc))

In [6]: q.save()
(0.003) INSERT INTO "polls_question" ("question_text", "pub_date") VALUES ('hi', '2020-11-11 13:18:35.736447'); args=['hi', '2020-11-11 13:18:35.736447']

In [8]: for i in range(3):
   ...:     c = Choice(question=q, choice_text='hi', votes=i)
   ...:     c.save()
   ...:
   ...:
(0.000) UPDATE "polls_choice" SET "choice_text" = 'hi', "votes" = 0 WHERE "polls_choice"."question_id" = 3; args=('hi', 0, 3)
(0.001) INSERT INTO "polls_choice" ("question_id", "choice_text", "votes") SELECT 3, 'hi', 0; args=(3, 'hi', 0)
(0.001) UPDATE "polls_choice" SET "choice_text" = 'hi', "votes" = 1 WHERE "polls_choice"."question_id" = 3; args=('hi', 1, 3)
(0.002) UPDATE "polls_choice" SET "choice_text" = 'hi', "votes" = 2 WHERE "polls_choice"."question_id" = 3; args=('hi', 2, 3)

In [9]: Choice.objects.count()
(0.000) SELECT COUNT(*) AS "__count" FROM "polls_choice"; args=()
Out[9]: 1

```

### 6. Clear data

```
Question.objects.all().delete()
Choice.objects.all().delete()
```

### 7. Save data (1): with `force_insert` option

- Opposite to step 5, now 3 `INSERT` and 0 `UPDATE` executed.

```
❯ python manage.py shell
Python 3.6.10 (default, Nov  2 2020, 21:46:18)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.16.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from polls.models import *

In [2]: from django.utils import timezone

In [3]: from datetime import datetime

In [4]: q = Question(question_text='hi', pub_date=datetime.now(tz=timezone.utc))

In [5]: q.save()
(0.002) INSERT INTO "polls_question" ("question_text", "pub_date") VALUES ('hi', '2020-11-11 13:34:55.806181'); args=['hi', '2020-11-11 13:34:55.806181']

In [6]: for i in range(3):
   ...:     c = Choice(question=q, choice_text='hi', votes=i)
   ...:     c.save(force_insert=True)
   ...:
(0.003) INSERT INTO "polls_choice" ("question_id", "choice_text", "votes") SELECT 7, 'hi', 0; args=(7, 'hi', 0)
(0.002) INSERT INTO "polls_choice" ("question_id", "choice_text", "votes") SELECT 7, 'hi', 1; args=(7, 'hi', 1)
(0.001) INSERT INTO "polls_choice" ("question_id", "choice_text", "votes") SELECT 7, 'hi', 2; args=(7, 'hi', 2)

In [7]: Choice.objects.count()
(0.000) SELECT COUNT(*) AS "__count" FROM "polls_choice"; args=()
Out[7]: 3

```
