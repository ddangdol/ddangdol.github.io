---
layout: posts
title: django LOGGING 의 disable_existing_loggers 동작 방식에 대한 이해
tags: [gunicorn, logging, django]
sitemap :
  priority : 1.0
---

## django logging ##
django는 python의 builtin 로깅 모듈을 사용하여 시스템 로깅 작업을 수행합니다. `settings.py` LOGGING을 통해 설정이 가능합니다. `setting.py` 에 `LOGGING` 설정이 없을 경우 아래 예시처럼 default 설정을 제공합니다.
```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'filters': {
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'formatters': {
        'django.server': {
            '()': 'django.utils.log.ServerFormatter',
            'format': '[{server_time}] {message}',
            'style': '{',
        }
    },
    'handlers': {
        'console': {
            'level': 'INFO',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
        },
        'django.server': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'django.server',
        },
        'mail_admins': {
            'level': 'ERROR',
            'filters': ['require_debug_false'],
            'class': 'django.utils.log.AdminEmailHandler'
        }
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'mail_admins'],
            'level': 'INFO',
        },
        'django.server': {
            'handlers': ['django.server'],
            'level': 'INFO',
            'propagate': False,
        },
    }
}
```
위 설정을 간단히 설명하자면 `django` 와 `django.server` 두 개의 logger 를 기본으로 사용하도록 설정되어있습니다. `django` logger 의 경우 root 를 제외한 최상위 logger 이며 `console` handler 를 사용하게 되는데 `debug=True` 일 경우만 동작하도록 `require_debug_true` Filter 를 사용하고 있습니다. 

`django.server` logger 는 django 의 요청처리와 관련된 로그 메시지를 기록하며 `propagate` 값이 False 이므로 상위 logger 인 `django` 에 전파되지 않습니다.

---

## django logger 종류 ##
`django` 는 여러 상황에 대한 로그메시지를 구분하고 기록하기 위해 기본적으로 아래 logger 들을 제공합니다. 자세한 내용은 [Django logging extensions 문서](https://docs.djangoproject.com/en/3.2/topics/logging/#django-s-logging-extensions)를 통해 확인합니다. 여기서는 간단하게 설명합니다.

### django ###
django framework 에서 로그 메시지를 기록하는데 직접적으로 사용되지는 않지만 아래 logger 들을 모두 포함하는 최상위 logger 입니다.

### django.request ###
요청을 처리와 관련된 로그 메시지 입니다. 5XX 응답은 `ERROR` 레벨로 기록되며, 4XX 응답은 `WARNING` 레벨로 기록됩니다. `django.security` logger 에 기록되는 내용은 `django.request` 로거에 중복으로 기록되지 않습니다.

### django.server ###
django `runserver command` 를 통해 구동되었을 경우 기록되는 logger 이며 요청 처리와 관련된 로그메시지를 기록합니다. 5XX 응답은 `ERROR` 레벨로 기록되며, 4XX 응답은 `WARNING` 레벨로 기록됩니다. 그 외 모든 로그메시지는 `INFO` 레벨로 기록됩니다.

### django.template ###
django template 랜더링과 관련된 로그 메시지입니다.

### django.db.backends ###
요청에 의해 실행되는 모든 애플리케이션 레벨의 SQL 문을 `DEBUG` 레벨로 기록합니다. 성능상의 이유로 설정된 logging 레벨과 관계없이 `DEBUG=True` 일 경우만 활성화됩니다.

### django.security.* ###
보안관련 로그메시지를 기록합니다. 자세한 내용은 위 django 문서 링크를 확인합니다. 관련된 내용은 추후 자세히 다룰 예정입니다.

### django.db.backends.schema ###
[migration frmaework](https://docs.djangoproject.com/en/3.2/topics/migrations/) 에 의해 실행된 데이터베이스의 스키마 변경 SQL 쿼리들을 기록합니다.

---

## diable_existing_loggers ##
`disable_existing_loggers` 값이 `True` 일 경우 해당 LOGGING 설정이 적용될 때 이미 존재하는 로거들을 비활성화 시키게 됩니다. `True` 로 설정한 경우 gunicorn 같은 WSGI 서버들의 logging 설정에 영향을 줄 수 있습니다. 예를 들어 아래 django LOGGING 설정을 사용한 app을 gunicorn 통해 실행한다고 가정해봅시다.
```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': True, # 위 설정에서 해당 값만 True 로 변경
    'filters': {
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'formatters': {
        'django.request': {
            '()': 'django.utils.log.ServerFormatter',
            'format': '[{server_time}] {message}',
            'style': '{',
        }
    },
    'handlers': {
        'console': {
            'level': 'INFO',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
        },
        'django.request': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'django.request',
        },
        'mail_admins': {
            'level': 'ERROR',
            'filters': ['require_debug_false'],
            'class': 'django.utils.log.AdminEmailHandler'
        }
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'mail_admins'],
            'level': 'INFO',
        },
        'django.request': {
            'handlers': ['django.request'],
            'level': 'INFO',
            'propagate': False,
        },
    }
}
```
### gunicorn logger ###
gunicorn 은 `gunicorn.access`, `gunicorn.error` 두 종류의 logger 를 사용합니다. gunicorn 은 `master` 와 `worker` process 로 구성됩니다. `master` process 는 `worker` 를 관리하는 역할을 하고 `worker` 가 실직적으로 django app 이 로드되는 process 입니다. 

`master` 의 경우 gunicorn logging 설정을 그대로 사용하므로 로드메시지가 기록되는데 문제가 없습니다. `worker` 의 경우 위 최초에 gunicorn logging 설정을 적용한 후 django app 이 등록되는 타이밍에 django logging 설정이 적용됩니다. 이 때 django 관련 logger 는 활성화되나 gunicorn 의 `gunicorn.access`, `gunicorn.error` 로거가 `disabled` 되어 로그메시지가 기록되지 않는 상황이 발생합니다.

master process 는 django app 을 로드하지 않으므로 `gunicorn.access`, `gunicorn.error` 로그메시지가 기록됩니다.

결국 아래와 같이 worker 의 `disabled` 된 로거들이 존재하게 됩니다.
![disable_existing_logger 과 활성화된 gunicorn logger 예시](/assets/images/disable-existing-loggers-for-django/gunicorn_logger.png)

---

## 결론 ##
대부분의 경우 disable_existing_loggers 값을 True 로 설정할 필요는 없으나 만약 필요한 경우 반드시 동작방식을 이해하고 사용하도록 합니다.

---

## 참고 ##
* [https://docs.djangoproject.com/en/3.2/topics/logging/](https://docs.djangoproject.com/en/3.2/topics/logging/)
* [https://docs.python.org/ko/3/howto/logging.html](https://docs.python.org/ko/3/howto/logging.html)
* [https://github.com/benoitc/gunicorn](https://github.com/benoitc/gunicorn)
