---
layout: post
title: "AngularJS: Reducing Initial Load Time by Preloading Data"
permalink: /p/angular-prefetch/
tags: [web, JavaScript]
---

## Background

Most of the web has transitioned into client-side heavy development using
various JavaScript frameworks. More and more sites are being built in
the "Single Page Application" (SPA) fashion. This puts more work on the user's browser
and less work on the servers. The shift keeps front-end and back-end development
separate and isolated, allowing clients to present information in different ways without
relying on server-side rendering. Additionally, most of smartphones these days
are more powerful than servers that serve thousands of users.

[ [Source Code](https://plnkr.co/edit/og8aqTSLU6IrpaHqX56O)
| [Demo](https://run.plnkr.co/plunks/og8aqTSLU6IrpaHqX56O) ]

One of the most common issues with this paradigm is the slow initial load time
of the page. Typically, the page is loaded very quickly only to show a spinning
icon for some time until the actual content is loaded. Here's the diagram
that explains the request cycle:

[![](//i.imgur.com/nQjRqXel.png)](//i.imgur.com/nQjRqXe.png)


1. Client requests the page. Server responds with some bootstrapped and
non-rendered HTML. (green)
2. Client requests for static assets, including the JavaScript files and HTML templates. (blue)
3. Application makes one or more HTTP calls to fetch various resources. (yellow)
4. The loaded resources are displayed to the user.

These steps are sequential. As one can imagine, the initial load time
can be very slow if the user has a slow internet connection and/or the user
is far away from the website data centers.

By looking at the diagram, let's assume the `api/products/123` is the highest
priority HTTP call. We want to show the result of it as soon as it is ready
while other HTTP calls may still be pending. If we somehow include the response of
this endpoint with the initial HTML page or `app.js`, the extra HTTP call
would not be necessary.

## AngularJS and Angular UI Router

[AngularJS](https://angularjs.org/) is a popular and somewhat opinionated
JavaScript framework that is backed by Google.
(Fun fact: the Angular team sat one floor below my team at Google!)

>AngularJS lets you write client-side web applications as if you had a smarter browser.
>It lets you use good old HTML (or HAML, Jade and friends!) as your template language and
>lets you extend HTML’s syntax to express your application’s components clearly and succinctly.
>It automatically synchronizes data from your UI (view) with your JavaScript objects (model)
>through 2-way data binding.


If you are building an _AngularJS 1.x_ app, you probably are already familiar with
[AngularUI Router](https://github.com/angular-ui/ui-router). It is generally
more flexible than the standard [`$route`](https://docs.angularjs.org/api/ngRoute/service/$route)
service. I suggest looking into it if you have not done so.
The example in this post uses AngularUI Router.

A typical application route setup might look something like this:

{% capture c0 %}$stateProvider
    .state('main', {
        abstract: true,
        resolve: {
            user: function (UserService) {
                return UserService.me();
            }
        }
    .state('main.product', {
        url: '/products/{id}'
        resolve: {
            product: function ($stateParams, ItemService) {
                return ItemService.get($stateParams.id);
            }
        }
{% endcapture %}

<pre><code class="js">{{ c0 | escape }}</code></pre>

The `user` and `product` properties are "resolved" (by making HTTP calls for example)
before the state transition happens.

## Preload (Prefetch) Data

The solution I propose is simple but somewhat ugly to implement.
It minimizes the initial load time, which is the best user experience.

1. When the server receives a request to `website.com/products/123`, it makes
an API call to `api/products/123` on behalf of the client.
This will be significantly faster than the client calling the API endpoint because
the data centers are typically close to each other with much faster network speeds.
2. The result from `api/products/123` is appended to the original as URL parameter
(let's call it `_data`). The URL change to: `website.com/products/123?_data={...}`
3. The client handles the case when this URL parameter is present.

**Important**: In order for the proposed solution to work, the application
needs to have
[html5Mode](https://code.angularjs.org/1.4.7/docs/api/ng/provider/$locationProvider#html5Mode)
enabled. This is because anything after the `#` portion of the URL is not sent
to the server (read more: [fragment identifier](https://en.wikipedia.org/wiki/Fragment_identifier)).
The default hashbang method uses the `#`. You should have strong reasons if
you do not already have html5Mode enabled.

In this scenario, we are only working with `api/products/123`. The solution
can be used in 100 different ways, including but not limited to calling multiple APIs or
including some placeholder data.

First, let's make that URL parameter an optional parameter (`_data`).

{% capture c1 %}class Config {
    constructor($locationProvider, $stateProvider, $urlRouterProvider) {
        $locationProvider.html5Mode(true);

        $urlRouterProvider.otherwise('/');

        $stateProvider
            .state('default', {
                url: '/',
                templateUrl: 'main.html'
            })
            .state('item', {
                url: '/items/{id}?_data',
                controller: 'Item as item',
                templateUrl: 'item.html',
                reloadOnSearch: false
        });
    }
}
{% endcapture %}

<pre><code class="js">{{ c1 | escape }}</code></pre>

It makes sense to remove the `_data` parameter from the URL as soon as the
application reads so the user does not share or bookmark the full URL
(and to make the URL look pretty). The `reloadOnSearch: false` option prevents
the state from being reloaded when this happens.

Now, the client logic skips the API call to get product information
when the `_data` parameter is present.

{% capture c2 %}class Item {
    constructor($log, $state, $stateParams, ItemService) {
        this.title = 'Loading..';
        this._log = $log;
        this._state = $state;
        this._stateParams = $stateParams;
        this._ItemService = ItemService;

        this.init();
    }

    init() {
        let context = this;

        this._log.debug(this._stateParams);

        // Pre-fetched data can come as a URL parameter (`_data`).
        var data = angular.fromJson(this._stateParams._data);

        if (data) {
            // Remove `_data` parameter from URL.
            this._state.go('.', {_data: null}, {location: 'replace'});

            return ready(data);
        }

        this._ItemService.fetchItem(this._stateParams.id).then(ready, ready);

        function ready(data) {
            context.title = data.name;
            context.item = data;
        }  
    }
}

class ItemService {
    constructor($timeout) {
        this._timeout = $timeout;
    }

    fetchItem(id) {
      return this._timeout(() => {
          return {id: id, name: 'TV', description: 'Fetched from the backend'};
      }, 1500);
    }
}
{% endcapture %}

<pre><code class="js">{{ c2 | escape }}</code></pre>

This method reduces the number of HTTP requests made by the application.
However, I do not recommend doing this everywhere in the application as it
introduces logic code both in the back-end endpoint and in the Angular
application.

**Full source and demo can be found [here](https://run.plnkr.co/plunks/og8aqTSLU6IrpaHqX56O).**

## Conclusion

We have looked at one of the biggest concerns against modern client-side heavy
development -- slower initial load times. We proposed a back-end agnostic solution
for the problem in AngularJS that makes use of URL parameters. Code samples for
both front-end and back-end have been provided.

## Bonus: Flask Back-end Endpoint Handler

Below is the quick implementation of step-1 and step-2 of the proposed solution above,
in [Flask](http://flask.pocoo.org/).
I'm sure a similar approach can be implemented in other frameworks.

{% capture c3 %}try:
    from urllib.parse import urlencode
except:
    from urllib import urlencode

import re
import flask as f

app = f.Flask(__name__)
ITEMS_RE = re.compile(r'^items/(\d+)')

@app.route('/app/', defaults={'path': ''})
@app.route('/app/<path:path>')
def index(path):
    """Main application entry. Let's assume our app is served at `/app`."""
    m = ITEMS_RE.match(path)
    if m and not f.request.args.get('_data'):
        data = get_item(int(m.groups()[0]))
        args = dict(f.request.args)
        args['_data'] = f.json.dumps(data)
        url = '{}?{}'.format(f.request.base_url, urlencode(args, doseq=True))
        return f.redirect(url)
    return 'return index.html'


def get_item(ident):
    return {'id': ident, 'name': 'fake', 'description': 'fake api return'}


if __name__ == '__main__':
    app.run(debug=True)
{% endcapture %}

<pre><code class="py">{{ c3 | escape }}</code></pre>
