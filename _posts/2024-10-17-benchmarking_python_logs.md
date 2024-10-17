---
layout: post
title: "Benchmarking Python Logs: An Investigation Into Efficient Logging in Python"
date: 2024-10-17 0:00:00
---

# An Investigation Into Efficient Logging in Python

I recently wrapped up work a guest contributor to a new team, which has been a lot of fun but there's also a bit of awkwardness; learning new ways of working, PR and deployment process, code style, etc.  In cases like these, I like to be a sponge and do my best to soak up as much institutional knowledge as I can.  I like to think of myself as the sort of developer who doesn't attach his ego to the work he does.  Maybe being a little more realistic with myself, I prefer to not *show* how much ego I attach to the work that I do.  Regardless of my internal motivations, I've found that it pays to be patient and thoughtful.  You might even learn something!

My new team owns a Flask application, a microservice which handles a lot of traffic from other services.  In the process of working through my first significant feature I found that in addition to f-strings, the developers used percent-formatting strings.  This isn't uncommon.  F-strings were only introduced in Python 3.6, and the version of this project is pinned to 3.10.  The earliest version I could find in the history is 3.9, so it's not surprising that folks were still in the habit of using the older %-formatting and `str.format()` methods.

When I put up my PR, one piece of feedback I received was that the team preferred to use f-string formatting in all places except for logs.  The reasoning was that f-strings were not efficient in log statements.

While most of my software development experience is in Python, I've been working in Ruby for the last couple years. I've not yet had the opportunity to work in an environment where I've had to squeeze every bit of performance from logs.  I was intrigued by this proposition and considered it a good opportunity to deepen my understanding.

## Understanding Python's Logging Mechanism

Flask doesn't implement its own logger, rather it relies on [logging.Logger](https://github.com/python/cpython/tree/3.12/Lib/logging) which is a part of Python ([source](https://github.com/pallets/flask/blob/main/src/flask/logging.py#L58-L79)):
```
def create_logger(app: App) -> logging.Logger:
    """Get the Flask app's logger and configure it if needed.

    The logger name will be the same as
    :attr:`app.import_name <flask.Flask.name>`.

    When :attr:`~flask.Flask.debug` is enabled, set the logger level to
    :data:`logging.DEBUG` if it is not set.

    If there is no handler for the logger's effective level, add a
    :class:`~logging.StreamHandler` for
    :func:`~flask.logging.wsgi_errors_stream` with a basic format.
    """
    logger = logging.getLogger(app.name)

    if app.debug and not logger.level:
        logger.setLevel(logging.DEBUG)

    if not has_level_handler(logger):
        logger.addHandler(default_handler)

    return logger
```

I was told that using f-string interpolation introduced inefficiencies in the logging code.  How could that be?

### Maybe it's an issue of log levels?

My mind first jumped to log levels as a possible scenario.  Maybe this codebase has a lot of `debug` logger statements, and if the application runs at `info` and higher, maybe there's some added drag in creating the messages for those unemitted statements? 

Say you've got some code like this:

```
import logging


def resource_intensive_function_call():
    return 5 * 10


def should_not_be_called():
    print(resource_intensive_function_call())
    return "should not see this message"


def main():
    logging.basicConfig(level=logging.INFO)
    logger = logging.getLogger(__name__)
    logger.debug("%i", should_not_be_called())
    logger.debug("debug level log message")


if __name__ == "__main__":
    main()
 
```

The result:
```
>> python3 loglevel.py
50
```

This makes sense, because we compose the result of `should_not_be_called` and pass it as an argument to the `debug` function.  It would not matter if we wrote the code with percent-formatting or with f-strings, like this:
`logger.debug('f{should_not_be_called()}')`.  Either way, we must create the string before we can pass it as an argument to the `debug` function.  

So the issue can't be about log levels.  Not *directly*, anyhow.

### Could it be an issue with the actual string interpolation?

Maybe the point my new team has isn't that a %-formatted string will prevent resource intensive functions from being called, but that *the act of interpolating the string with the value* can be prevented.

Let's take a look at the logger code:
```
class Logger(Filterer):   

  ...

  def debug(self, msg, *args, **kwargs):
    """
    Log 'msg % args' with severity 'DEBUG'.

    To pass exception information, use the keyword argument exc_info with
    a true value, e.g.

    logger.debug("Houston, we have a %s", "thorny problem", exc_info=True)
    """
    if self.isEnabledFor(DEBUG):
      self._log(DEBUG, msg, args, **kwargs)

class LogRecord(object):

  ...

  def getMessage(self):
    """
    Return the message for this LogRecord.

    Return the message for this LogRecord after merging any user-supplied
    arguments with the message.
    """
    msg = str(self.msg)
    if self.args:
      msg = msg % self.args
    return msg
```

There's a LOT of code in between the `debug` function call and the `getMessage` function call.  But the point is that if we have logging set to `INFO` and above, `Logger.isEnabledFor(DEBUG)` will return `False` and we will never end up calling `getMessage` and therefore *never interpolating the string*.  We can say that the log message is "lazily interpolated" because we only apply the interpolation in `getMessage` (as opposed to passing a fully interpolated string as an argument).  Now we're getting somewhere!  If a developer added a bunch of `debug` log statements, interpolating the strings to create them won't happen unless the log message is definitely going to be emitted.

## Is the juice worth the squeeze?

We've established the fact that using a log statement like this:

`logger.debug("interpolating a value right here: %s", "foo")`

is different than doing it like this: `logger.debug(f"interpolating a value right here: {'foo'}")` 

or even this: `logger.debug("interpolating a value right here: %s" % "foo")` (subtle difference!)

because the logger won't apply the function:
`msg = msg % self.args` 
if the log level is below the current threshold.  But how much performance gain can we actually get from doing this? Let's find out.

```
import logging
import timeit

# set up the logger
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# define functions for test cases with % syntax
def test_percent_syntax_with_strings():
    for _ in range(10000):
        logger.debug("debug message with % syntax: %s", "test")


def test_percent_syntax_with_dict():
    for _ in range(10000):
        data = {
            "name": "Alice",
            "age": 25,
            "height": 170,
            "city": "New York",
            "country": "USA"
        }
        logger.debug(
            "Debug message: %(name)s - %(age)d -%(height)i\
            - %(city)s: %(country)s", data
        )


def test_percent_syntax_with_decimal():
    for _ in range(10000):
        logger.debug("debug message with % syntax: %d", 0.5)


# define functions for test cases with f-strings
def test_f_string_with_strings():
    for _ in range(10000):
        logger.debug(f"debug message with f-string: {'test'}")


def test_f_string_with_decimals():
    for _ in range(10000):
        logger.debug(f"debug message with f-string: {0.5}")


def test_f_string_with_dict():
    for _ in range(10000):
        data = {
            "name": "Alice",
            "age": 25,
            "height": 170,
            "city": "New York",
            "country": "USA"
        }
        logger.debug(
            f"Debug message: {data['name']} -{data['age']} -{data['height']}\
                    -{data['city']}: {data['country']}"
        )


percent_syntax_time_strings = timeit.timeit(
        test_percent_syntax_with_strings, number=1)
percent_syntax_time_decimals = timeit.timeit(
        test_percent_syntax_with_decimal, number=1)
percent_syntax_with_dict = timeit.timeit(
        test_percent_syntax_with_dict, number=1)
f_string_time_strings = timeit.timeit(test_f_string_with_strings, number=1)
f_string_time_decimals = timeit.timeit(test_f_string_with_decimals, number=1)
f_string_time_dict = timeit.timeit(test_f_string_with_dict, number=1)


print(f"percent_syntax_time string: {percent_syntax_time_strings}")
print(f"percent_syntax_time decimal: {percent_syntax_time_decimals}")
print(f"percent_syntax_time dict: {percent_syntax_with_dict}")
print(f"f_string_time string: {f_string_time_strings}")
print(f"f_string_time decimal: {f_string_time_decimals}")
print(f"f_string_time dict: {f_string_time_dict}")
```

I've set up scenarios to time the execution of %-strings and f-strings.  Just for fun, I've also added some checks for interpolating a decimal value into the string.  I'll run each scenario ten thousand times on my mid-range laptop:

```
percent_syntax_time string: 0.0017548749999999995
percent_syntax_time decimal: 0.0017607079999999997
percent_syntax_time dict: 0.002460500000000001
f_string_time string: 0.0020255829999999975
f_string_time decimal: 0.0034761670000000022
f_string_time dict: 0.0043287920000000014

```

So there's definitely an improvement there.  it took 1.75ms to make 10,000 logs with percent syntax strings, while it took 2.00ms to do it with f-strings.  Since we're working with nice round numbers in multiples of ten, this makes it easy to convert: 175ns vs 200ns per individual operation, respectively. 

200ns-175ns = 25ns of time saved per log statement if we convert our logs to %-string formatting.  A nanosecond is a billionth of a second.  An astonishingly small amount of time, but we know that an increase in traffic can magnify small issues.


### At what point does it become important?

Let's do some back-of-the-napkin math and make some estimations.  Let's assume that our site has a round-trip request time of 1 second.  We don't know what our site does, but let's assume that we have to do things like get the current user, authorize them to access an endpoint, do some form processing, maybe write to a database.  The speed and efficiency of our logging request is probably not going to be the highest priority.  That said, maybe we're setting an ambitious goal of cutting that response time down from 1 second to just half that.  We're looking to gain speed in any way that we can, so examining our logging habits isn't necessarily a bad idea.  


**What if we processed 100,000 log statements per second?**

25ns * 100,000 = 2,500,000ns or 2.5ms of savings in total

**How about a million?**

25ns * 1,000,000 = 250,000,000ns or 250ms of savings

**A BILLION?!**

25ns * 1,000,000,000 = 25,000,000,000ns or 25s of savings

Ok, these numbers are obviously imprecise and at best reflect the theoretical maximum savings you can get by avoiding interpolating variables into strings until the logger emits the message.  As we discovered earlier, if a message being logged involves a function call, that function call will execute, even if the log level is below the current threshold.

It's hard to know where the juice is worth the squeeze.  Somewhere, someplace, I read that it takes a 25% improvement before a user notices an increase in response time for a website.  That said, we're imagining that we are a part of a broader, company-wide effort to reduce our response time, and even if we can squeeze out a 5% improvement, our efforts still might be worth it.

It's worth noting that if you can save 25ns by converting an f-string to a %-formatted one, you can save a further 150ns by *not logging anything at all*.  Removing a single log statement will save you the equivalent of converting *seven* log statements from the f-string- to %-formatted style.

### Despite inefficiencies, there are benefits to formatting logs with f-strings

In an environment where raw throughput trumps all, it makes sense to squeeze every drop of performance out of your logging.  After all, it's a relatively easy lift to do - you can even add rules into your CI/CD pipeline to prohibit f-strings from being introduced into the codebase.  There are some tradeoffs, of course. 

#### Avoid human error

I'll reckon it's easier to make this mistake:
```
def foo():
    one = 1
    two = 2
    three = 3
    four = 4
    five = 5
    # did not supply the fifth argument, but still have 5 %s's
    logger.info("%s - %s - %s - %s - %s ", one, two, three, four)
```

than it is to make this one:
```
def foo():
    one = 1
    two = 2
    three = 3
    four = 4
    five = 5
    # omit the fifth argument
    logger.info(f"{one} - {two} - {three} - {four}")
```

In the second example, you're missing the fifth argument but there will be no error, aside from possibly a warning from your IDE that you have an unused variable.  In the first example, you won't see the issue until it attempts to build the string at runtime.

#### Developer happiness

This one's pretty subjective, I will admit.  It's also highly situational. F-strings make for a nice developer experience, and it makes the code easier to read.  It also feels good to be working on projects that use up-to-date software versions and best practices.  In the past, I worked on a Python 2.7 codebase that was actively in development for YEARS after Python 3 was released, and I felt a little hobbled, being stuck with the "old way" of doing things.  Come to think of it, this particular scenario might have dredged up some old trauma - one of the main differences between Python 2 and 3 was the way strings were handled.

## Lessons Learned

Honestly I did not even know how Python's logging functionality worked - that it allowed you to bypass string interpolation until the absolute last second.  It's cool to know that functionality like that exists, and I would like the opportunity to someday work in an environment where every last drop of performance was squeezed out of the codebase.  At least I *think* I'd like that.  Hard to know until I'm in that position.

One thing that I always try to do is to put myself in the shoes of people who've come before me.  My host team inherited a codebase that was forked from an external team's work, and there are issues aplenty.  Finding myself in this situation, I can easily imagine myself reaching for any low hanging fruit, any small wins that would help to distract myself from the urge to just throw the whole thing out and write it over brand new.  But at the same time, I feel like it's important to pick your battles.  A lot of ink has been spilled about premature optimization and "ya ain't gonna need it" (YAGNI).  It's not a hill that I'm willing to die on, but it's my opinion that blocking a PR that used an f-string in a logging statement is probably going a bit too far.

In this particular circumstance, I would hazard a guess that much more significant performance increases could be gained by implementing a "refactor in place" strategy - an agreement by the team that any time an alteration to an area of the code that the team's not worked in should trigger an analysis of potential cleanup or improvement.  Developers should feel empowered to make changes in the code they touch, even if it's not directly related to the task at hand.
