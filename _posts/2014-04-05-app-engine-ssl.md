---
layout: post
title: App Engine and SSL
permalink: /p/gae-ssl
tags: [gae]
---

Google App Engine is a great platform to get things done fast.
However, it can be very unpleasant to work with due to its sandboxed environment
and close source code. Basic needs such as installing third-party
libraries can be tricky to install as well.

Getting one of the most
popular python libraries, [python-requests](http://docs.python-requests.org/en/latest/),
was particularly tricky to get it running and working with SSL connections.
I'll walk through how I fixed the issue.

Start by adding the library to the project:

{% capture c0 %}# From project root.
pip install -t lib/ requests
{% endcapture %}

<pre><code class="bash">{{ c0 | escape }}</code></pre>

The above command pip-installs the `requests` library into the `lib`
directory. This is where all the third-party libraries can be placed.

Now we need to let App Engine know about this. Create or modify the file
`appengine_config.py` in the root of the project.

{% capture c1%}from google.appengine.ext import vendor
vendor.add('lib')
{% endcapture %}

<pre><code class="python">{{ c1 | escape }}</code></pre>

`appengine_config.py` runs when a new instance is created. `vendor.add` adds the specified path to `$PYTHONPATH`.

At this point, most third-party libraries work just fine. However,
there's a bit of work that needs to be done to get `requests` working.

Head over to [http://localhost:8000/console]() and execute:

{% capture c2 %}import requests

r = requests.get('https://httpbin.org/status/200')
print(r.status_code)
{% endcapture %}

<pre><code class="python">{{ c2 | escape }}</code></pre>

In a normal Python environment, the code executes just fine printing a
200 status. But on GAE, the following exception occurs:

{% capture c3 %}Traceback (most recent call last):
  File
"/Applications/GoogleAppEngineLauncher.app/Contents/Resources/GoogleAppEngine-default.bundle/Contents/Resources/google_appengine/google/appengine/tools/devappserver2/python/request_handler.py",
line 225, in handle_interactive_request
    exec(compiled_code, self._command_globals)
  File "<string>", line 1, in <module>
  File ".../lib/requests/__init__.py", line 58, in <module>
    from . import utils
  File ".../lib/requests/utils.py", line 26, in <module>
    from .compat import parse_http_list as _parse_list_header
  File ".../lib/requests/compat.py", line 42, in <module>
    from .packages.urllib3.packages.ordered_dict import OrderedDict
  File ".../lib/requests/packages/__init__.py", line 95, in load_module
    raise ImportError("No module named '%s'" % (name,))
ImportError: No module named 'requests.packages.urllib3'
{% endcapture %}

<pre><code class="python">{{ c3 | escape }}</code></pre>

The issue goes away once the
[ssl](https://cloud.google.com/appengine/docs/python/sockets/ssl_support) library is included in `app.yaml`:

<pre>
<code class="nohighlight">libraries:
- name: ssl
  version: latest
</code></pre>

But wait, there's more! The code should now work remotely.
However, it still doesn't work on the development server.

{% capture c4 %}Traceback (most recent call last):
  File "/Applications/GoogleAppEngineLauncher.app/Contents/Resources/GoogleAppEngine-default.bundle/Contents/Resources/google_appengine/google/appengine/tools/devappserver2/python/request_handler.py", line 225, in handle_interactive_request
    exec(compiled_code, self._command_globals)
  File "<string>", line 3, in <module>
  File ".../lib/requests/api.py", line 68, in get
    return request('get', url, **kwargs)
  File ".../lib/requests/api.py", line 50, in request
    response = session.request(method=method, url=url, **kwargs)
  [...]
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/ssl.py", line 387, in wrap_socket
    ciphers=ciphers)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/ssl.py", line 141, in __init__
    ciphers)
TypeError: must be _socket.socket, not socket
{% endcapture %}

<pre><code class="python">{{ c4 | escape }}</code></pre>

The problem GAE has a "whitelist" of select standard libraries.
SSL (_ssl, _socket) is not one of them.
So, we need to tweak the sandbox environment (dangerous) carefully.
The below code uses the standard Python socket library instead of the GAE-provided
in the development environment. Modify `appengine_config.py`:

{% capture c5 %}import os

# Workaround the dev-environment SSL
#   http://stackoverflow.com/q/16192916/893652
if os.environ.get('SERVER_SOFTWARE', '').startswith('Development'):
    import imp
    import os.path
    from google.appengine.tools.devappserver2.python import sandbox

    sandbox._WHITE_LIST_C_MODULES += ['_ssl', '_socket']
    # Use the system socket.
    psocket = os.path.join(os.path.dirname(os.__file__), 'socket.py')
    imp.load_source('socket', psocket)
{% endcapture %}

<pre><code class="python">{{ c5 | escape }}</code></pre>

{% capture c6 %}INFO     2015-04-04 06:57:28,449 module.py:737] default: "POST / HTTP/1.1" 200 4
INFO     2015-04-04 06:57:46,868 connectionpool.py:735] Starting new HTTPS connection
    (1): httpbin.org
{% endcapture %}

<pre><code class="nohighlight">{{ c6 | escape }}</code></pre>

This solution mostly works, except for non-blocking sockets.
I haven't had a need for that yet :)
