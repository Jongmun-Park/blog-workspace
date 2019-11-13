---
title: "Django: annotate, values, F function, order_by 등"
date: "2019-11-12"
template: "post"
draft: false
slug: "/posts/5"
category: "Django"
tags:
  - "Django"
  - "annotate"
  - "values"
  - "F function"
  - "order_by"
description: ""
---

Django ORM을 사용한지 어느덧 일년 가까이 됐다.
그치만 사용할 때마다 개념이 헷갈리는 QuerySet 들이 있다.

정리를 안 했기에, 그때 그때 쓰고 기억에 남진 않았던 것..
이제는 자주 헷갈렸던 것들은 꼭 정리하고 넘어가려고 한다.

- `annotate` : Annotates each object in the QuerySet with the provided list of query expressions. An expression may be a simple value, a reference to a field on the model (or any related models), or an aggregate expression (averages, sums, etc.) that has been computed over the objects that are related to the objects in the QuerySet.
Each argument to annotate() is an annotation that will be added to each object in the QuerySet that is returned.
SQL Query 에서 as와 비슷한 역할을 한다고 생각하면 될 듯 하다.
- `values` : Returns a QuerySet that returns dictionaries, rather than model instances, when used as an iterable.
Each of those dictionaries represents an object, with the keys corresponding to the attribute names of model objects.
내가 지정한 필드만 dictionary(key: value) 형태로 리턴된다.
- `F` function : An F() object represents the value of a model field or __annotated column__. It makes it possible to refer to model field values and perform database operations using them without actually having to pull them out of the database into Python memory.
가장 유용한 개념 중 하나라고 생각한다. 필드 혹은 annotated 필드의 값까지 데이터베이스에서 Python 메모리로 끌어낼 필요 없이 모델 필드 값을 참조하고 이를 사용하여 데이터베이스 작업을 수행할 수 있다.

### # annotate, values, distinct 활용해보기

```python
import pytz
from dateutil.parser import parse
from django.utils import timezone
from datetime import timedelta
tz = pytz.timezone('Asia/Seoul')
start_date = tz.localize(parse('2019-01-01'))
contents=[11592,11589,12681,12682]

freepass_users = UserVideoLecturePlayLog.objects.filter(
    user_content__content__in=contents, created_at__gte=start_date,
    user_content__parent__content__content_type=6, play_time__gt=timedelta(seconds=0)
    ).annotate(
        user_info_id = F('user_content__order__user_info_id'),  # annotate를 활용하여 join한 필드 간편하게 사용
        freepass_id = F('user_content__parent__content_id')
    ).values(
        'user_info_id', 'freepass_id'                           # values를 활용하여 내가 사용할 필드 data만 가져오기
    ).order_by('user_info_id', 'freepass_id')                   # order_by 사용하여 data 정렬

freepass_users.count()
# 253192
distinct_user_id = freepass_users.distinct('user_info_id').count()  # distinct에 사용하는 필드와 그 순서에 따라 data가 달라진다
# 6091
distinct_user_and_freepass = freepass_users.distinct('user_info_id', 'freepass_id').count()
# 6298

```


### # distinct + values_list 활용하여 중복 제거한 list 만들기

```python
distinct_trial_answer = TrialUserAnswer.objects.distinct(
  'user', 'question', 'answer'
  ).values_list('id', flat=True)
```

### # Sum, Case, When, F function 활용하여 유저별 총계 data 가져오기

```python
user_total_score_list = TrialUserAnswer.objects.filter(
    id__in=distinct_trial_answer,
    question__area__trial_time_id=trial_time_id
    ).values('user'
    ).annotate(
        total_score=Sum(Case(When(answer=F('question__answer'), then=F('question__score')),
        output_field=IntegerField()))
    ).order_by('-total_score')
```
