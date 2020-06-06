# Logger_tt
Make configuring logging simpler and log even exceptions that you forgot to catch.

## Install
* From PYPI: `pip install logger_tt`
* From Github: clone or download this repo then `python setup.py install` 
    
## Overview:

In the most simple case, add the following code into your main python script of your project:

```python
from logger_tt import setup_logging    

setup_logging()
```

Then from any of your modules, you just need to get a `logger` and start logging.

```python
from logging import getLogger

logger = getLogger(__name__)

logger.debug('Module is initialized')
logger.info('Making connection ...')
```


This will provide your project with the following **default** log behavior:

* log file: Assume that your `working directory` is `project_root`,
 log.txt is stored at your `project_root/logs/` folder. 
If the log path doesn't exist, it will be created. 
The log file is time rotated at midnight. A maximum of 15 dates of logs will be kept.
This log file's `level` is `DEBUG`.<br>
The log format is `[%(asctime)s] [%(name)s %(levelname)s] %(message)s` where time is `%Y-%m-%d %H:%M:%S`.<br>
Example: `[2020-05-09 00:31:33] [myproject.mymodule DEBUG] Module is initialized`

* console: log with level `INFO` and above will be printed to `stdout` of the console. <br>
The format for console log is simpler: `[%(asctime)s] %(levelname)s: %(message)s`. <br>
Example: `[2020-05-09 00:31:34] INFO: Making connection ...`

* `urllib3` logger: this ready-made logger is to silent unwanted messages from `requests` library.

* `root` logger: if there is no logger initialized in your module, this logger will be used with the above behaviors.
This logger is also used to log **uncaught exception** in your project. Example:

```python
raise RecursionError
```

```python
# log.txt
[2020-05-31 19:16:01] [root ERROR] Uncaught exception
Traceback (most recent call last):
  File "D:/MyProject/Echelon/eyes.py", line 13, in <module>
    raise RecursionError
     => var_in = Customer(name='John', member_id=123456)
     => arg = (456, 789)
     => kwargs = {'my_kw': 'hello', 'another_kw': 'world'}
RecursionError
```

* context logging: When an exception occur, variables used in the line of error are also logged.<br>
If the line of error is `raise {SomeException}`, then local variables are also logged.<br>
To always log full local variables, pass `full_context=True` to `setup_logging`.


## Usage:
All configs are done through `setup_logging` function:
```python
setup_logging(config_path="", log_path="", 
              capture_print=False, strict=False, guess_level=False,
              full_context=False)
```


1. You can overwrite the default log path with your own as follows:
    
   ```python
   setup_logging(log_path='new/path/to/your_log.txt')
   ```

2. You can config your own logger and handler by providing either `yaml` or `json` config file as follows:
    
   ```python
   setup_logging(config_path='path/to/.yaml_or_.json')
   ```

   Without providing a config file, the default config file with the above **default** log behavior is used.
   You could copy `log_conf.yaml` or `log_conf.json` shipped with this package to start making your version.

   **Warning**: To process `.yaml` config file, you need `pyyaml` package: `pip install pyyaml`

3. Capture stdout:

   If you have an old code base with a lot of `print(msg)` or `sys.stdout.write(msg)` and 
   don't have access or time to refactor them into something like `logger.info(msg)`, 
   you can capture these `msg` and log them to file, too.
   
   To capture only `msg` that is printed out by `print(msg)`, simply do as follows: 
    
   ```python
   setup_logging(capture_print=True)
   ```
   
   Example:
   ```python
   print('To be or not to be')
   sys.stdout.write('That is the question')
   ```
   
   ```
   # log.txt
   [2020-05-09 11:42:08] [PrintCapture INFO] To be or not to be
   ```
   
   <hr>
   
   Yes, `That is the question` is not captured. 
   Some libraries may directly use `sys.stdout.write` to draw on the screen (eg. progress bar) or do something quirk.
   This kind of information is usually not useful for users. But when you do need it, you can capture it as follows:
   
   ```python
   setup_logging(capture_print=True, strict=True)
   ```
   
   Example:
   ```python
   sys.stdout.write('The plane VJ-723 has been delayed')
   sys.stdout.write('New departure time has not been scheduled')
   ```
   
   ```
   # log.txt
   [2020-05-09 11:42:08] [PrintCapture INFO] The plane VJ-723 has been delayed
   [2020-05-09 11:42:08] [PrintCapture INFO] New departure time has not been scheduled
   ```
  
   <hr>
   
   As you have seen, the log level of the captured message is `INFO` . 
   What if the code base prints something like `An error has occurred. Abort operation.` and you want to log it as `Error`?
   Just add `guess_level=True` to `setup_logging()`.
   
   ```python
   setup_logging(capture_print=True, guess_level=True)
   ```
   
   Example:
   ```python
   print('An error has occurred. Abort operation.')
   print('A critical error has occurred during making request to database')
   ```
   
   ```
   # log.txt
   [2020-05-09 11:42:08] [PrintCapture ERROR] An error has occurred. Abort operation.
   [2020-05-09 11:42:08] [PrintCapture CRITICAL] A critical error has occurred during making request to database
   ```
   
   **Note**: Capturing stdout ignores messages of `blank line`. 
   That means messages like `\n\n` or `  `(spaces) will not appear in the log. 
   But messages that contain blank line(s) and other characters will be fully logged.
   For example, `\nTo day is a beautiful day\n` will be logged as is.  

4. Exception logging:
   
   Consider the following error code snippet:
   
   ```python
   API_KEY = "asdjhfbhbsdf82340hsdf09u3ionf98230234ilsfd"
   TIMEOUT = 60
   
   class MyProfile:
       def __init__(self, name):
           self.my_boss = None
           self.name = name

   def my_faulty_func(my_var, *args, **kwargs):
       new_var = 'local scope variable'
       me = MyProfile('John Wick')
       boss = MyProfile('Winston')
       me.my_boss = boss
       print(f'Information: {var} and {me.my_boss.name} at {me.my_boss.location} with {API_KEY}')
   
   if __name__ == '__main__':
       cpu_no = 4
       max_concurrent_processes = 3
       my_faulty_func(max_concurrent_processes, 'ryzen 7', freq=3.4)
   ```
   
   In our hypothetical code above,`print` function will raise an exception. 
   This exception, by default, will not only be logged but also analyzed with objects that appeared in the line:
   
   ```python
   [2020-06-06 09:36:01] ERROR: Uncaught exception:
   Traceback (most recent call last):
     File "D:/MyProject/AutoBanking/main.py", line 31, in <module>
       my_faulty_func(max_concurrent_processes, 'ryzen 7', freq=3.4)
        -> my_faulty_func = <function my_faulty_func at 0x0000023770C6A288>
        -> max_concurrent_processes = 3
   
     File "D:/MyProject/AutoBanking/main.py", line 25, in my_faulty_func
       print(f'Information: {var} and {me.my_boss.name} at {me.my_boss.location} with {API_KEY}')
        -> me.my_boss.name = 'Winston'
        -> me.my_boss.location = '!!! Not Exists'
        -> (outer) API_KEY = 'asdjhfbhbsdf82340hsdf09u3ionf98230234ilsfd'
   NameError: name 'var' is not defined
   ```
   
   For each level in the stack, any object that appears in the error line is shown with its `readable representation`.
   This representation may not necessarily be `__repr__`. The choice between `__str__` and `__repr__` are as follows:
   * `__str__` : `__str__` is present and the object class's `__repr__` is default with `<class name at Address>`.
   * `__repr__`: `__str__` is present but the object class's `__repr__` is anything else, such as `ClassName(var=value)`.<br>
   Also, when `__str__` is missing, even if `__repr__` is `<class name at Address>`, it is used.
   
   Currently, if an object doesn't exist and is directly accessed, as `var` in this case, it will not be shown up.
   But if it is attribute accessed with dot `.`, as `location` in `me.my_boss.location`, 
   then its value is an explicit string `'!!! Not Exists'`.
   
   As you may have noticed, a variable `API_KEY` has its name prefixed with `outer`. <br>
   This tells you that the variable is defined in the outer scope, not local. 
   
   More often than not, only objects in the error line are not sufficient to diagnose what has happened.
   You want to know what the inputs of the function were. You want to know what the intermediate 
   calculated results were. You want to know other objects that appeared during runtime,
   not only local but also outer scope. In other words, you want to know the full context of what has happened.
   `logger-tt` is here with you:
   
   ```python
   setup_logging(full_context=True)
   ```
   
   With the above hypothetical code snippet, the error log becomes the following:
   
   ```python
   [2020-06-06 10:35:21] ERROR: Uncaught exception:
   Traceback (most recent call last):
     File "D:/MyProject/AutoBanking/main.py", line 31, in <module>
       my_faulty_func(max_concurrent_processes, 'ryzen 7', freq=3.4)
        -> my_faulty_func = <function my_faulty_func at 0x0000019E3599A288>
        -> max_concurrent_processes = 3
          => __name__ = '__main__'
          => __doc__ = None
          => __package__ = None
          => __loader__ = <_frozen_importlib_external.SourceFileLoader object at 0x0000019E35840E48>
          => __spec__ = None
          => __annotations__ = {}
          => __builtins__ = <module 'builtins' (built-in)>
          => __file__ = 'D:/MyProject/AutoBanking/main.py'
          => __cached__ = None
          => setup_logging = <function setup_logging at 0x0000019E35D111F8>
          => getLogger = <function getLogger at 0x0000019E35BC7C18>
          => logger = <Logger __main__ (DEBUG)>
          => API_KEY = 'asdjhfbhbsdf82340hsdf09u3ionf98230234ilsfd'
          => TIMEOUT = 60
          => MyProfile = <class '__main__.MyProfile'>
          => cpu_no = 4
   
     File "D:/MyProject/AutoBanking/main.py", line 25, in my_faulty_func
       print(f'Information: {var} and {me.my_boss.name} at {me.my_boss.location} with {API_KEY}')
        -> me.my_boss.name = 'Winston'
        -> me.my_boss.location = '!!! Not Exists'
        -> (outer) API_KEY = 'asdjhfbhbsdf82340hsdf09u3ionf98230234ilsfd'
          => my_var = 3
          => args = ('ryzen 7',)
          => kwargs = {'freq': 3.4}
          => new_var = 'local scope variable'
          => me = <__main__.MyProfile object at 0x0000019E35D3BA48>
          => boss = <__main__.MyProfile object at 0x0000019E35D3B9C8>
   NameError: name 'var' is not defined
   ```
   
   Additional objects that not appear in the error line are prefixed with `=>`.
   


# Sample config:

1. Yaml format:

   ```yaml
   version: 1
   disable_existing_loggers: False
   formatters:
     simple:
       format: "[%(asctime)s] [%(name)s %(levelname)s] %(message)s"
       datefmt: "%Y-%m-%d %H:%M:%S"
     brief: {
       format: "[%(asctime)s] %(levelname)s: %(message)s"
       datefmt: "%Y-%m-%d %H:%M:%S"
   handlers:
     console:
       class: logging.StreamHandler
       level: INFO
       formatter: simple
       stream: ext://sys.stdout
   
     error_file_handler:
       class: logging.handlers.TimedRotatingFileHandler
       level: DEBUG
       formatter: simple
       filename: logs/log.txt
       backupCount: 15
       encoding: utf8
       when: midnight
   
   loggers:
     urllib3:
       level: WARNING
       handlers: [console, error_file_handler]
       propagate: no
   
   root:
     level: DEBUG
     handlers: [console, error_file_handler]
   ```

<br>
2. Json format:

   ```json
   {
     "version": 1,
     "disable_existing_loggers": false,
     "formatters": {
       "simple": {
         "format": "[%(asctime)s] [%(name)s %(levelname)s] %(message)s",
         "datefmt": "%Y-%m-%d %H:%M:%S"
       },
       "brief": {
         "format": "[%(asctime)s] %(levelname)s: %(message)s",
         "datefmt": "%Y-%m-%d %H:%M:%S"
       }
     },
   
     "handlers": {
       "console": {
         "class": "logging.StreamHandler",
         "level": "INFO",
         "formatter": "brief",
         "stream": "ext://sys.stdout"
       },
   
       "error_file_handler": {
         "class": "logging.handlers.TimedRotatingFileHandler",
         "level": "DEBUG",
         "formatter": "simple",
         "filename": "logs/log.txt",
         "backupCount": 15,
         "encoding": "utf8",
         "when": "midnight"
       }
     },
   
     "loggers": {
       "urllib3": {
         "level": "ERROR",
         "handlers": ["console", "error_file_handler"],
         "propagate": false
       }
     },
   
     "root": {
       "level": "DEBUG",
       "handlers": ["console", "error_file_handler"]
     }
   }
   ```

# changelog
##1.2.0
* Add logging context for uncaught exception. Now automatically log variables surrounding the error line, too.
* Add test cases for logging exception

##1.1.1
* Fixed typos and grammar
* Add config file sample to README
* using full name `log_config.json` instead of `log_conf.json`, the same for yaml file 
* add test cases for `capture print`

##1.1.0
* Add `capture print` functionality with `guess level` for the message.