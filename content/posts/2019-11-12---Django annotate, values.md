---
title: "Django: annotate, values, F Object 등"
date: "2019-11-12"
template: "post"
draft: false
slug: "/posts/5"
category: "Django"
tags:
  - "Django"
  - "annotate"
  - "values"
  - "F Object"
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
- `F` object : An F() object represents the value of a model field or __annotated column__. It makes it possible to refer to model field values and perform database operations using them without actually having to pull them out of the database into Python memory.
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
        user_info_id = F('user_content__order__user_info_id'),
        # annotate를 활용하여 join한 필드 간편하게 사용
        freepass_id = F('user_content__parent__content_id')
    ).values(
        'user_info_id', 'freepass_id'  # values를 활용하여 내가 사용할 필드 data만 가져오기
    ).order_by('user_info_id', 'freepass_id')  # order_by 사용하여 data 정렬

freepass_users.count()
# 253192

# distinct에 사용하는 필드와 그 순서에 따라 data가 달라진다
distinct_user_id = freepass_users.distinct('user_info_id').count()
# 6091
distinct_user_and_freepass = freepass_users.distinct('user_info_id', 'freepass_id').count()
# 6298

```

## 유저별 시험 점수 data 만들기

프론트 단의 실수로 유저 데이터가 중복으로 입력되는 경우가 발생했다.
이 때문에 통계 데이터를 만들 때, 중복 제거를 해야만 했다.
작업 중 아래와 같은 에러를 만났다.

```python
user_scores = TrialUserAnswer.objects.filter(
        question__area__type=2, question__trial_id=trial_id
    ).distinct('user', 'question', 'answer'
    ).values('user', 'question__area', 'question__area__trial_time'
    ).annotate(
        correct_count = Sum(Case(
            When(answer=F('question__answer'), then=1),
            output_field=IntegerField())),
    ).order_by('user', 'question__area')
```

`NotImplementedError: annotate() + distinct(fields) is not implemented`

`distinct`한 `question` field를 `annotate`에서 사용할 수 없다는 에러로 보인다.

이를 해결하고 데이터를 추출하기 위해서 아래와 같은 방법을 사용할 수 있었다.
먼저, distinct + values_list 활용하여 중복 제거한 list 만든다.

```python
distinct_trial_answers = TrialUserAnswer.objects.filter(
        question__area__type=2, question__trial_id=trial_id
    ).distinct('user', 'question', 'answer').values_list('id', flat=True)
```

그리고 `distinct_trial_answers`에 담긴 id list로 필터링한 유저별 `correct_count`를 계산한다.

```python
user_scores = TrialUserAnswer.objects.filter(
        id__in = distinct_trial_answers
    ).values('user', 'question__area', 'question__area__trial_time'
    ).annotate(
        correct_count = Sum(Case(
            When(answer=F('question__answer'), then=1),
            output_field=IntegerField())),
    ).order_by('user', 'question__area')
```