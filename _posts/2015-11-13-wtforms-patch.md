---
layout: post
title: Handling Unexpected WTForms Default Values
permalink: /p/wtforms-patch/
tags: [python, wtforms]
---

The [WTForms](https://github.com/wtforms/wtforms) library is great for
working with forms and validations. However, it does not work well with
JSON form data. Specifically, fields that were not provided in the request
are assigned default values. This causes problems when you expect `PATCH`-like
requests. Example:


{% capture c0 %}import wtforms

class MyBaseForm(wtforms.Form):
    pass

class FooForm(MyBaseForm):
    foo = wtforms.StringField('Foo')
    bar = wtforms.IntegerField('Bar')
    is_baz = wtforms.BooleanField('IzBaz')

fd = DummyMultiDict({'foo': 'ayy'})
f = FooForm(fd)

print(f.data)

# # # # #
class DummyMultiDict(dict):
    def getlist(self, key):
        v = self[key]
        return v if isinstance(v, (list, tuple)) else [v]
{% endcapture %}

<pre><code class="py">{{ c0 | escape }}</code></pre>


The above produces:

    {'is_baz': False, 'foo': 'ayy', 'bar': None}


This is not necessarily what you want when working with REST APIs or PATCH-like
requests. (Again, wtforms is not really meant for that). The default values your
code expects may be different from the default values wtforms assigns.

Suppose you have:

{% capture c1 %}def save(foo='No name', bar=42, is_baz=True):
    return db.save(Object(foo=foo, bar=bar, is_baz=is_baz))

fd = DummyMultiDict({'foo': 'ayy'})
f = Foo(fd)

save(**f.data)
{% endcapture %}

<pre><code class="py">{{ c1 | escape }}</code></pre>

In this case, you would expect a new entry in the "database" to be
`Object(foo='ayy', bar=42, is_baz=True)`. But the actual entry saved is
 `Object(foo='ayy', bar=None, is_baz=False)`.
 All your unit tests fail, your users are confused, and your manager is mad.


Here is a small wrapper around `wtforms.Form` to better handle the above problem. (There is also [wtforms-json](https://github.com/kvesteri/wtforms-json),
but its interface is quite different and feels wrong.)

## JForm

{% capture c2 %}import wtforms


class JForm(wtforms.Form):

    def process(self, *args, **kwargs):
        if args:
            formdata = args[0]
        else:
            formdata = kwargs.get('formdata', None)
        self._formdata = formdata
        super(JForm, self).process(*args, **kwargs)

    @property
    def data(self):
        """Returns form data with fields that were not in request popped."""
        d = super(JForm, self).data
        if self._formdata is None:
            return d
        keys = d.keys()
        keys = keys if isinstance(keys, list) else list(keys)
        for k in keys:
            if k not in self._formdata:
                del d[k]
        return d
{% endcapture %}

<pre><code class="py">{{ c2 | escape }}</code></pre>

When `form.data` is requested, `JForm` ignores any property that was not
included in the original form creation.


## Usage:

{% capture c3 %}import wtforms

class MyBaseForm(JForm):
    pass

class FooForm(MyBaseForm):
    foo = wtforms.StringField('Foo')
    bar = wtforms.IntegerField('Bar')
    is_baz = wtforms.BooleanField('IzBaz')

fd = DummyMultiDict({'foo': 'ayy'})
f = Foo(fd)

print(f.data) # {'foo': 'ayy'}
{% endcapture %}

<pre><code class="py">{{ c3 | escape }}</code></pre>

Happy coding. It also works nicely with other wrappers such as [flask-wtf](https://github.com/lepture/flask-wtf). Simply change to


    class MyBaseForm(JForm, flask_wtf.Form):
        pass


Update: I did not want to publish this as a pip-package because ... I is lazy.
