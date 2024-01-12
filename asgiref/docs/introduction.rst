Introduction
============

ASGI is a spiritual successor to
`WSGI <https://www.python.org/dev/peps/pep-3333/>`_, the long-standing Python
standard for compatibility between web servers, frameworks, and applications.

WSGI succeeded in allowing much more freedom and innovation in the Python
web space, and ASGI's goal is to continue this onward into the land of
asynchronous Python.


What's wrong with WSGI?
-----------------------

You may ask "why not just upgrade WSGI"? This has been asked many times over
the years, and the problem usually ends up being that WSGI's single-callable
interface just isn't suitable for more involved Web protocols like WebSocket.

WSGI applications are a single, synchronous callable that takes a request and
returns a response; this doesn't allow for long-lived connections, like you
get with long-poll HTTP or WebSocket connections.

Even if we made this callable asynchronous, it still only has a single path
to provide a request, so protocols that have multiple incoming events (like
receiving WebSocket frames) can't trigger this.


How does ASGI work?
-------------------

ASGI is structured as a single, asynchronous callable. It takes a ``scope``, 
which is a ``dict`` containing details about the specific connection,
``send``, an asynchronous callable, that lets the application send event messages
to the client, and ``receive``, an asynchronous callable which lets the application
receive event messages from the client.

This not only allows multiple incoming events and outgoing events for each
application, but also allows for a background coroutine so the application can
do other things (such as listening for events on an external trigger, like a
Redis queue).

In its simplest form, an application can be written as an asynchronous function,
like this::

    async def application(scope, receive, send):
        event = await receive()
        ...
        await send({"type": "websocket.send", ...})

Every *event* that you send or receive is a Python ``dict``, with a predefined
format. It's these event formats that form the basis of the standard, and allow
applications to be swappable between servers.

These *events* each have a defined ``type`` key, which can be used to infer
the event's structure. Here's an example event that you might receive from
``receive`` with the body from a HTTP request::

    {
        "type": "http.request",
        "body": b"Hello World",
        "more_body": False,
    }

And here's an example of an event you might pass to ``send`` to send an
outgoing WebSocket message::

    {
        "type": "websocket.send",
        "text": "Hello world!",
    }


WSGI compatibility
------------------

ASGI is also designed to be a superset of WSGI, and there's a defined way
of translating between the two, allowing WSGI applications to be run inside
ASGI servers through a translation wrapper (provided in the ``asgiref``
library). A threadpool can be used to run the synchronous WSGI applications
away from the async event loop.
