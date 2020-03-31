.. _rq: http://python-rq.org


.. _deployment:

Architecture and Deploying
==========================

Pulp's architecture has three components to it: a REST API, a content serving application, and the
tasking system. Each component can be horizontally scaled for both high availability and/or
additional capacity for that part of the architecture.

REST API
--------

Pulp's REST API is a Django application which runs under any WSGI server. It serves the following
things:

* The REST API hosted at ``/pulp/api/v3/``
* The browse-able documentation at ``/pulp/api/v3/docs/``
* Any viewsets or views provided by plugins
* Static content used by Django, e.g. images used by the browse-able API. This is not Pulp content.

.. note::

   A simple, but limited way to run the REST API as a standalone service using the built-in Django
   runserver. The ``pulpcore-manager`` command is ``manage.py`` configured with the
   ``DJANGO_SETTINGS_MODULE="pulpcore.app.settings"``. Run the simple webserver with::

      $ pulpcore-manager runserver 24817


The REST API can be deployed with any any WSGI webserver like a normal Django application. See the
`Django deployment docs <https://docs.djangoproject.com/en/2.1/howto/deployment/wsgi/>`_ for more
information.

Content Serving Application
---------------------------

An aiohttp.server based application that serves content to clients. The content could be
:term:`Artifacts<artifact>` already downloaded and saved in Pulp, or
:term:`on-demand content units<on-demand content>`. When serving
:term:`on-demand content units<on-demand content>` the downloading also happens from within this
component as well.

.. note::

   Pulp installs a script that lets you run the content serving app as a standalone service as
   follows:::

      $ pulpcore-content

The content serving application can be deployed like any aiohttp.server application. See the
`aiohttp Deployment docs <https://aiohttp.readthedocs.io/en/stable/deployment.html>`_ for more
information.


Tasking System
--------------

Pulp's tasking system has two components: a resource manager and workers, all of which are run using
`rq`_.

Worker
  Pulp workers perform most tasks "run" by the tasking system including long-running tasks like
  synchronize and short-running tasks like a Distribution update. Each worker handles one task at a
  time, and additional workers provide more concurrency. Workers auto-name and are auto-discovered,
  so they can be started and stopped without notifying Pulp.

Resource Manager
  A different type of Pulp worker that plays a coordinating role for the tasking system. You must
  run exactly one of these for Pulp to operate correctly. The ``resource-manager`` is identified by
  configuring using exactly the name ``resource-manager`` with the ``-n 'resource_manager'`` option.

  *N* ``resource-manager`` rq processes can be started with 1 being active and *N-1* being passive.
  The *N-1* will exit and should be configured to auto-relaunch with either systemd, supervisord, or
  k8s.

.. note::

   Pulp serializes tasks that are unsafe to run in parallel, e.g. a sync and publish operation on
   the same repo should not run in parallel. Generally tasks are serialized at the "repo" level, so
   if you start *N* workers you can process *N* repo sync/modify/publish operations concurrently.


Static Content
--------------

When browsing the REST API or the browsable documentation with a web browser, for a good experience,
you'll need static content to be served.

In Development
^^^^^^^^^^^^^^

If using the built-in Django webserver and your settings.yaml has ``DEBUG: True`` then static
content is automatically served for you.

In Production
^^^^^^^^^^^^^

Collect all of the static content into place using the ``collectstatic`` command. The
``pulpcore-manager`` command is ``manage.py`` configured with the
``DJANGO_SETTINGS_MODULE="pulpcore.app.settings"``. Run ``collectstatic`` as follows::

    $ pulpcore-manager collectstatic

