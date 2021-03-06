Decorators
==========

`The uWSGI API <http://uwsgi-docs.readthedocs.org/en/latest/PythonModule.html>`_ is very low-level, as it must be language-independent.

That said, being too low-level is not a Good Thing for many languages, such as Python.

Decorators are, in our humble opinion, one of the more kick-ass features of Python, so in the uWSGI source tree you will find a module exporting a bunch of decorators that cover a good part of the uWSGI API.




Notes
-----

Signal-based decorators execute the signal handler in the first available worker.
If you have enabled the spooler you can execute the signal handlers in it, leaving workers free to manage normal requests. Simply pass ``target='spooler'`` to the decorator.

.. code-block:: py

    @timer(3, target='spooler')
    def hello(signum):
        print("hello")


Example: a Django session cleaner and video encoder
---------------------------------------------------

Let's define a :file:`tasks.py` module and put it in the Django project directory.

.. code-block:: py

    import os
    from django.contrib.sessions.models import Session
    from django_uwsgi.decorators import cron, spool

    @cron(40, 2, -1, -1, -1)
    def clear_django_session(num):
        print("it's 2:40 in the morning: clearing django sessions")
        Session.objects.all().delete()

    @spool
    def encode_video(arguments):
        os.system("ffmpeg -i \"%s\" image%%d.jpg" % arguments['filename'])

The session cleaner will be executed every day at 2:40, to enqueue a video encoding we simply need to spool it from somewhere else.

.. code-block:: py

    from tasks import encode_video

    def index(request):
        # launching video encoding
        encode_video.spool(filename=request.POST['video_filename'])
        return render_to_response('enqueued.html')

Now run uWSGI with the spooler enabled:

.. code-block:: ini

    [uwsgi]
    ; a couple of placeholder
    django_projects_dir = /var/www/apps
    my_project = foobar
    ; chdir to app project dir and set pythonpath
    chdir = %(django_projects_dir)/%(my_project)
    pythonpath = %(django_projects_dir)
    ; load django
    module = django.core.handlers:WSGIHandler()
    env = DJANGO_SETTINGS_MODULE=%(my_project).settings
    ; enable master
    master = true
    ; 4 processes should be enough
    processes = 4
    ; enable the spooler (the mytasks dir must exist!)
    spooler = %(chdir)/mytasks
    ; load the task.py module
    import = task
    ; bind on a tcp socket
    socket = 127.0.0.1:3031

The only especially relevant option is the ``import`` one. It works in the same way as ``module`` but skips the WSGI callable search.
You can use it to preload modules before the loading of WSGI apps. You can specify an unlimited number of '''import''' directives.


django_uwsgi.decorators API reference
-----------------------------

.. default-domain:: py

.. module:: django_uwsgi.decorators

.. function:: postfork(func)

   uWSGI is a preforking (or "fork-abusing") server, so you might need to execute a fixup task after each ``fork()``. The ``postfork`` decorator is just the ticket.
   You can declare multiple ``postfork`` tasks. Each decorated function will be executed in sequence after each ``fork()``.

   .. code-block:: py

      @postfork
      def reconnect_to_db():
          myfoodb.connect()

      @postfork
      def hello_world():
          print("Hello World")

.. function:: spool(func)

   The uWSGI `spooler <http://uwsgi-docs.readthedocs.org/en/latest/Spooler.html>`_ can be very useful. Compared to Celery or other queues it is very "raw". The ``spool`` decorator will help!

   .. code-block:: py

      @spool
      def a_long_long_task(arguments):
          print(arguments)
          for i in xrange(0, 10000000):
              time.sleep(0.1)

      @spool
      def a_longer_task(args):
          print(args)
          for i in xrange(0, 10000000):
              time.sleep(0.5)

      # enqueue the tasks
      a_long_long_task.spool(foo='bar',hello='world')
      a_longer_task.spool({'pippo':'pluto'})

   The functions will automatically return ``uwsgi.SPOOL_OK`` so they will be executed one time independently by their return status.

.. XXX: What does the above mean?

.. function:: spoolforever(func)

   Use ``spoolforever`` when you want to continuously execute a spool task.
   A ``@spoolforever`` task will always return ``uwsgi.SPOOL_RETRY``.

   .. code-block:: py

     @spoolforever
     def a_longer_task(args):
         print(args)
         for i in xrange(0, 10000000):
             time.sleep(0.5)

     # enqueue the task
     a_longer_task.spool({'pippo':'pluto'})



.. function:: spoolraw(func)

  Advanced users may want to control the return value of a task.


   .. code-block:: py

      @spoolraw
      def a_controlled_task(args):
          if args['foo'] == 'bar':
              return uwsgi.SPOOL_OK
          return uwsgi.SPOOL_RETRY

      a_controlled_task.spool(foo='bar')

.. function:: rpc("name", func)

   uWSGI `RPC <http://uwsgi-docs.readthedocs.org/en/latest/RPC.html>`_ is the fastest way to remotely call functions in applications hosted in uWSGI instances. You can easily define exported functions with the @rpc decorator.

   .. code-block:: py

      @rpc('helloworld')
      def ciao_mondo_function():
          return "Hello World"

.. function:: signal(num)(func)

   You can register signals for the `signal framework <http://uwsgi-docs.readthedocs.org/en/latest/Signals.html>`_ in one shot.

   .. code-block:: py

       @signal(17)
       def my_signal(num):
           print("i am signal %d" % num)

.. function:: timer(interval, func)

   Execute a function at regular intervals.

   .. code-block:: py

      @timer(3)
      def three_seconds(num):
          print("3 seconds elapsed")

.. function:: rbtimer(interval, func)

   Works like @timer but using red black timers.

.. XXX: What the hell does _that_ mean?

.. function:: cron(min, hour, day, mon, wday, func)


   Easily register functions for the `CronInterface <http://uwsgi-docs.readthedocs.org/en/latest/CronInterface.html>`_.

   .. code-block:: py

      @cron(59, 3, -1, -1, -1)
      def execute_me_at_three_and_fiftynine(num):
          print("it's 3:59 in the morning")

   Since 1.2, a new syntax is supported to simulate ``crontab``-like intervals (every Nth minute, etc.). ``*/5 * * * *`` can be specified in uWSGI like thus:

   .. code-block:: py

      @cron(-5, -1, -1, -1, -1)
      def execute_me_every_five_min(num):
          print("5 minutes, what a long time!")

.. function:: filemon(path, func)

   Execute a function every time a file/directory is modified.

   .. code-block:: py

        @filemon("/tmp")
        def tmp_has_been_modified(num):
            print("/tmp directory has been modified. Great magic is afoot")

.. function:: erlang(process_name, func)

   Map a function as an `Erlang <http://uwsgi-docs.readthedocs.org/en/latest/Erlang.html>` process.

   .. code-block:: py

        @erlang('foobar')
        def hello():
            return "Hello"


.. function:: thread(func)

    Mark function to be executed in a separate thread.

    .. important:: Threading must be enabled in uWSGI with the ``enable-threads`` or ``threads <n>`` option.

    .. code-block:: py

        @thread
        def a_running_thread():
            while True:
                time.sleep(2)
                print("i am a no-args thread")

        @thread
        def a_running_thread_with_args(who):
            while True:
                time.sleep(2)
                print("Hello %s (from arged-thread)" % who)

        a_running_thread()
        a_running_thread_with_args("uWSGI")

    You may also combine ``@thread`` with ``@postfork`` to spawn the postfork handler in a new thread in the freshly spawned worker.

    .. code-block:: py

        @postfork
        @thread
        def a_post_fork_thread():
            while True:
                time.sleep(3)
                print("Hello from a thread in worker %d" % uwsgi.worker_id())

.. function:: lock(func)

    This decorator will execute a function in fully locked environment, making it impossible for other workers or threads (or the master, if you're foolish or brave enough) to run it simultaneously.
    Obviously this may be combined with @postfork.

    .. code-block:: py

        @lock
        def dangerous_op():
            print("Concurrency is for fools!")


.. function:: mulefunc([mulespec], func)

    Offload the execution of the function to a `mule <http://uwsgi-docs.readthedocs.org/en/latest/Mules.html>`. When the offloaded function is called, it will return immediately and execution is delegated to a mule.

    .. code-block:: py

        @mulefunc
        def i_am_an_offloaded_function(argument1, argument2):
            print argument1,argument2

    You may also specify a mule ID or mule farm to run the function on. Please remember to register your function with a uwsgi import configuration option.

    .. code-block:: py

        @mulefunc(3)
        def on_three():
            print "I'm running on mule 3."

        @mulefunc('old_mcdonalds_farm')
        def on_mcd():
            print "I'm running on a mule on Old McDonalds' farm."

.. function:: harakiri(time, func)

    Starting from uWSGI 1.3-dev, a customizable secondary :term:`harakiri` subsystem has been added. You can use this decorator to kill a worker if the given call is taking too long.

    .. code-block:: py

        @harakiri(10)
        def slow_function(foo, bar):
            for i in range(0, 10000):
                for y in range(0, 10000):
                    pass

        # or the alternative lower level api

        uwsgi.set_user_harakiri(30) # you have 30 seconds. fight!
        slow_func()
        uwsgi.set_user_harakiri(0) # clear the timer, all is well


.. _Emperor: http://uwsgi-docs.readthedocs.org/en/latest/Emperor.html
