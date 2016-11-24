---
layout: post
title: Automatically Download the Latest App Engine SDK
permalink: /p/gae-sdk/
tags: [gae]
---

**TL;DR**:
Google doesn't provide a permanent link to the latest SDK;
update check endpoint is not reliable;
I provide a snippet ([view](https://gist.github.com/Bekt/7cb68b12674b282c8d78))
to work around this.

Google provides an SDK for interacting with App Engine. `dev_appserver.py`
simulates the App Engine Python runtime environment locally.
`appcfg.py` is used to interact with App Engine to do tasks such as deploying
a new version of the application.

For continuous deployment / integration scripts such as those for
[CircleCI](https://circleci.com/docs/deploy-google-app-engine),
[Travis CI](https://github.com/travis-ci/dpl/issues/94),
or [Jenkins CI](https://jenkins-ci.org), it is usually necessary to download
the latest version of the Google App Engine SDK. Unfortunately, Google does not
provide a permanent link to the latest version for easy download.
It seems like a basic need. I talk about how to programatically get the latest
SDK in this post.

## UpdateCheck Endpoint

In `appengine/tools/sdk_update_checker.py`, I found the API endpoint for
checking for updates to the SDK: `https://appengine.google.com/api/updatecheck`.

{% capture c0 %}$ curl https://appengine.google.com/api/updatecheck

release: "1.9.19"
timestamp: 1424415497
api_versions: ['1']
supported_api_versions:
  python:
    api_versions: ['1']
  python27:
    api_versions: ['1']
  go:
    api_versions: ['go1']
  java7:
    api_versions: ['1.0']
{% endcapture %}

<pre><code class="bash">{{ c0 | escape }}</code></pre>

Great, this is all we need. Not so fast! But it is useful.

## SDK Download URL

On the App Engine SDK [downloads](https://cloud.google.com/appengine/downloads),
the only link to the latest SDK is something like:

`https://storage.googleapis.com/appengine-sdks/featured/google_appengine_1.9.20.zip`

Notice the version number is 1.9.20, not 1.9.19.

Using the Google Cloud Storage API, we can list all the files in the
`appengine-sdks/featured` bucket:

`https://www.googleapis.com/storage/v1/b/appengine-sdks/o?prefix=featured`

## Fetch the Latest SDK

Given the two points from above, we can can automatically check for the latest SDK
version and download the zip with that version.

{% capture c1 %}# script.bash
# DON'T COPY-PASTE, READ BELOW.

#!/usr/bin/env bash

API_CHECK=https://appengine.google.com/api/updatecheck
SDK_VERSION=$(curl -s $API_CHECK | awk -F '\"' '/release/ {print $2}')
SDK_URL=https://storage.googleapis.com/appengine-sdks/featured/google_appengine_$SDK_VERSION.zip

function download_sdk {
  echo ">>> Downloading..."
  curl -fo $HOME/gae.zip $SDK_URL || exit 1
  unzip -qd $HOME $HOME/gae.zip
}

function upload {
  $HOME/google_appengine/appcfg.py \
      --oauth2 update my-app
}

download_sdk
upload
{% endcapture %}

<pre><code class="bash">{{ c1 | escape }}</code></pre>

Easy, right? Ehhhh.

{% capture c2 %}./script.bash

>>> Downloading...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0

curl: (22) The requested URL returned error: 404 Not Found
{% endcapture %}

<pre><code class="bash">{{ c2 | escape }}</code></pre>

Apparently, the UpdateCheck endpoint does not always have the latest
version. Like wut. From my experience, it can take anywhere from 1-7 days for the
endpoint to return the latest version.
Additionally, once a new version becomes available,
previous download links become unavailable. The App Engine team acknowledged the [issue](https://code.google.com/p/googleappengine/issues/detail?id=11604)
five months ago but still has not addressed it.

## Revisiting SDK Download URL

Well, obviously at this point the only way to get the latest SDK version is to
parse the HTML content of the download page. We could also look through all
the files in the `featured` bucket and find the one correct file to download.
However, nobody should do that!
Since we don't necessarily need the latest version to simply deploy the project,
we can fallback to a slightly older SDK version (the one UpdateCheck returns).

As indicated on the downloads page, deprecated SDK downloads can be found at
the following link:

`https://console.developers.google.com/m/cloudstorage/b/appengine-sdks/o/deprecated/1919/google_appengine_1.9.19.zip`

which I found out is the same as:

`https://storage.googleapis.com/appengine-sdks/deprecated/1919/google_appengine_1.9.19.zip`

## New script.bash

So now we can write a script that fall-backs to the deprecated SDK URL if the first
URL does not work.

{% capture c3 %}# script.bash

#!/usr/bin/env bash

API_CHECK=https://appengine.google.com/api/updatecheck
SDK_VERSION=$(curl -s $API_CHECK | awk -F '\"' '/release/ {print $2}')
# Remove the dots.
SDK_VERSION_S=${SDK_VERSION//./}

SDK_URL=https://storage.googleapis.com/appengine-sdks/
SDK_URL_A="${SDK_URL}featured/google_appengine_${SDK_VERSION}.zip"
SDK_URL_B="${SDK_URL}deprecated/$SDK_VERSION_S/google_appengine_${SDK_VERSION}.zip"

function download_sdk {
  echo ">>> Downloading..."
  curl -fo $HOME/gae.zip $SDK_URL_A || \
      curl -fo $HOME/gae.zip $SDK_URL_B || \
      exit 1
  unzip -qd $HOME $HOME/gae.zip
}

function upload {
  echo ">>> Deploying..."
  $HOME/google_appengine/appcfg.py \
      --oauth2 update my-app
}

download_sdk
upload
{% endcapture %}

<pre><code class="bash">{{ c3 | escape }}</code></pre>

If we run the script again, it works as expected:
* Tries to download from the `features` folder first
* Otherwise, tries to download from the `deprecated` folder

## Conclusion

We came up with `script.bash` that automatically downloads the "latest"
Google App Engine Python/PHP SDK to make life easier. This is especially useful
during automated builds and deployments.

### References
0: https://code.google.com/p/googleappengine/issues/detail?id=11604
