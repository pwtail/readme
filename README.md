# RFC: balrog project

*Ancient demons fighting by your side.*

**No async/await**

The balrog project takes an unconventional approach: **async/await keywords are discouraged!**
You write the async code in the same way you would write the sync code.
Under the hood, it will still run asynchronously, if you choose so.
Balrog aims to support both sync and async I/O equally well.

**How is it possible?**

Balrog uses the **greenlet** hack, which is best known from its use in sqlalchemy. It removes the need for functions to have the
async/await keywords despite them having an async implementation.

Here I've put up a small package [greenbrew](https://github.com/balrogproject/greenbrew) for those who haven't seen
this trick.

**Intended for web applications**

Balrog is mainly oriented at web applications. **django** will definitely be supported.
The issue is that this approach requires an ecosystem of libraries written or adapted specifically for it.
So, web applications fit well, since they have a predictable and limited scope.

**An example**

```python
import requests
from delivery.models import Order

def food_delivery(request):
    order: Order = prepare_order(request)
    order.save()
    resp = requests.post('http://kitchen-place.org/orders/', data=order.as_dict())
    match resp.status_code, resp.json():
        case 201, {"mins": mins}:
            pass
        case _:
            kitchen_error(resp)
    ws = get_ws_connection(request)
    ws.send(f'Order #{order.id} will be delivered in {mins} minutes.')
    return HttpResponse()
```

What we see above is a regular synchronous django view. Except the last line: `ws.send` cannot really be a synchronous call,
if it is a server-sent message. However, we can imagine that we have a separate service for sending messages, and that
`ws.send` delegates to it. So, a perfectly valid django view.

Now, few people know that we can use the same, or very similar, code to run it asynchronously - without a single
await statement (and yes, `ws.send` could actually be a server-sent message in that case). For that, django should be
using an async database backend, the http client - to have the async implementation, and the ws server is always
asynchronous. But the public **API can be the same** for sync and async versions. And with balrog, it is the same!

**The name**

The most wicked demon of ancient times is greenlet of course, but django is powerful too.