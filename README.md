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
usecases.

to propose sticking with the go-like approach, not to use async/await.

**The drawbacks**

1

**The plans**

The blocker now is the absence of any async database backends in django. But that is not much work to add one.
The first database driver supported will be [psycopg](https://github.com/psycopg/psycopg).
The sync backend for that driver is [almost](https://github.com/django/django/pull/15687) merged.