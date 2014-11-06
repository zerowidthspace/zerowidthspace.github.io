---
layout: post
title: Faux Function Type Signatures in Python
date: 2011-08-10 15:00:00
disqus: y
---

A recent post on [stackoverflow.com](http://stackoverflow.com/questions/7019283/automatically-type-cast-parameters-in-python) piqued my interest — how can we implement a sort of type signature for functions in Python? I.e., a function like

{% highlight python %}
@signature(int, int)
def foo(x, y):
    print x+y
{% endhighlight %}

A call to `foo(3,4)` should result in `7` and not `'34'`. This arguably goes against Python’s duck typing principles, but let us humor the idea for just a moment. The following Python features will be helpful:

* Reified types: types are first class citizens in Python…try `print int` to see for yourself.
* `inspect`: we need some code reflection to get the function parameter names, in order of application.

Here’s what I came up with (and you can view my original submission on the StackOverflow):

{% highlight python %}
import inspect
 
def signature(*sig):
    def __decorator__(fn):
        def __wrapped__(*args, **kwargs):
            arglist = inspect.getargspec(fn).args
            mod_args = dict([(vid, ctype(rval)) for vid, ctype, rval in zip(arglist, sig, args)])
 
            try:
                for k, v in kwargs.items():
                    mod_args[k] = sig[arglist.index(k)](v)
            except ValueError, e:
                raise TypeError("%s() got an unexpected keyword argument." % fn.__name__)
 
            fn(**mod_args)
 
        return __wrapped__
    return __decorator__
{% endhighlight %}

The breakdown is as follows. Inside the function decorator, we use inspection to get `arglist` — a list of the arguments to fn our function, in the order they should be applied. Then we zip `arglist, sig` (containing the types following the order of the arguments), and `args` (the actual values of the arguments passed to `fn`) together in our list comprehension; this allows us to apply a type cast to each of the `args`:

```
[(vid, ctype(rval)) for vid, ctype, rval in zip(arglist, sig, args)]
```

As you can see, I also tacked on the argument name (`vid` here is supposed to stand for “variable identifier”), which will come in handy soon.

The next step is to merge and convert the passed keyword arguments to the arguments we’ve already constructed. If the user passes a keyword argument that isn’t supposed to be passed, we will raise a more appropriate `TypeError` as opposed to having a more obtuse `ValueError` come up.

And that’s about it! Let’s try it out:

{% highlight python %}
@signature(int, int, str)
def foo(x, y, z):
    print x+y
    print type(z), z
 
foo(3, '4', z=["testing", "123"])
{% endhighlight %}

And indeed the output is:

{% highlight python %}
7
['testing', '123']
{% endhighlight %}