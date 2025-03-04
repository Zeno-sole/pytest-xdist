

.. image:: http://img.shields.io/pypi/v/pytest-xdist.svg
    :alt: PyPI version
    :target: https://pypi.python.org/pypi/pytest-xdist

.. image:: https://img.shields.io/conda/vn/conda-forge/pytest-xdist.svg
    :target: https://anaconda.org/conda-forge/pytest-xdist

.. image:: https://img.shields.io/pypi/pyversions/pytest-xdist.svg
    :alt: Python versions
    :target: https://pypi.python.org/pypi/pytest-xdist

.. image:: https://github.com/pytest-dev/pytest-xdist/workflows/build/badge.svg
    :target: https://github.com/pytest-dev/pytest-xdist/actions

.. image:: https://img.shields.io/badge/code%20style-black-000000.svg
    :target: https://github.com/ambv/black

xdist: pytest distributed testing plugin
========================================

The `pytest-xdist`_ plugin extends pytest with some unique
test execution modes:

* test run parallelization_: if you have multiple CPUs or hosts you can use
  those for a combined test run.  This allows to speed up
  development or to use special resources of `remote machines`_.


* ``--looponfail``: run your tests repeatedly in a subprocess.  After each run
  pytest waits until a file in your project changes and then re-runs
  the previously failing tests.  This is repeated until all tests pass
  after which again a full run is performed.

* `Multi-Platform`_ coverage: you can specify different Python interpreters
  or different platforms and run tests in parallel on all of them.

Before running tests remotely, ``pytest`` efficiently "rsyncs" your
program source code to the remote place.  All test results
are reported back and displayed to your local terminal.
You may specify different Python versions and interpreters.

If you would like to know how pytest-xdist works under the covers, checkout
`OVERVIEW <https://github.com/pytest-dev/pytest-xdist/blob/master/OVERVIEW.md>`_.


**NOTE**: due to how pytest-xdist is implemented, the ``-s/--capture=no`` option does not work.


Installation
------------

Install the plugin with::

    pip install pytest-xdist


To use ``psutil`` for detection of the number of CPUs available, install the ``psutil`` extra::

    pip install pytest-xdist[psutil]


.. _parallelization:

Speed up test runs by sending tests to multiple CPUs
----------------------------------------------------

To send tests to multiple CPUs, use the ``-n`` (or ``--numprocesses``) option::

    pytest -n NUMCPUS

Pass ``-n auto`` to use as many processes as your computer has CPU cores. This
can lead to considerable speed ups, especially if your test suite takes a
noticeable amount of time.

If a test crashes a worker, pytest-xdist will automatically restart that worker
and report the test’s failure. You can use the ``--max-worker-restart`` option
to limit the number of worker restarts that are allowed, or disable restarting
altogether using ``--max-worker-restart 0``.

By default, using ``--numprocesses`` will send pending tests to any worker that
is available, without any guaranteed order. You can change the test
distribution algorithm this with the ``--dist`` option. It takes these values:

* ``--dist no``: The default algorithm, distributing one test at a time.

* ``--dist loadscope``: Tests are grouped by **module** for *test functions*
  and by **class** for *test methods*. Groups are distributed to available
  workers as whole units. This guarantees that all tests in a group run in the
  same process. This can be useful if you have expensive module-level or
  class-level fixtures. Grouping by class takes priority over grouping by
  module.

* ``--dist loadfile``: Tests are grouped by their containing file. Groups are
  distributed to available workers as whole units. This guarantees that all
  tests in a file run in the same worker.

Making session-scoped fixtures execute only once
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``pytest-xdist`` is designed so that each worker process will perform its own collection and execute
a subset of all tests. This means that tests in different processes requesting a high-level
scoped fixture (for example ``session``) will execute the fixture code more than once, which
breaks expectations and might be undesired in certain situations.

While ``pytest-xdist`` does not have a builtin support for ensuring a session-scoped fixture is
executed exactly once, this can be achieved by using a lock file for inter-process communication.

The example below needs to execute the fixture ``session_data`` only once (because it is
resource intensive, or needs to execute only once to define configuration options, etc), so it makes
use of a `FileLock <https://pypi.org/project/filelock/>`_ to produce the fixture data only once
when the first process requests the fixture, while the other processes will then read
the data from a file.

Here is the code:

.. code-block:: python

    import json

    import pytest
    from filelock import FileLock


    @pytest.fixture(scope="session")
    def session_data(tmp_path_factory, worker_id):
        if worker_id == "master":
            # not executing in with multiple workers, just produce the data and let
            # pytest's fixture caching do its job
            return produce_expensive_data()

        # get the temp directory shared by all workers
        root_tmp_dir = tmp_path_factory.getbasetemp().parent

        fn = root_tmp_dir / "data.json"
        with FileLock(str(fn) + ".lock"):
            if fn.is_file():
                data = json.loads(fn.read_text())
            else:
                data = produce_expensive_data()
                fn.write_text(json.dumps(data))
        return data


The example above can also be use in cases a fixture needs to execute exactly once per test session, like
initializing a database service and populating initial tables.

This technique might not work for every case, but should be a starting point for many situations
where executing a high-scope fixture exactly once is important.

Running tests in a Python subprocess
------------------------------------

To instantiate a python3.9 subprocess and send tests to it, you may type::

    pytest -d --tx popen//python=python3.9

This will start a subprocess which is run with the ``python3.9``
Python interpreter, found in your system binary lookup path.

If you prefix the --tx option value like this::

    --tx 3*popen//python=python3.9

then three subprocesses would be created and tests
will be load-balanced across these three processes.

.. _boxed:

Running tests in a boxed subprocess
-----------------------------------

This functionality has been moved to the
`pytest-forked <https://github.com/pytest-dev/pytest-forked>`_ plugin, but the ``--boxed`` option
is still kept for backward compatibility.

.. _`remote machines`:

Sending tests to remote SSH accounts
------------------------------------

Suppose you have a package ``mypkg`` which contains some
tests that you can successfully run locally. And you
have a ssh-reachable machine ``myhost``.  Then
you can ad-hoc distribute your tests by typing::

    pytest -d --tx ssh=myhostpopen --rsyncdir mypkg mypkg

This will synchronize your :code:`mypkg` package directory
to a remote ssh account and then locally collect tests
and send them to remote places for execution.

You can specify multiple :code:`--rsyncdir` directories
to be sent to the remote side.

.. note::

  For pytest to collect and send tests correctly
  you not only need to make sure all code and tests
  directories are rsynced, but that any test (sub) directory
  also has an :code:`__init__.py` file because internally
  pytest references tests as a fully qualified python
  module path.  **You will otherwise get strange errors**
  during setup of the remote side.


You can specify multiple :code:`--rsyncignore` glob patterns
to be ignored when file are sent to the remote side.
There are also internal ignores: :code:`.*, *.pyc, *.pyo, *~`
Those you cannot override using rsyncignore command-line or
ini-file option(s).


Sending tests to remote Socket Servers
--------------------------------------

Download the single-module `socketserver.py`_ Python program
and run it like this::

    python socketserver.py

It will tell you that it starts listening on the default
port.  You can now on your home machine specify this
new socket host with something like this::

    pytest -d --tx socket=192.168.1.102:8888 --rsyncdir mypkg mypkg


.. _`atonce`:
.. _`Multi-Platform`:


Running tests on many platforms at once
---------------------------------------

The basic command to run tests on multiple platforms is::

    pytest --dist=each --tx=spec1 --tx=spec2

If you specify a windows host, an OSX host and a Linux
environment this command will send each tests to all
platforms - and report back failures from all platforms
at once. The specifications strings use the `xspec syntax`_.

.. _`xspec syntax`: http://codespeak.net/execnet/basics.html#xspec

.. _`socketserver.py`: https://raw.githubusercontent.com/pytest-dev/execnet/master/execnet/script/socketserver.py

.. _`execnet`: http://codespeak.net/execnet

Identifying the worker process during a test
--------------------------------------------

*New in version 1.15.*

If you need to determine the identity of a worker process in
a test or fixture, you may use the ``worker_id`` fixture to do so:

.. code-block:: python

    @pytest.fixture()
    def user_account(worker_id):
        """ use a different account in each xdist worker """
        return "account_%s" % worker_id

When ``xdist`` is disabled (running with ``-n0`` for example), then
``worker_id`` will return ``"master"``.

Worker processes also have the following environment variables
defined:

* ``PYTEST_XDIST_WORKER``: the name of the worker, e.g., ``"gw2"``.
* ``PYTEST_XDIST_WORKER_COUNT``: the total number of workers in this session,
  e.g., ``"4"`` when ``-n 4`` is given in the command-line.

The information about the worker_id in a test is stored in the ``TestReport`` as
well, under the ``worker_id`` attribute.

Since version 2.0, the following functions are also available in the ``xdist`` module:

.. code-block:: python

    def is_xdist_worker(request_or_session) -> bool:
        """Return `True` if this is an xdist worker, `False` otherwise

        :param request_or_session: the `pytest` `request` or `session` object
        """

    def is_xdist_master(request_or_session) -> bool:
        """Return `True` if this is the xdist controller, `False` otherwise

        Note: this method also returns `False` when distribution has not been
        activated at all.

        deprecated alias for is_xdist_controller

        :param request_or_session: the `pytest` `request` or `session` object
        """

     def is_xdist_controller(request_or_session) -> bool:
        """Return `True` if this is the xdist controller, `False` otherwise

        Note: this method also returns `False` when distribution has not been
        activated at all.

        :param request_or_session: the `pytest` `request` or `session` object
        """

    def get_xdist_worker_id(request_or_session) -> str:
        """Return the id of the current worker ('gw0', 'gw1', etc) or 'master'
        if running on the controller node.

        If not distributing tests (for example passing `-n0` or not passing `-n` at all) also return 'master'.

        :param request_or_session: the `pytest` `request` or `session` object
        """


Identifying workers from the system environment
-----------------------------------------------

*New in version UNRELEASED TBD FIXME*

If the `setproctitle`_ package is installed, ``pytest-xdist`` will use it to
update the process title (command line) on its workers to show their current
state.  The titles used are ``[pytest-xdist running] file.py/node::id`` and
``[pytest-xdist idle]``, visible in standard tools like ``ps`` and ``top`` on
Linux, Mac OS X and BSD systems.  For Windows, please follow `setproctitle`_'s
pointer regarding the Process Explorer tool.

This is intended purely as an UX enhancement, e.g. to track down issues with
long-running or CPU intensive tests.  Errors in changing the title are ignored
silently.  Please try not to rely on the title format or title changes in
external scripts.

.. _`setproctitle`: https://pypi.org/project/setproctitle/


Uniquely identifying the current test run
-----------------------------------------

*New in version 1.32.*

If you need to globally distinguish one test run from others in your
workers, you can use the ``testrun_uid`` fixture. For instance, let's say you
wanted to create a separate database for each test run:

.. code-block:: python

    import pytest
    from posix_ipc import Semaphore, O_CREAT

    @pytest.fixture(scope="session", autouse=True)
    def create_unique_database(testrun_uid):
        """ create a unique database for this particular test run """
        database_url = f"psql://myapp-{testrun_uid}"

        with Semaphore(f"/{testrun_uid}-lock", flags=O_CREAT, initial_value=1):
            if not database_exists(database_url):
                create_database(database_url)

    @pytest.fixture()
    def db(testrun_uid):
        """ retrieve unique database """
        database_url = f"psql://myapp-{testrun_uid}"
        return database_get_instance(database_url)


Additionally, during a test run, the following environment variable is defined:

* ``PYTEST_XDIST_TESTRUNUID``: the unique id of the test run.

Accessing ``sys.argv`` from the master node in workers
------------------------------------------------------

To access the ``sys.argv`` passed to the command-line of the master node, use
``request.config.workerinput["mainargv"]``.


Specifying test exec environments in an ini file
------------------------------------------------

You can use pytest's ini file configuration to avoid typing common options.
You can for example make running with three subprocesses your default like this:

.. code-block:: ini

    [pytest]
    addopts = -n3

You can also add default environments like this:

.. code-block:: ini

    [pytest]
    addopts = --tx ssh=myhost//python=python3.9 --tx ssh=myhost//python=python3.6

and then just type::

    pytest --dist=each

to run tests in each of the environments.


Specifying "rsync" dirs in an ini-file
--------------------------------------

In a ``tox.ini`` or ``setup.cfg`` file in your root project directory
you may specify directories to include or to exclude in synchronisation:

.. code-block:: ini

    [pytest]
    rsyncdirs = . mypkg helperpkg
    rsyncignore = .hg

These directory specifications are relative to the directory
where the configuration file was found.

.. _`pytest-xdist`: http://pypi.python.org/pypi/pytest-xdist
.. _`pytest-xdist repository`: https://github.com/pytest-dev/pytest-xdist
.. _`pytest`: http://pytest.org
