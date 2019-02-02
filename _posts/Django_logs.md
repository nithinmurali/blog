
First read about python logging.

## The setting

The logs setting has 4 major components. Loggers, Handlers, Filters, Formatters

Loggers - you can specifiy the loggers name here. only those loggers will be used
If you are using some custom logger like  `logger = logging.getLogger(__name__)`, then you have to put that name here, only then it 
will be logged. Eg
```
  'django': {
          'handlers': ['console', 'logfile'],
          'propagate': True,
          'level': 'INFO'
      },
```
Django has some internal logger already defined, which are child of the `django` logger. some examples are `django.request`, `django.server`
etc. See more info here.


Handlers - each logger has one or mode handler associated. An handler defines where does the logs go. eg files, console, email etc.
Filters - A filter is associated with an handler, the filter can basically choose what all to pass to an handler based on certain conditions, 
like pass only logs which has a specific string in them.
Formatters - A handler has a formatter assciated with it. It can be used to specify the exact format of that text of the log record to be rendered.
