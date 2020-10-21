# Django, Celery, Redis를 이용한 비동기처리

![d_c_r](https://user-images.githubusercontent.com/26461307/96726540-0a8b2480-13ed-11eb-9241-0fda130421ec.png)

#### Celery 란?
- `Celery`는 안 보이는 곳에서 열심히 일하는 (백그라운드)일꾼이다. 처리해야 할 일을 Queue로 쌓아둔다. `큐`(queue)에 쌓인 일을 일꾼들이 가져다가 열심히 일을한다. 파이썬 언어로 작성되어 있다.
#### Redis 란?
- `Redis`는 실제 컴퓨터 메모리를 이용한 캐쉬다. `Key`와 `Value`값을 이용해 처리할 작업을 Celery에게 보낸 다음 `캐쉬` 시스템에서 해당 키를 없애는 방식으로 동작한다.

  좋은 점은 로컬과 DB사이에서 자료가 왔다갔다 하는 것보다 메모리에서 캐쉬를 가져다 쓰는 것이 훨씬 빠르다는 것이다. 따라서 특정 데이터를 반복적으로 돌려줘야 한다면 메모리 캐쉬를 사용하면 좋다.
###### [출처](https://whatisthenext.tistory.com/127)



## 1. 설치
Celery 설치
``` terminal
pip install 'celery[redis]'

pip install django-celery-beat
pip install django-celery-results
```

Redis 설치
``` termianl
brew install wget # wget이 없을 시

wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
redis-server # redis 실행

# 다른 터미널로 정상적으로 실행됐는지 확인할 때
redis-cli ping
# PONG 이 나옴
```

## 2. Django에 적용
#### [프로젝트 이름]/_\_init__.py
``` python
from __future__ import absolute_import, unicode_literals

from .celery import app as celery_app

__all__ = ('celery_app',
```

#### [프로젝트 이름]/celery.py (생성)
``` python
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')
app = Celery('myproject')

app.config_from_object('django.conf:settings', namespace='CELERY')

app.autodiscover_tasks()

@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

#### [프로젝트 이름]/settings.py
``` python
CELERY_BROKER_URL = 'redis://localhost:6379'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_RESULT_BACKEND = 'redis://localhost:6379'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TASK_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Asia/Seoul'

INSTALLED_APPS += (
    'django_celery_beat',
    'django_celery_results',
)
```

#### [앱 이름]/tasks.py (생성)
``` python
from __future__ import absolute_import, unicode_literals
from celery.decorators import task

@task(name="x_plus_y") # 예시
def x_plus_y(x, y):
    return x + y
```


## 3. 실행
1. redis 실행
``` terminal
redis-server
```
2. celery 실행
``` terminal
# 프로젝트가 있는 dir에서
celery -A [프로젝트명] worker -l info
```
3. task 실행
``` terminal
# 프로젝트가 있는 dir에서
python manage.py shell
>>> from proj.celery import debug_task
>>> debug_task.delay()
# celery를 실행한 terminal에서 log가 보일 것임
```


## 4. 내 서비스에 적용하기
##### 특정 시간마다 View에 있는 함수를 실행시키기 위함
###### <i>해당 함수는 크롤링한 정보를 바탕으로 약 10만개의 데이터들 중에 보유중인 데이터를 제외하고 모델에 저장하는 기능을 함 </i>
``` python
# tasks.py
...

from .views import save_test, lotteproduct

@task(name="save_temp")
def save_temp():
    return save_test()

@task(name="save_lottedata")
def save_lottedate():
    return lotteproduct()


# celery.py
...

app.conf.beat_schedule = {
    'add-every-100-seconds': {
        'task': 'save_temp', # 해당 name의 task를 실행
        'schedule': 100.0,  # 100초마다
    },
}
```
##### redis, celery worker을 실행 후
``` terminal
celery -A [프로젝트명] beat -l info
```

## 5. [실행결과](https://youtu.be/wAfYfT6tNiI)




<iframe width="560" height="315" src="https://www.youtube.com/embed/wAfYfT6tNiI" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
