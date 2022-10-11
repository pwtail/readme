# balrog project

*Ancient demons fighting by your side.*

**No async/await**

The balrog project takes an unconventional approach: **async/await keywords are discouraged!**
You write the async code in the same way you would write the sync code.
Under the hood, it will still run asynchronously, if you choose so.
Balrog aims to support both sync and async I/O equally well.

**How is it possible?**

Balrog uses the **greenlet** hack, which is best known from its use in sqlalchemy. It removes the need for functions to have the
async/await keywords despite them having an async implementation.

Those who haven't seen the trick - take a look at the [shadow](https://github.com/balrogproject/shadow) package.

**Intended for web applications**

Primarily. Since they have predictable needs and a limited scope.
**django** will definitely be supported.

**An example**

Balrog allows you to write async code that looks synchronous.
Not only does it look that way, you also can use sync libraries like django/drf
without them knowing much about that. Here is an example of such use:

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

This is a regular sync django view. Except that `ws.send` cannot really be a synchronous call,
as it is a server-sent message. But, we can imagine we have a separate sender service, so `ws.send` delegates to it.

In ordert to make this view use async, we need:

- async database backend for django
- async implementation of the http client (check)
- websocket server - this thing is always asynchronous (check)

With barlog, when we provide the above requirements, the code will look the same, or very similar.
Also, it will be able to integrate with higher-level frameworks like drf,
without them even knowing the I/O is async.

**Some background**

The idea was born out of my [discussion](https://github.com/balrogproject/rfc/issues/3) with
@zzzeek about whether the use of
greenlets is justified.

**The name**

The most wicked demon of ancient times is greenlet of course, but django is powerful too.

**The REPL**

Some people know that advanced Python REPLs like IPython allow expressions like `await smth`.

However, it is limited. Nested
event loops are not allowed, and unfortunately that is exactly what you need
when debugging an async service and want to call an async function.
To work around that, people usually use
[nest_asyncio](https://github.com/erdewit/nest_asyncio), which patches asyncio.

To my surprise, the greenlet approach works in the REPL **out of the box**. Even in the regular Python REPL, pdb!
`nest_asyncio` is not required.

Some integration with IPython will still be needed, in the case when it's just code you type in the REPL
(which has to be wrapped in a greenlet). The case that works out of the box, that I was talking about, was debugging existing code.

**The plans**

The blocker now is the absence of any async database backends in django. But that is not much work to add one.
The first database driver supported will be [psycopg](https://github.com/psycopg/psycopg).
The sync backend for that driver is [almost](https://github.com/django/django/pull/15687) merged.