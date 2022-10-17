## Doing async I/O in Python without async-await

> Then the Bi-Coloured-Python-Rock-Snake came down from the bank, and knotted himself in a double-clove-hitch round the Elephant’s Child’s hind legs, and said, ‘Rash and inexperienced traveller, we will now seriously devote ourselves to a little high tension, because if we do not, it is my impression that yonder self-propelling man-of-war with the armour-plated upper deck’ (and by this, O Best Beloved, he meant the Crocodile), ‘will permanently vitiate your future career.
> 
> That is the way all Bi-Coloured-Python-Rock-Snakes always talk.

"The Elephant's Child", by Rudyard Kipling

**Bi-coloured Python**

In "What color is your function" <author> is dividing programming languages into the ones that do ... async functions into a separate
    category (or type if you like), and the ones that do not. Python, for example, fell in the first category, go - in the second.

if your function uses async I/O, you have to mark it with async.

problems arise not only what to deal with "legacy" codebases like django or sqlalchemy,
but also, what if a library wants to support both sync and async I/O?

I want to propose an alternative sol-n, that almost doesn't have drawbacks, except that it doesn't work for all
usecases. I will ... below.


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
greenlets was justified.

**The name**

The most wicked demon of ancient times is greenlet of course, but django is powerful too.

**The REPL**

Some people know that advanced Python REPLs like IPython allow expressions like `await smth`.

However, it is limited. Nested
event loops are not allowed, and unfortunately that is exactly what you need
when debugging an async service and want to call an async function.
To work around that, people usually use
[nest_asyncio](https://github.com/erdewit/nest_asyncio), which patches asyncio.

To my surprise, the greenlet approach works in the REPL **out of the box**. Even in the regular Python REPL, even in pdb!
`nest_asyncio` is not required.

Some integration with IPython will still be needed, when it's just code you type in the REPL, not the one being run
by the event loop.

**The plans**

The blocker now is the absence of any async database backends in django. But that is not much work to add one.
The first database driver supported will be [psycopg](https://github.com/psycopg/psycopg).
The sync backend for that driver is [almost](https://github.com/django/django/pull/15687) merged.