.. include:: <s5defs.txt>
..
  Copyright 2011, Jason Kirtland
  License: Creative Commons Attribution-Share Alike 3.0

========================
decouple your everything
========================


with publish / subscribe
------------------------


publish / subscribe
===================

.. class:: incremental

- publish / emit / send
- events / signals
- subscribe / consume / connect


a method call
=============

Method calls are imperative: tell an object to do something

.. code-block:: python

  window.resize(x=640, y=480)

.. class:: incremental

 - event "resize"
 - receiver "window"
 - metadata ``x=640``, ``y=480``


events
======

Pub/Sub events are generally reactive: something happened or will happen

.. code-block:: python

  resized(window, x=640, y=480)

.. class:: incremental

 - event "resized"
 - sender "window"
 - metadata ``x=640``, ``y=480``


decoupled
=========

- subscribers don't know about publishers
- publishers don't know about subscribers


decoupled
=========

- at the simplest, events are just names
- a 3rd party (pub/sub library) brokers everything
- publishers don't need to import subscribers
- and vice-versa


python implementations
======================

- PyDispatcher / Louie
- PyPubSub
- Django signals
- blinker
- ...


introducing blinker
===================

- small, tuned implementation
- inspired by Django event api
- used in Flask


resized
=======

.. code-block:: pycon

  >>> from blinker import signal
  >>> window_resized = signal('window_resized')
  >>> window_resized.send('some window', x=640, y=480)
  []
  >>>


resized
=======

.. code-block:: pycon

  >>> def receiver(sender, **kw):
  ...     print "received %r %r" % (sender, kw)
  ...
  >>> window_resized.connect(receiver)
  >>> window_resized.send('some window', x=640, y=480)
  received 'some window' {'x': 640, 'y': 480}
  [(<function receiver at 0x3243b0>, None)]


blinker
=======

- publish/subscribe is multicast
- subscribe to all emissions, or only those of some senders
- long or short term subscriptions


why?
====

- instrumentation
- open up 3rd party extension points
- plug-ins and extensions
- unexpected wins


instrumentation
===============

.. code-block:: python

  from blinker import signal

  started = signal('request-started')
  ended = signal('request-ended')

  def my_wsgi_app(environ, start_response):
      started.send(app, environ=environ)
      response = do_something(...)
      ended.send(app, environ=environ, response=response)
      return response


...and then
===========

.. code-block:: python

  from blinker import signal

  started = signal('request-started')
  ended = signal('request-ended')

- what is the request rate and latency?
- "we need to log referrer on homepage requests"
- configure logging formats
- release resources
- ...


open up
=======

.. code-block:: python

  from blinker import signal
  from sqlalchemy import ConnectionProxy


  sql_executed = signal('sql-executed')

  class Watcher(ConnectionProxy):

      def cursor_execute(self, execute, cursor, statement, parameters,
                         context, executemany):
          if not sql_executed.receivers:
              return execute(cursor, statement, parameters, context)
          start = time.time()
          r = execute(cursor, statement, parameters, context)
          sql_executed.send(cursor, statement=statement, parameters=parameters,
                            elapsed=(time.time() - start))
          return r


remix and extend
================

Extend the gunicorn!

.. code-block:: python

  def my_app(environ, start_response):
     ...

  def pre_request(worker, req):
      """Called just before a worker processes the request."""

  def post_request(worker, req):
      """Called after a worker processes the request."""


remix and extend
================

.. code-block:: python

  from blinker import signal

  request_started = signal('request_started')
  current_environ = None

  @request_started.connect
  def stash_environ(app, environ):
      global current_environ
      current_environ = environ

  def pre_request(worker, req):
      signal('pre-request').send(worker, req=req)

  def post_request(worker, req):
      signal('post-request').send(worker, req=req, environ=current_environ)


it's your app
=============

.. code-block:: ini

  [auto-import]
  sql_engine = idealist.model.utilities.engine
  mail = idealist.mail
  traps = idealist.webapp.traps


it's your app
=============

.. code-block:: python


  @config.post_config_load.connect
  def autoinstall(config):
      requested = config.slice('webapp.traps')
      for trap, value in requested.items():
          try:
              enabler = traps[trap]
          except KeyError:
              raise LookupError("Unknown trap webapp.traps.%s" % trap)
          else:
              enabler(value)


tips
====

- send signals at interesting transitions
- unit of data/request processing started, ended
- include useful locals
- think about when to fire... before, after, around


thanks
======

- search pypi for 'events', 'signals', 'dispatch'...
- django, flask docs on signals

.. class:: center

  jek@discorporate.us



