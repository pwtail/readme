# RFC: balrog project

*Ancient demons fighting by your side.*

**No async/await**

The balrog project takes an unconventional approach: **async/await keywords are discouraged!**
You write the async code in the same way you would write the sync code.
Under the hood, it will still run asynchronously, if you choose so.
Balrog aims to support both sync and async I/O equally well.

**How is it possible?**

Balrog uses the **greenlet** hack, which is best known from its use in sqlalchemy. It removes the need for functions to have the
async/await keywords despite them having an async implementation. Balrog uses that trick all the way down.

Here I've put up a small package [greenbrew](https://github.com/balrogproject/greenbrew) for those who haven't seen
the trick.

**Intended for web applications**

The issue is, the approach requires an ecosystem of libraries written or adapted specifically for it.
So, the web applications fit well, since they have a predictable and limited scope.
**django** will definitely be supported.

**An example**

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

Here you can see a regular synchronous django view. Except that `ws.send` cannot really be a synchronous call,
if it is a server-sent message. However, we can imagine we have a separate service for sending messages, and
`ws.send` delegates to it. So, a perfectly valid django view.

Now, we'll try to apply the above-stated approach for it. No real code at this stage yet, but we can imagine
we have all of the following provided:

- async database backend for django
- async implementation of the http client
- websocket server - this thing is always asynchronous

Now, what I'm trying to say is, all of the above can be hidden under the hood, as an implementation detail, the public API
likely being shared with their sync counterparts. The code snippet above likely won't change at all.

**The name**

The most wicked demon of ancient times is greenlet of course, but django is powerful too.

**The REPL**

Some people know that some advanced Python REPL's like IPython, allow expressions like `await smth`. However, nested
event loops are not allowed, so you can not call async functions when debugging an async service.
To work around that problem people usually use `nest_asyncio` (which patches asyncio).

To my surprise, the greenlet approach works in the REPL out of the box. Even in the regular Python REPL, pdb!
`nest_asyncio` is not required.

Some integration with IPython will still be needed, for the case when it's not debugging an existing service
(which has already been wrapped in a greenlet), but when it's just code you type in the REPL (which has to be wrapped).