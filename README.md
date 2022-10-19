## Doing async I/O in Python without async-await

> Then the Bi-Coloured-Python-Rock-Snake came down from the bank, and knotted himself in a double-clove-hitch round the Elephant’s Child’s hind legs, and said, ‘Rash and inexperienced traveller, we will now seriously devote ourselves to a little high tension, because if we do not, it is my impression that yonder self-propelling man-of-war with the armour-plated upper deck’ (and by this, O Best Beloved, he meant the Crocodile), ‘will permanently vitiate your future career.
> 
> That is the way all Bi-Coloured-Python-Rock-Snakes always talk.

"The Elephant's Child", by Rudyard Kipling

**Bi-coloured Python**

There is a pretty known writing by Bob Nystrom named
["What color is you function"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).
In it, the author reflects on the different approaches to the async programming that programming languages use.
The majority of languages, including Python, use function coloring to do the async programming.
That means, you have to mark your async functions with a keyword and use another keyword for calling them (await).
Golang does not use function coloring, as a vivid counter-example.

The above-mentioned article was just for a preface.
I won't discuss the ways of implementing concurrency, because I'm not too much of an expert.
However, there is one special usecase, that I want to highlight.

**When concurrency is not actually needed**

Imagine an application that has clear isolated logical threads of execution.
Threads do not depend on each other: although being run concurrently, they don't exchange any data.
So the code defining thread's logic can be written as if other threads not existed.
 We don't need concurrency within a thread: all operations happen one after another.

 In terms of goroutines, we can say that we don't actually need the `go` statement (the means to start a goroutine)
 - if we can presume the top-level goroutines are somehow started for us.

 A real-life example can be handling web requests. Often  in web programming we just want to use the async I/O instead of the blocking
 one, because it is more performant for the needs of web services. We don't usually need any concurrency in our handling
 of a web request.

 **An alternative approach: no async/await**

The same way that you don't need the `go` statement, you also can do without async/await in the afore-mentioned case.
And I want to show it is also a practical choice to do so, that almost doesn't have any drawbacks.
And since this case is very common for web development, I think the web
development is better off without async/await (mostly).

Technically it is done using the [greenlet hack](https://github.com/Bi-Coloured-Python-Rock-Snake/greenhack).
It actually allows the async functions to be called within regular ones.
The greenlet hack has been used in sqlalchemy for it's async functionality for a couple of years by now,
so is kind of battle-tested.

The problems with async-await arise when you have to deal with "legacy" codebases like django or sqlalchemy.
It wasn't a coincidence sqlalchemy was the one to adopt the greenlet trick.
Also there is no simple way for a library to support both sync and async I/O.

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

An example is worth a thousand words. Here you see a regular sync django view.
Except that `ws.send` cannot really be a synchronous call,
as it is a server-sent message. But, we can imagine we have a separate sender service, so `ws.send` delegates to it.


**The plans**

TODO