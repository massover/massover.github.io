Title: Why I am excited about ASGI & Django
Date: 2020-06-23 8:14
Category: Django

Most talk about ASGI usually involves performance. While a performance boost can be nice, WSGI has been good enough for me.
As WSGI is blocking by nature, we could not use it to implement a streaming protocol.
Before Django had native ASGI support, you may have reached for [channels](https://channels.readthedocs.io/en/latest/) or even used
a third party service to support a streaming connection with the browser. With ASGI support coming in Django, 
we can implement [Server Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) 
as a nice sweet spot between WebSocket complexity and long polling.

## Lets start with WSGI

This is a single module [django](https://www.djangoproject.com/) application using [3.1 beta](https://www.djangoproject.com/weblog/2020/jun/15/django-31-beta-1-released/). 
Full code can be [found here](https://github.com/massover/sse-example).

```python
import json
import os
import sys
import time

from django.conf import settings
from django.core.asgi import get_asgi_application
from django.core.wsgi import get_wsgi_application
from django.http import HttpResponse, StreamingHttpResponse
from django.urls import path

settings.configure(
    DEBUG=(os.environ.get("DEBUG", "") == "1"),
    ALLOWED_HOSTS=["*"],
    ROOT_URLCONF=__name__,
    SECRET_KEY="super-secret-key",
    WSGI_APPLICATION=f"{__name__}.application",
)

application = get_wsgi_application()


def stream():
    counter = 100
    while counter > -1:
        i = json.dumps({"i": counter})
        content = f"event:i\ndata: {i}\n\n"
        counter -= 1
        yield content
        time.sleep(1)


def sse(request):
    response = StreamingHttpResponse(streaming_content=stream(), content_type='text/event-stream')
    response['Cache-Control'] = 'no-cache'
    return response


def index(request):
    return HttpResponse("""
    <h1>Countdown <span id="countdown"></span></h1>

    <script type="text/javascript">
        const evtSource = new EventSource("/sse/");
        const element = document.getElementById("countdown")
        evtSource.addEventListener("i", function(event) {
            i = JSON.parse(event.data).i
            if (i > 0) {
                element.innerText = i;
            } else {
                element.innerText = 'Blastoff!';
            }    
        });
    </script>
    """)


urlpatterns = [
    path("", index),
    path('sse/', sse)
]

if __name__ == "__main__":
    from django.core.management import execute_from_command_line
    execute_from_command_line(sys.argv)
```

In production, you would serve this application using a wsgi server such as [gunicorn](https://gunicorn.org/)

```python
# we'll use 1 worker for a "fair" comparison to asgi
guncorn wsgi:application --workers 1
```

If you successfully loaded that app up as is for wsgi, you'd probably notice:

- The countdown stops at 70s, this is due to gunicorn's default timeout at 30s
- The first page load is ok, subsequent page loads don't work as the worker is blocked.

```python
[2020-06-22 20:37:19 -0400] [16151] [CRITICAL] WORKER TIMEOUT (pid:16170)
[2020-06-22 20:37:19 -0400] [16170] [INFO] Worker exiting (pid: 16170)
[2020-06-22 20:37:19 -0400] [16355] [INFO] Booting worker with pid: 16355
[2020-06-22 20:37:50 -0400] [16151] [CRITICAL] WORKER TIMEOUT (pid:16355)
[2020-06-22 20:37:50 -0400] [16355] [INFO] Worker exiting (pid: 16355)
[2020-06-22 20:37:50 -0400] [16541] [INFO] Booting worker with pid: 16541
[2020-06-22 20:38:22 -0400] [16151] [CRITICAL] WORKER TIMEOUT (pid:16541)
[2020-06-22 20:38:22 -0400] [16541] [INFO] Worker exiting (pid: 16541)
[2020-06-22 20:38:22 -0400] [16729] [INFO] Booting worker with pid: 16729
[2020-06-22 20:38:53 -0400] [16151] [CRITICAL] WORKER TIMEOUT (pid:16729)
[2020-06-22 20:38:53 -0400] [16729] [INFO] Worker exiting (pid: 16729)
[2020-06-22 20:38:53 -0400] [16914] [INFO] Booting worker with pid: 16914
```

## Update to ASGI

To make this async, there are a few easy things we need to:

- Make our stream an async generator
- Make the index an async view

Then there are a few things to do that will remind you that ASGI support is new. If we try and use a `StreamingHttpResponse`
with our async generator, we'll run into `TypeError: 'async_generator' object is not iterable`.  To support an async generator 
for a streaming http response, we can extend `HttpResponse` with an [async http streaming response class](https://github.com/massover/sse-example/blob/master/responses.py).
Additionally, to support this new response, we need a [custom asgi handler](https://github.com/massover/sse-example/blob/master/asgi.py). 
The existing `ASGIHandler` does not support streaming from async_generators. We need to use an `async for part in response` 
instead of just our `for part in response`. Without methods to super, most of the logic for `__call__` and `send_response` 
is copied and pasted from the Django `ASGIHandler`, with additional code to close the request when browsers disconnect.

```python
import asyncio
import json
import os
import sys

from django.conf import settings
from django.http import HttpResponse
from django.urls import path

from handlers import get_asgi_application
from responses import AysncStreamingHttpResponse

settings.configure(
    DEBUG=(os.environ.get("DEBUG", "") == "1"),
    ALLOWED_HOSTS=["*"],
    ROOT_URLCONF=__name__,
    SECRET_KEY="super-secret-key",
    ASGI_APPLICATION=f"{__name__}.application",
)

application = get_asgi_application()


async def stream():
    counter = 100
    while counter > -1:
        i = json.dumps({"i": counter})
        content = f"event:i\ndata: {i}\n\n"
        counter -= 1
        yield content
        await asyncio.sleep(1)


async def sse(request):
    response = AysncStreamingHttpResponse(streaming_content=stream(), content_type='text/event-stream')
    response['Cache-Control'] = 'no-cache'
    return response


async def view(request):
    return HttpResponse("""
    <h1>Countdown <span id="countdown"></span></h1>

    <script type="text/javascript">
        const evtSource = new EventSource("/sse/");
        const element = document.getElementById("countdown")
        evtSource.addEventListener("i", function(event) {
            i = JSON.parse(event.data).i
            if (i > 0) {
                element.innerText = i;
            } else {
                element.innerText = 'Blastoff!';
            }    
        });
    </script>
    """)


urlpatterns = [
    path("", view),
    path('sse/', sse)
]

if __name__ == "__main__":
    from django.core.management import execute_from_command_line
    execute_from_command_line(sys.argv)
```

In production, you would serve this application using an asgi server such as [uvicorn](https://www.uvicorn.org/)

```python
uvicorn asgi:application --workers 1
```

Once loaded, open a tab, and then open some more. Even with one worker, it's able to handle it because of the non blocking nature of
an ASGI request. Besides the custom `ASGIHandler` and `AysncStreamingHttpResponse`, the code is very familiar to WSGI
and doesn't change much besides using async constructs. What else can you think of that we can implement with native ASGI in Django?