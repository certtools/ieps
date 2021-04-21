# IEP02 - IntelMQ Mixins

I would like to lower the bar for adding additional bots by introducing mixins.
Mixins are a concept coming from OOP theory and it is closely related to
multiple inheritance.
I think it could be a way to avoid a lot of duplicate code by outsourcing
various functionalities to specific classes.

## Current situation

At the moment there are two types of bots that could make use of mixins.
The first are bots that make HTTP requests.
Currently we make that easier for bots by providing two methods,
`create_request_session` in intelmq/lib/utils.py and `set_request_parameters`
as part of the bot module (intelmq/lib/bot.py) itself. When the bot wants to
use a request session, it first uses `set_request_parameters` and then uses the
session object returned by `create_request_session` to make the request. I
think its a bit confusing to have methods that are related in two different
modules and I think the Bot class itself should not contain any code that is
not used by all the bots.

The second type of bots that could make use of mixins are bots that use a
cache. There are a lot of those that all use the `intelmq.lib.cache` module
which provides a Cache object to work with.

## Improvement using mixins

Using a Python class we could move everything related to HTTP requests to a
class called HttpMixin. A bot that wants to use functionality from this Mixin
just would have to inherit from this class, i.e.:
```python
class MySuperBot(CollectorBot, HttpMixin):
```
The MySuperBot class would then have all the relevant attributes to use session
objects and it would also have a private `__session` attribute to work with.
Another upside of this approach is, that it is possible to generate a list of
all the attributes a Bot has write access to (i.e. http_proxy, http_username,
ssl_client_cert...)

Similarly we could introduce a `CacheMixin` that allow access to private
`__cache` attribute. If MySuperBot does not only want to make HTTP requests,
but also have some caching mechanisms, it could simply inherit from both
classes:
```python
class MySuperBot(CollectorBot, HttpMixin, CacheMixin):
```

I propose to create a new module `intelmq.lib.mixins` that should be the home
for the mixins.

I have created a POC that implements the HttpMixin and uses it in the
HTTPCollectorBot:
https://github.com/certtools/intelmq/tree/schacht/http-mixin


Mixins are also used in Django to add functionality to their class based Views,
for more details see https://docs.djangoproject.com/en/3.1/topics/class-based-views/mixins/
A longer article about inheritance, MRO, mixins and Python that also talks
about the implementation in Django can be found on
https://www.thedigitalcatonline.com/blog/2020/03/27/mixin-classes-in-python/