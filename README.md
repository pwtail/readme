# RFC: balrog project

*Ancient demons fighting by your side.*

**No async/await**

The balrog project takes an unconventional approach: **async/await keywords are discouraged!**
You write the async code in the same way as you write the sync code.
Under the hood, it will still run asynchronously, if you choose so.
Balrog aims to support both sync and async I/O equally well.

**For web applications mostly**

The approach is mainly oriented at the web applications. **django** will be definitely supported.
The thing is that balrog requires an ecosystem of libraries written or adapted specifically for it.
So, web applications fit well, since they have a predictable and a quite limited set of needs.

**An example**

```python
import requests
from delivery.models import Order

def food_delivery(request):
    order: Order = prepare_order(request)
    order.save()
    resp = requests.post('http://kitchen-place.org/orders/', data=order.as_dict())
    assert resp.status_code == 201
    match resp.status_code, resp.json():
        case 201, {"mins": mins}:
            pass
        case _:
            kitchen_error(resp)
    ws = get_ws_connection(request)
    ws.send(f'Order #{order.id} will be delivered in {mins} minutes.')
```

What we see above is a regular synchronous django view. Except the last line: `ws.send` cannot really be a synchronous call
if it is a server-sent message. However, we can imagine that we have a separate service for sending messages, and that
`ws.send` uses either a ws connection to it, or any other way of communication. So, a perfectly valid django view.

Now, few people know that we can use the same, or very similar, code to run it asynchronously - without a single
await statement (and yes, `ws.send` could be actually a server-sent message then). Django can be
using an async database backend, the http client - an async implementation, and the ws server is always
asynchronous. But the public API can be the same for sync and async code. And with balrog, it is the same!

**How is it possible?**

Balrog uses the greenlet hack, best known from it's use in sqlalchemy. It removes the need for your functions to have the
`async` and `await` keywords despite having an async implementation.

Here I've put a small package [greenbrew](https://github.com/balrogproject/greenbrew) for demonstration.

**The plan of action**

The first milestone is enabling the use of django orm. An async database backend is required for this.
The first database will be PostgreSql, psycopg3 driver, because it is the simplest to support: it's async API mirrors the sync one.

The next set of activities could be:

- async backends for other database providers and implementations
- http client
- ws functionality
- etc
