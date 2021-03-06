django-cors-headers
===================

A Django App that adds CORS (Cross-Origin Resource Sharing) headers to
responses.

Although JSON-P is useful, it is strictly limited to GET requests. CORS
builds on top of ``XmlHttpRequest`` to allow developers to make cross-domain
requests, similar to same-domain requests. Read more about it here:
http://www.html5rocks.com/en/tutorials/cors/

.. image:: https://travis-ci.org/ottoyiu/django-cors-headers.svg?branch=master
   :target: https://travis-ci.org/ottoyiu/django-cors-headers


Requirements
------------

Tested with all combinations of:

* Python: 2.7, 3.5
* Django: 1.8, 1.9, 1.10

Setup
-----

Install from **pip**:

.. code-block:: sh

    pip install django-cors-headers

and then add it to your installed apps:

.. code-block:: python

    INSTALLED_APPS = (
        ...
        'corsheaders',
        ...
    )

You will also need to add a middleware class to listen in on responses:

.. code-block:: python

    MIDDLEWARE = [  # Or MIDDLEWARE_CLASSES on Django < 1.10
        ...
        'corsheaders.middleware.CorsMiddleware',
        'django.middleware.common.CommonMiddleware',
        ...
    ]

``CorsMiddleware`` should be placed as high as possible, especially before any
middleware that can generate responses such as Django's ``CommonMiddleware`` or
Whitenoise's ``WhiteNoiseMiddleware``. If it is not before, it will not be able
to add the CORS headers to these responses.

Also if you are using ``CORS_REPLACE_HTTPS_REFERER`` it should be placed before
Django's ``CsrfViewMiddleware`` (see more below).

Configuration
-------------

Configure the middleware's behaviour in your Django settings. You must add the
hosts that are allowed to do cross-site requests to
``CORS_ORIGIN_WHITELIST``, or set ``CORS_ORIGIN_ALLOW_ALL`` to ``True``
to allow all hosts.

``CORS_ORIGIN_ALLOW_ALL``
~~~~~~~~~~~~~~~~~~~~~~~~~
If ``True``, the whitelist will not be used and all origins will be accepted.
Defaults to ``False``.

``CORS_ORIGIN_WHITELIST``
~~~~~~~~~~~~~~~~~~~~~~~~~

A list of origin hostnames that are authorized to make cross-site HTTP
requests. The value ``'null'`` can also appear in this list, and will match the
``Origin: null`` header that is used in `"privacy-sensitive contexts"
<https://tools.ietf.org/html/rfc6454#section-6>`_, such as when the client is
running from a ``file://`` domain. Defaults to ``[]``.

Example:

.. code-block:: python

    CORS_ORIGIN_WHITELIST = (
        'google.com',
        'hostname.example.com',
        'localhost:8000',
        '127.0.0.1:9000'
    )


``CORS_ORIGIN_REGEX_WHITELIST``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A list of regexes that match origin regex list of origin hostnames that are
authorized to make cross-site HTTP requests. Defaults to ``[]``. Useful when
``CORS_ORIGIN_WHITELIST`` is impractical, such as when you have a large
number of subdomains.

Example:

.. code-block:: python

    CORS_ORIGIN_REGEX_WHITELIST = (r'^(https?://)?(\w+\.)?google\.com$', )

--------------

The following are optional settings, for which the defaults probably suffice.

``CORS_URLS_REGEX``
~~~~~~~~~~~~~~~~~~~

A regex which restricts the URL's for which the CORS headers will be sent.
Defaults to ``r'^.*$'``, i.e. match all URL's. Useful when you only need CORS
on a part of your site, e.g. an API at ``/api/``.

Example:

.. code-block:: python

    CORS_URLS_REGEX = r'^/api/.*$'

``CORS_ALLOW_METHODS``
~~~~~~~~~~~~~~~~~~~~~~

A list of HTTP verbs that are allowed for the actual request. Defaults to:

.. code-block:: python

    CORS_ALLOW_METHODS = (
        'DELETE',
        'GET',
        'OPTIONS',
        'PATCH',
        'POST',
        'PUT',
    )

The default can be imported as ``corsheaders.defaults.default_methods`` so you
can just extend it with your custom methods. This allows you to keep up to date
with any future changes. For example:

.. code-block:: python

    from corsheaders.defaults import default_methods

    CORS_ALLOW_METHODS = default_methods + (
        'POKE',
    )

``CORS_ALLOW_HEADERS``
~~~~~~~~~~~~~~~~~~~~~~

The list of non-standard HTTP headers that can be used when making the actual
request. Defaults to:

.. code-block:: python

    CORS_ALLOW_HEADERS = (
        'accept',
        'accept-encoding',
        'authorization',
        'content-type',
        'dnt',
        'origin',
        'user-agent',
        'x-csrftoken',
        'x-requested-with',
    )

The default can be imported as ``corsheaders.defaults.default_headers`` so you
can extend it with your custom headers. This allows you to keep up to date with
any future changes. For example:

.. code-block:: python

    from corsheaders.defaults import default_headers

    CORS_ALLOW_HEADERS = default_headers + (
        'my-custom-header',
    )

``CORS_EXPOSE_HEADERS``
~~~~~~~~~~~~~~~~~~~~~~~

The list of HTTP headers that are to be exposed to the browser. Defaults to
``[]``.


``CORS_PREFLIGHT_MAX_AGE``
~~~~~~~~~~~~~~~~~~~~~~~~~~

The number of seconds a client/browser can cache the preflight response. If
this is 0 (or any falsey value), no max age header will be sent. Defaults to
``86400`` (one day).


**Note:** A preflight request is an extra request that is made when making a
"not-so-simple" request (e.g. ``Content-Type`` is not
``application/x-www-form-urlencoded``) to determine what requests the server
actually accepts. Read more about it in the `HTML 5 Rocks CORS tutorial
<https://www.html5rocks.com/en/tutorials/cors/>`_.

``CORS_ALLOW_CREDENTIALS``
~~~~~~~~~~~~~~~~~~~~~~~~~~

If ``True``, cookies will be allowed to be included in cross-site HTTP
requests. Defaults to ``False``.

``CORS_REPLACE_HTTPS_REFERER``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If ``True``, the ``HTTP_REFERER`` header will get replaced when CORS checks
pass, so that the Django CSRF middleware checks work with HTTPS. Defaults to
``False``.

**Note:** With this feature enabled, you also need to add
``corsheaders.middleware.CorsPostCsrfMiddleware`` after
``django.middleware.csrf.CsrfViewMiddleware`` in your ``MIDDLEWARE_CLASSES`` to
undo the header replacement:

.. code-block:: python

    MIDDLEWARE = [
        ...
        'corsheaders.middleware.CorsMiddleware',
        ...
        'django.middleware.csrf.CsrfViewMiddleware',
        'corsheaders.middleware.CorsPostCsrfMiddleware',
        ...
    ]

``CORS_MODEL``
~~~~~~~~~~~~~~

If set, this should be the path to a model to look up allowed origins, in the
form ``app.modelname``. Defaults to ``None``.

The model should inherit from ``corsheaders.models.AbstractCorsModel`` and specify
the allowed origin in the ``CharField`` called ``cors``.

Signals
-------

If you have a use case that requires more than just the above configuration,
you can attach code to check if a given request should be allowed. For example,
this can be used to read the list of origins you allow from a model. Attach any
number of handlers to the ``check_request_enabled``
`Django signal <https://docs.djangoproject.com/en/1.10/ref/signals/>`_, which
provides the ``request`` argument (use ``**kwargs`` in your handler to protect
against any future arguments being added). If any handler attached to the
signal returns a truthy value, the request will be allowed.

For example you might define a handler like this:

.. code-block:: python

    # myapp/handlers.py
    from corsheaders.signals import check_request_enabled

    from .models import MySite

    def cors_allow_mysites(sender, request, **kwargs):
        return MySite.objects.filter(host=request.host).exists()

    check_request_enabled.connect(cors_allow_mysites)

Then connect it at app ready time using a `Django AppConfig
<https://docs.djangoproject.com/en/1.10/ref/applications/>`_:

.. code-block:: python

    # myapp/__init__.py

    default_app_config = 'myapp.apps.MyAppConfig'

.. code-block:: python

    # myapp/apps.py

    from django.apps import AppConfig

    class MyAppConfig(AppConfig):
        name = 'myapp'

        def ready(self):
            # Makes sure all signal handlers are connected
            from . import handlers  # noqa


A common use case for the signal is to allow *all* origins to access a subset
of URL's, whilst allowing a normal set of origins to access *all* URL's. This
isn't possible using just the normal configuration, but it can be achieved with
a signal handler.

First set ``CORS_ORIGIN_WHITELIST`` to the list of trusted origins that are
allowed to access every URL, and then add a handler to
``check_request_enabled`` to allow CORS regardless of the origin for the
unrestricted URL's. For example:

.. code-block:: python

    # myapp/handlers.py
    from corsheaders.signals import check_request_enabled

    def cors_allow_api_to_everyone(sender, request, **kwargs):
        return request.path.startswith('/api/')

    check_request_enabled.connect(cors_allow_api_to_everyone)

Credits
-------

``django-cors-headers`` was created by Otto Yiu (`@ottoyiu
<https://github.com/ottoyiu>`_) and has been worked on by `25+ contributors
<https://github.com/ottoyiu/django-cors-headers/graphs/contributors>`_.
Thanks to every contributor, and if you want to get involved please don't
hesitate to make a pull request.
