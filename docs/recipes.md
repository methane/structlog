# Recipes

## Integration with Frameworks

To have consistent log output, it makes sense to configure *structlog* *before* any logging is done.
The best place to perform your configuration varies with applications and frameworks.
If you use standard library's logging, it makes sense to configure them next to each other.


### Django

[*django-structlog*](https://pypi.org/project/django-structlog/) is a popular and well-maintained package that does all the heavy lifting.


### Flask

See Flask's [Logging docs](https://flask.palletsprojects.com/en/latest/logging/).

Generally speaking: put it *before* instantiating `flask.Flask`.


### Pyramid

[Application constructor](https://docs.pylonsproject.org/projects/pyramid/en/latest/narr/startup.html#the-startup-process>).


### Twisted

The [plugin definition](https://docs.twisted.org/en/stable/core/howto/plugin.html) is the best place.
If your app is not a plugin, put it into your [tac file](https://docs.twisted.org/en/stable/core/howto/application.html).

(custom-wrappers)=

## Custom Wrappers

```{eval-rst}
.. testsetup:: *

   import structlog
   structlog.configure(
       processors=[structlog.processors.KeyValueRenderer()],
   )
```

```{eval-rst}
.. testcleanup:: *

   import structlog
   structlog.reset_defaults()

```

The type of the *bound loggers* that are returned by {func}`structlog.get_logger()` is called the *wrapper class*, because it wraps the original logger that takes care of the output.
This wrapper class is [configurable](configuration.md).

Originally, *structlog* used a generic wrapper class {class}`structlog.BoundLogger` by default.
That class still ships with *structlog* and can wrap *any* logger class by intercepting unknown method names and proxying them to the wrapped logger.

Nowadays, the default is a {class}`structlog.typing.FilteringBoundLogger` that imitates standard library’s log levels with the possibility of efficiently filtering at a certain level (inactive log methods are a plain `return None` each).

If you’re integrating with {mod}`logging` or Twisted, you may was to use one of their specific *bound loggers* ({class}`structlog.stdlib.BoundLogger` and {class}`structlog.twisted.BoundLogger`, respectively).

—

On top of that all, you can also write your own wrapper classes.
To make it easy for you, *structlog* comes with the class {class}`structlog.BoundLoggerBase` which takes care of all data binding duties so you just add your log methods if you choose to sub-class it.

(wrapper-class-example)=

### Example

It’s easiest to demonstrate with an example:

```{eval-rst}
.. doctest::

   >>> from structlog import BoundLoggerBase, PrintLogger, wrap_logger
   >>> class SemanticLogger(BoundLoggerBase):
   ...    def info(self, event, **kw):
   ...        if not "status" in kw:
   ...            return self._proxy_to_logger("info", event, status="ok", **kw)
   ...        else:
   ...            return self._proxy_to_logger("info", event, **kw)
   ...
   ...    def user_error(self, event, **kw):
   ...        self.info(event, status="user_error", **kw)
   >>> log = wrap_logger(PrintLogger(), wrapper_class=SemanticLogger)
   >>> log = log.bind(user="fprefect")
   >>> log.user_error("user.forgot_towel")
   user='fprefect' status='user_error' event='user.forgot_towel'
```

You can observe the following:

- The wrapped logger can be found in the instance variable {attr}`structlog.BoundLoggerBase._logger`.
- The helper method {meth}`structlog.BoundLoggerBase._proxy_to_logger` that is a [DRY] convenience function that runs the processor chain, handles possible {class}`structlog.DropEvent`s and calls a named function on `_logger`.
- You can run the chain by hand through using {meth}`structlog.BoundLoggerBase._process_event` .

These two methods and one attribute are all you need to write own *bound loggers*.

[dry]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself