---
layout: post
title:  "Mock decorator during tests to override its parameters"
date:   2021-01-21 09:31:00 +0200
categories:
tags: [python, mock, decorator]
---

# Mock decorator during tests to override its parameters


What happens if you want to mock a decorated method during tests? And in what kind of scenarios you need this?

As one can find, there are various articles in Stackoverflow and all around about cases where people needs to mock a decorated method, 
mainly in order to change the functionality of the decorator during tests.

In those articles, you'll find out that, decorators are applied at the time that a module is imported. 
This is why if you imported a module which uses a decorator you want to patch at the top of your file. 
Any attempt to patch it later, without reloading it, the patch would have no effect.

I've found to very helping questions in stackoverflow about this issue:

* https://stackoverflow.com/questions/19812570/how-to-mock-a-decorated-function
* https://stackoverflow.com/questions/7667567/can-i-patch-a-python-decorator-before-it-wraps-a-function

But that wasn't my case... 

## What if you want to actually change how the decorator is called, but not the decorator itself?

A specific case I was dealing with was how to change a decorator's call and pass as arguments specific values that I needed for testing.

I was working with Piston3 for Python and I had a custom handler where I needed to run some tests to validate that the throttle policy was applied.
Piston is an old project but it is very useful for my example. If you want to check some code you'll find it at 
[the github repo](https://github.com/userzimmermann/django-piston3) since it seems that all bitbucket links are not working...


My custom handler class was something like the following:

```
from piston3.handler import AnonymousBaseHandler
from piston3.utils import throttle


class FooHandler(AnonymousBaseHandler):
    """
    Gets all foo.
    """
    allowed_methods = ('GET',)


    @throttle(settings.API_MAX_REQUESTS, settings.API_THROTTLE_INTERVAL)
    def read(self, request):
        # do your logic here to get return foos 

```

As you can see, the `throttle` decorator is getting two parameters from the settings which are loaded when it is called. 
If I want to change the settings' values during tests, I can override of course with the `override_settings` decorator on top of my method.


```
from module.handler import FooHandler


class FooHandlerTestCase(TestCase):

    @override_settings(API_MAX_REQUESTS=1)
    @override_settings(API_THROTTLE_INTERVAL=10)
    def test_foo_read(self):
        handler = FooHandler()
        request = RequestFactory()
        request.user = UserFactory()

        response = handler.read(request)
        response = handler.read(request)
        self.assertEqual(response.status_code, 429) # Because we set only 1 call per 10 seconds

```


But this doesn't work because, as it is already mentioned above, when we load FooHandler the existing decorator
is already loaded (when we first import it) and has already got the values from settings.

So the `override_settings` has no actual effect at all. I've read quite some articles about how to mock the `throttle` decorator but non worked. Not at least the way I wanted/expected. 
Having spent most of my day, eventually, my solution was somehow more naive, but it did the trick...

I mocked the decorator, by inheriting and overriding the read method that I wanted to test... I don't know if this is a good approch, but in the end I had something that gave actual results.


```
from module.handler import FooHandler


class MockFooHandler(FooHandler):

    @throttle(1, 10)
    def read(request):
        return super(MockFooHandler, self).read(request)


class FooHandlerTestCase(TestCase):

    def test_foo_read(self):
        handler = MockFooiHandler()
        request = RequestFactory()
        request.user = UserFactory()

        response = handler.read(request)
        response = handler.read(request)
        self.assertEqual(response.status_code, 429) # Because we set only 1 call per 10 seconds

```

This did the trick! Hack? I don't know... Perhaps :)











