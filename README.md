## Doing async I/O in Python without async-await

The quotations are from "The Elephant's Child" by Rudyard Kipling.

**Bi-coloured Python**

>The first thing that he found was a Bi-Coloured-Python-Rock-Snake curled round a rock.

There is a pretty known writing by Bob Nystrom named
["What color is you function"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).

In it, the author reflects on the different approaches to async programming that programming languages have.
The majority of languages, including Python, use "functions of different colors".
That means, one color for regular functions and another for async ones. In Python that is done with async and await keywords.
Golang does not use function coloring, as a vivid counter-example.

That is said just for a preface actually.
I won't discuss the ways of implementing concurrency, because I'm not too much of an expert.
However, there is one special usecase, that I want to discuss.

**When concurrency is not actually needed**

Imagine an application that can be split into clear isolated logical threads of execution.
Handling web requests is a good example.

Threads do not depend on each other: although being run concurrently, they don't exchange any data.
The code defining thread's logic can be written as if other threads not existed.
 We don't need concurrency within a thread: all operations can happen sequentially,
 one after another.

 In terms of goroutines, we can say that we don't actually need the `go` statement (the means to start a goroutine) - if
 we can presume the top-level goroutines are somehow started for us.

Speaking of the web programming, it is often just the case: we use the async I/O instead of the blocking
 one, because our services are more performant that way. We don't usually use concurrent tasks to handle a web request.

 **An alternative approach: no async/await**

In the same way as you don't need the `go` statement, you also can do without async/await in the afore-mentioned case.
And I will show it is also a practical choice to do so, that almost doesn't have any drawbacks.
Since this case is common, at least for the web development, it can be widely aplicable.

Technically it is done using the [greenlet hack](https://github.com/Bi-Coloured-Python-Rock-Snake/greenhack).
It actually allows the async functions to be called from within the regular ones.
The greenlet hack has been used in sqlalchemy to enable the use of async database drivers.

The no-async-await approach lets you avoid many troubles of async programming in Python:

- It enables the use of "legacy" codebases like django. It wasn't a coincidence sqlalchemy was the first to adopt the greenlet trick. Speaking of django, it is a just a matter of switching the database backend to an async one (and then specifying the new backend in settings.py).

- It gives you the possibility to easily support both sync and async I/O. However,
  since you write code in synchronous style in both cases, there are not so many reasons to use blocking I/O for server-side programming.

**An example**

>Vantage number one!’ said the Bi-Coloured-Python-Rock-Snake. ‘You couldn’t have done that with a mere-smear nose'

```python
import requests
from delivery.models import Order

def food_delivery(request):
    order: Order = prepare_order(request)
    order.save()
    resp = requests.post('https://kitchen-place.org/orders/', data=order.as_dict())
    match resp.status_code, resp.json():
        case 201, {"mins": mins} as when:
            ws = get_ws_connection(request)
            ws.send(f'Order #{order.id} will be delivered in {mins} minutes.')
            return JsonResponse(when)
        case _:
            kitchen_error(resp)
```
Here you see a regular django view using blocking I/O.
Except that `ws.send` cannot really be a synchronous call,
as it is a server-sent message. But, we can imagine we have a separate sender service, so `ws.send` delegates to it.

I have [rewriten](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/blob/main/kitchen/views.py) it into an async one. If you follow the link you will see the code very similar to the code snippet above. Because you write the code in the same way regardless of whether you use the blocking or async I/O.

Things that were required for the rewrite:

- Async database backend for django (actually, that is a repository for the backend, the project being an example of its use)
- An http client ([myhttpx](https://github.com/Bi-Coloured-Python-Rock-Snake/pgbackend/blob/main/myhttpx.py), a thin wrapper over httpx)
- [channels](https://channels.readthedocs.io/en/stable/) for the websocket functionality (because it hasn't been included in django for unknown reasons)

In case you haven't figured it out, the approach lets you use higher-level libraries like
django-rest-framework or swagger tools without modifications.

>‘’Vantage number two!’ said the Bi-Coloured-Python-Rock-Snake. ‘You couldn’t have done that with a mear-smear nose

**Is it production-ready?**

The greenlet hack has been used in sqlalchemy for a couple of years by now, so it is kind of battle-tested. The async backends for django are new so require to be tested. However, if we take the `psycopg` driver, it's async API is a mirror of the sync one, which is [almost merged](https://github.com/django/django/pull/15687). So creating the async backend for is not much work.

The rest of the needs of the web development can be satisfied even more easily, as you have seen from the example above.

**Present issues**

You can have some issues related to debugging and profiling, more with the latter than the former.
The code gets split between the sync and the async greenlet, the stack of frames in those too being not connected to each other. You can always print the correct stack of frames yourself however. [zzzeek](https://github.com/zzzeek) says the profiling isn't easy too.

On the bright side, the async code does work in the REPL! With asyncio it is not so, since nested event loops are forbidden, so you have to use [nest_asyncio](https://github.com/erdewit/nest_asyncio) for debugging.

Again, in relation with debugging/profiling issues: there is nothing that cannot be implemented. The overall approach is simple, simpler than the one asyncio has. Because asyncio provides real concurrency, whereas for us sequential execution of tasks is enough (within a given logical thread).

> ‘Well,’ said the Bi-Coloured-Python-Rock-Snake, ‘you will find that new nose of yours very useful to spank people with.’
> ‘Thank you,’ said the Elephant’s Child, ‘I’ll remember that; and now I think I’ll go home to all my dear families and try.’