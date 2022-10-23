## Async programming in the gevent style. An approach without async-await.

There is a pretty known writing by Bob Nystrom named
["What color is you function"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)?

In it, the author discusses the existing approaches to async programming.
The majority of programming languages, including Python, he says, use functions "of different colors".
That is, one color for "regular" functions and a special one - for async ones. If you have to use async and await keywords for async functions, you are using a special color.
Golang does not have function colors, as a vivid counter-example.

Recently, taking a closer look at how sqlalchemy provides async I/O,
I have found a way to do the async programming not using function coloring.

It is best suited for cases when you do not actually need concurrency, just the async I/O. When you have many logical threads, executing at the same time, but they are not dependent on each other. Within a particular thread operations execute sequentially, so concurrency is not needed. A web server is a good example, handling of the web requests being those independent "threads". 

**What does it look like**

It looks like a regular synchronous code, that's why I say it's programming in the gevent style. However, we don't do any monkeypatching. Instead, our code should add support for the async I/O.  

Here is an example, it is a working [code](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/blob/main/kitchen/views.py#L20),
you can run it yourself.

```python
from kitchen.models import Order

@as_async
def food_delivery(request):
    order: Order = prepare_order(request)
    order.save()
    resp = myhttpx.post(settings.KITCHEN_SERVICE, data=order.as_dict())
    match resp.status_code, resp.json():
        case 201, {"mins": _mins} as when:
            if consumer := ws.consumers.get(request.user.username):
                consumer.send_json(when)
            return JsonResponse(when)
        case _:
            kitchen_error(resp)
```

In the above django view, we save an order into the database, then make a request to a kitchen service, then notify the customer on when to expect the delivery.

The vanilla django is used, however, with an async [backend](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/tree/main/pgbackend).
myhttpx is a wrapper over httpx. Ws consumer needed [wrapping](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/blob/main/kitchen/ws.py#L8) too, of course.

As you can see, we made django an async-capable framework. The higher-level libraries like django-rest-framework, for example, require even less effort (zero).

**How it is done: the greenlet hack**

The greenlet hack is the same that is used by sqlalchemy, and can be found [here](https://github.com/Bi-Coloured-Python-Rock-Snake/greenhack).

The idea of it is splitting the code between two greenlets, a sync and an async one. All async operation are "filtered out" to the async greenlet, the remaining code being composed solely of "regular" functions.

There are two ways of using [`greenhack`](https://github.com/Bi-Coloured-Python-Rock-Snake/greenhack).

In the first case there already is an event loop your application is run within. In this case you would usually decorate your top-level function with @as_async.

The second case is when you don't have an event loop and have to start one yourself. An example for this is Python REPL. Or some legacy console utility. In this case you should call

```python
greenhack.start_loop()
```

After this your code is able to use async I/O. Here you can see that I've added this line to [manage.py](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/blob/main/manage.py#L25) (django console utility) to make it work with an async backend.

**Merits**

Here I will try and share my own opinion on the possible value of the described approach.

It is just an approach, it doesn't have any code. It relies on asyncio by default.

It is indispensable for working with legacy codebases. I am sure this is the future of the django project, for example.

The approach has a benefit in comparinson with async-native code: it can support both async and blocking code with one codebase.

Hardly it can be viewed as a general-purpose approach to async programming, since it doesn't offer any means for concurrent tasks. But I'm sure it can have a wide range of uses.

**Is it production-ready?**

It has been tested in production primarily by sqlalchemy, and zzzeek [says](https://github.com/Bi-Coloured-Python-Rock-Snake/readme/issues/3#issuecomment-1273439668) it's production-ready.                                                                 

**Issues/cons**

Using the greenlet hack, one can face certain issues related to the debugging and profiling.
Since the code gets split between the sync and the async greenlet, the stack of frames is also split. However, you can always print the correct stack of frames yourself. [zzzeek](https://github.com/zzzeek) says the profiling isn't easy too.

On the bright side, the async code does work inside the REPL! With asyncio, it is not so: since nested event loops are forbidden, you have to use [nest_asyncio](https://github.com/erdewit/nest_asyncio) for debugging.

In relation with debugging/profiling issues, there is nothing that cannot be implemented. The overall approach is simple, because we don't bother with concurrent tasks, and are happy with the sequential execution.


**Plans**

The project is set to be the home for async django.

*I encourage the django project to adopt the described approach as a default.*

There already is proof-of-concept [code](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/tree/main/pgbackend) for async database backend using the psycopg driver. The community can start adding support for other providers too.

The next step should probably be extracting the tests for database backends out of the overall testsuite of the django project.