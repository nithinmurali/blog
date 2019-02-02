---
layout: post
title: Notes on Django logging
---

This is a short article about django(drf) logging. This are basically what i figureed out while working on a project.

Before you read this, read about [python logging](https://medium.com/@imnmfotmal/logging-in-python-for-library-6e13b37b1930). In a production system dont use files for logging, it will slow down your system, use either syslog, of elk stack or something.

## The setting

The logs setting has 4 major components. Loggers, Handlers, Filters, Formatters.

### Loggers 
you can specifiy the loggers name here. only those loggers will be used
If you are using some custom logger like  `logger = logging.getLogger(__name__)`, then you have to put that name here, only then it will be logged. Eg
```
  'django': {
          'handlers': ['console', 'logfile'],
          'propagate': True,
          'level': 'INFO'
      },
```
Django has some internal logger already defined, which are child of the `django` logger. some examples are `django.request`, `django.server`etc. See more info [here](https://docs.djangoproject.com/en/2.1/topics/logging/#django). If you want to have logs from a specific app. you have just have that app name for logger name. If you dont specify any logger name, all the logs will be considered (see my config).

### Handlers
Each logger has one or mode handler associated. An handler defines where does the logs go. eg files, console, email etc.

### Filters
A filter is associated with an handler, the filter can basically choose what all to pass to an handler based on certain conditions, like pass only logs which has a specific string in them.

### Formatters
A handler has a formatter assciated with it. It can be used to specify the exact format of that text of the log record to be rendered.

## My Config

In each python file i have defined `logger = logging.getLogger(__name__)` on top. 

A samlple logging:

```
class TestView(views.APIView):

    def post(self, request):
        logger.debug("test debug log")
        logger.info("test info log")
        try:
            a=1/0
        except:
            logger.exception("test exception log")
            pass
        logger.error("test error log")
        logger.critical("test critical log")
        return custom_responses.response_ok()
```

settings.py

```
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '[%(asctime)s] %(levelname)s %(module)s [%(name)s.%(funcName)s:%(lineno)d] %(message)s',
        },
        'simple': {
            'format': '%(levelname)s %(message)s'
        },
    },
    'filters': {
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
    },
    'handlers': {
        'logfile': {  # Rotate log file daily, only keep 1 backup
            'level': 'DEBUG',
            'class': 'logging.handlers.TimedRotatingFileHandler',
            'filename': os.path.join(LOG_DIR, 'django.log'),
            'when': 'd',
            'interval': 7,
            'backupCount': 3,
        },
        'logfile_all': {  # Rotate log file daily, only keep 1 backup
            'level': 'INFO',
            'class': 'logging.handlers.TimedRotatingFileHandler',
            'filename': os.path.join(LOG_DIR, 'django_all.log'),
            'when': 'd',
            'interval': 7,
            'backupCount': 3,
            'formatter': 'verbose'
        },
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        }
        'slack_admins': {
            'level': 'ERROR',
            'filters': ['require_debug_false'],
            'class': 'config.slack_logger.SlackExceptionHandler',
            'formatter': 'simple'
        }
    },
    'loggers': {
        'django': { # only django logs
            'handlers': ['console', 'logfile'],
            'propagate': True,
            'level': 'INFO'
        },
        '': {  # will have the custom logs as well
            'handlers': ['logfile_all'],
            'level': 'INFO',
            'propagate': True,
        },

    }
}

```
