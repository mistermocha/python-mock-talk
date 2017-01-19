Mocks for Testing
=================
:Author: Brian Weber, SRE at Twitter
:Contact: bweber@twitter.com
:Twitter: @mistermocha

Headers
=======

What will be covered:

- Overview of the mock library and its features
- Examples of how to use the mock library
- Review of its purposes
- When to use and when not to use

Why I wrote this talk
=====================

Timing! I was asked to give a talk after writing a bunch of mock test code.

- Testing makes better code
- Mocking can make better tests
- Learn from my experience

What is mocking?
=================

``unittest.mock`` is a library for testing in Python. It allows you to replace parts of your system under test with mock objects and make assertions about how they have been used.

:sub:`https://docs.python.org/3/library/unittest.mock.html`

Why should I mock?
==================

- Protect things from outside of your code from getting impacted by test runs
- Partition tests into distinct "units"
- Don't bother testing third-party libraries

Why should I mock?
==================

- Unit test safely
- Write better code
- Isolation

About the mock library
======================

Mock objects intend to replace another part of code so that it pretends to be that code

.. code-block:: python

   from unittest.mock import Mock
   from mycode import MyClass

   def test_myclass():
       my_object = MyClass()
       my_object.sub_method = Mock()
       my_object.visible_method()
       my_object.sub_method.assert_called_with("arg", this="that")

An example of using a mock
==========================

Replacements can be done with object patching

.. code-block:: python

  # yourcode.py
  def count_the_shells():
      p = Popen(['ps', '-a'], stdout=PIPE, stderr=PIPE)
      if p.wait():
          raise Exception('We had a fail')
      count = 0
      for proc in p.stdout.readlines():
          if "-bash" in proc:
              count += 1
      return count

.. code-block:: python

   # test.py
   @mock.patch('subprocess.Popen')
   def test_count_the_shells(mocked_popen):
       mocked_popen.return_value.stdout = open('testps.out')
       mocked_popen.return_value.wait.return_value = False
       assert count_the_shells() == 4

Let's dive deeper into this

An example of using a mock
==========================

.. code-block:: python

  # yourcode.py
  def count_the_shells():
      p = Popen(['ps', '-a'], stdout=PIPE, stderr=PIPE)
      if p.wait():
          raise Exception('We had a fail')
      count = 0
      for proc in p.stdout.readlines():
          if "-bash" in proc:
              count += 1
      return count

- ``Popen`` runs a command line execution and returns a subprocess object. In this case, ``p``
- ``p.wait()`` blocks until it gets back the shell's exit code and returns it as an integer.
- ``p.stdout`` is a filelike object that captures STDOUT

An example of using a mock
==========================

.. code-block:: python

   # test.py
   @mock.patch('subprocess.Popen')
   def test_count_the_shells(mocked_popen):
       mocked_popen.return_value.stdout = open('testps.out')
       mocked_popen.return_value.wait.return_value = False
       assert count_the_shells() == 4

- ``@mock.patch`` decorator replaces ``subprocess.Popen`` with a mock object. That gets passed in as
  the first argument in the test function. The test function receives it as ``mocked_popen``
- The ``Popen`` call returns a subprocess object. We're now amending the ``return_value`` of that
  object by applying behavior to ``stdout`` and ``wait``, which get used in the function
- Now when ``count_the_shells`` is executed, it calls the mock instead of ``Popen`` and gets back
  expected values.

About the mock library
======================

A default mock object will accept any undeclared function

.. code-block:: python

    >>> mock = Mock()
    >>> mock.this_is_never_assigned('hello')
    <Mock name='mock.this_is_never_assigned()' id='4422797328'>

This prevents accidental calls from blowing up your code, but, leaves room for a lot of error.

Safer instantiation by autospeccing - make the mock behave like more like the thing you're mocking


Spec and Autospec
==================

- ``spec`` tells the mock to closely behave like another. Mocks instantiated with ``spec=RealObject``
  will pass ``isinstance(the_mock, RealObject)``

.. code-block:: python

    >>> from collections import OrderedDict
    >>> mymock = Mock(spec=OrderedDict)
    >>> isinstance(mymock, OrderedDict)
    True
    >>> type(mymock)
    <class 'mock.Mock'>

Spec and Autospec
==================

- ``spec`` also affords protection, preventing calls to undeclared attributes. You can declare any
  additional attributes you wish.

.. code-block:: python

    >>> a = mymock.this_does_not_exist()
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/opt/twitter/lib/python2.7/site-packages/mock.py", line 658, in __getattr__
        raise AttributeError("Mock object has no attribute %r" % name)
    AttributeError: Mock object has no attribute 'this_does_not_exist'

    >>> mymock.this_does_not_exist = "this exists now"
    >>> print(mymock.this_does_not_exist)
    this exists now

Spec and Autospec
==================

- ``spec_set`` stricter spec, prevents amending missing attributes. Attempts to define undeclared
  attributes will fail on ``AttributeError``.

.. code-block:: python

    >>> mymock = Mock(spec_set=OrderedDict)
    >>> mymock.this_does_not_exist = "o no you didn't"
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/opt/twitter/lib/python2.7/site-packages/mock.py", line 761, in __setattr__
        raise AttributeError("Mock object has no attribute '%s'" % name)
    AttributeError: Mock object has no attribute 'this_does_not_exist'
    >>>

Spec and Autospec
==================

- ``create_autospec`` is even stricter. Mock functions defined to spec will enforce argument patterns
  for functions.

.. code-block:: python

  >>> def myfunc(foo, bar):
  ...     pass
  ...
  >>> mymock = create_autospec(myfunc)
  >>> mymock("one", "two")
  <MagicMock name='mock()' id='4493382480'>
  >>> mymock("just one")
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<string>", line 2, in myfunc
  TypeError: <lambda>() takes exactly 2 arguments (1 given)
  >>>

Spec and Autospec
==================

Appropriate use of spec can help you write cleaner code and catch typos

.. code-block:: python

   >>> mock = Mock(name='Thing', return_value=None)
   >>> mock(1, 2, 3)
   >>> mock.assret_called_once_with(4, 5, 6)
   # typo of "assert" passes because mock objects are forgiving

.. code-block:: python

   >>> from urllib import request
   >>> mock = Mock(spec=request.Request)
   >>> mock.assret_called_with
   Traceback (most recent call last):
   ...
   AttributeError: Mock object has no attribute 'assret_called_with'
   # since "assret_called_with" is a typo, it's not declared. Proper exception caught!

- ``name`` your mocks, which shows in the repr - useful for debugging!

Introspection
=============

Built-in functions for introspection

- ``called`` - boolean, true if ever called
- ``call_count`` - integer, number of times called
- ``call_args`` - mock.call() object with args from last call
- ``call_args_list`` - list of mock.call() with all args ever used
- ``method_calls`` - track calls to methods and attributes, and their descendents
- ``mock_calls`` - *all* calls to the mock object

Introspection
=============

Built-in assertion tests

- ``assert_called`` - if ever called
- ``assert_called_once`` - if called exactly once
- ``assert_called_with`` - specific args used in the last call
- ``assert_called_once_with`` - specific args are used exactly once
- ``assert_any_call`` - specific args used in any call ever
- ``assert_has_calls`` - like "any_call" but with multiple calls
- ``assert_not_called`` - has never been called

Modeling behavior
=================

Built-in functions that model behavior

- ``return_value`` coerces a function's returned value

.. code-block:: python

    >>> mymock.return_value = "Your name here"
    >>> mymock()
    'Your name here'

- ``side_effect`` runs arbitrary code

.. code-block:: python

   mocked = Mock(spec=MyClass)
   def my_side_effect(some_number):
       mocked.increment += 1
       return some_number + 4
   mocked.myfunc.side_effect = my_side_effect

   assert mocked.myfunc(4) == 8
   assert mocked.increment == 1
   assert mocked.myfunc(7) == 11
   assert mocked.increment == 2

Modeling behavior
=================

.. code-block:: python

    class DBWriter(object):
        counter = 0

        def __init__(self):
            self.db = DBLibrary()

        def commit_to_db(self, sql):
            self.counter += 1
            self.db.commit(sql)

        def save(self, string):
            sql = "INSERT INTO mytable SET mystring = '{}'".format(string)
            self.commit_to_db(sql)

        def drop(self, string):
            sql = "DELETE FROM mytable WHERE mystring = '{}'".format(string)
            self.commit_to_db(sql)

``save`` and ``drop`` Behavior is:
- Prepare the sql statement
- Write the statement to the database
- Increment the counter

How to exercise all code without writing to DB?

Modeling behavior
=================

Model 1: Patch commit_to_db and model behavior

.. code-block:: python

  @mock.patch('dbwriter.DBWriter.commit_to_db', autospec=True)
  def test_save(mock_commit):
      writer = DBWriter()

      def fake_commit(self, sql):
          writer.counter += 1

      mock_commit.side_effect = fake_commit

      writer.save("Hello World")
      mock_commit.assert_called_with(writer,
          "INSERT INTO mytable SET mystring = 'Hello World'")

- Gain introspection into how ``DBWriter`` internals are called
- Does not exercise any code in ``commit_to_db``

Modeling behavior
=================

Model 2: Patch db.commit so it doesn't actually run

.. code-block:: python

  @mock.patch('namespace.of.DBLibrary', autospec=True)
  def test_save(mock_dblib):
      writer = DBWriter()
      writer.save("Hello World")
      mock_dblib.return_value.commit.assert_called_with(writer,
          "INSERT INTO mytable SET mystring = 'Hello World'")

- Full exercise of ``DBWriter`` internal code
- No introspection into how ``commit_to_db`` is called

Another example
===============

Mock objects provide introspection

.. code-block:: python

    def get_example():
      r = requests.get('http://example.com/')
      if r.status_code == 200:
        return True
      else:
        return False

.. code-block:: python

   @mock.patch('requests.get', autospec=True)
   def test_get_example_passing(mocked_get):
       mocked_req_obj = mock.Mock()
       mocked_req_obj.status_code = 200
       mocked_get.return_value = mocked_req_obj
       assert get_example()

       assert mocked_get.called
       assert mocked_get.call_args = mock.call('http://example.com/')

Let's dive deeper into this

Another example
===============

.. code-block:: python

    def get_example():
      r = requests.get('http://example.com/')
      if r.status_code == 200:
        return True
      else:
        return False

- The ``requests`` library is used for URL calls
- ``requests.get`` returns a ``request`` object and assigns to ``r``
- ``r.status_code`` is a property with the HTTP status code of the response

Another example
===============

.. code-block:: python

   @mock.patch('requests.get', autospec=True)
   def test_get_example_passing(mocked_get):
       mocked_req_obj = mock.Mock()
       mocked_req_obj.status_code = 200
       mocked_get.return_value = mocked_req_obj
       assert get_example()

       mocked_get.assert_called()
       mocked_get.assert_called_with('http://example.com/')


- Just like earlier, ``@mock.patch`` specs & replaces ``requests.get`` with a mock that gets passed
  into ``mocked_get`` and give it the ``status_code`` property
- We then create ``mocked_req_obj`` and bolt it into the ``return_value`` of ``mocked_get``
- Now when we run ``get_example`` we exercise the code without calling the outside.

Another example
===============

.. code-block:: python

   @mock.patch('requests.get', autospec=True)
   def test_get_example_passing(mocked_get):
       mocked_req_obj = mock.Mock()
       mocked_req_obj.status_code = 400
       mocked_get.return_value = mocked_req_obj
       assert get_example()

       mocked_get.assert_called()
       mocked_get.assert_called_with('http://example.com/')


When to use a mock
==================

Replace a part of your code with a mock so it pretends like it's doing something

- Command-line execution
- State changes
- External API
- Really slow procedures
- Already well-tested code

Remember, this is for unit-testing, not acceptance/integration testing!

When to use a mock
==================

.. code-block:: python

  # yourcode.py
  def wipe_directory(path):
    p = Popen(['rm', '-rf', path], stdout=PIPE, stderr=PIPE)
    if p.wait():
      raise Exception('We had a fail')

.. code-block:: python

   # test.py
   @mock.patch('subprocess.Popen', spec_set=True)
   def test_count_the_shells(mocked_popen):
       mocked_popen.return_value.wait.return_value = False
       wipe_directory('fakepath')
       assert mocked_popen.assert_called_with(['rm', '-rf', path], stdout=PIPE, stderr=PIPE)

When to use a mock
==================

.. code-block:: python

    # yourcode.py 
    def get_example():
        r = requests.post('http://example.com/',
            data={'delete': 'everything', 'autocommit': 'true'})
        if r.status_code == 200:
            print('All things have been deleted')
            return True
        else:
            print('Got an error: {}'.format(r.headers))
            return False

.. code-block:: python

    # test.py
    @mock.patch('requests.post', autospec=True)
    def test_get_example_passing(mocked_get):
        mocked_req_obj = mock.Mock()
        mocked_req_obj.status_code = 200
        mocked_get.return_value = mocked_req_obj
        assert get_example()
        assert mocked_get.called

    @mock.patch('requests.get', autospec=True)
    def test_get_example_failing(mocked_get):
        mocked_get.return_value.status_code = 400
        assert not get_example()
        assert mocked_get.called

When not to use a mock
======================

- Never mock the filesystem
- Be judicious about mocking shared libraries (integration tests)

When not to use a mock
======================

The mock library does provide file-like objects for mocks, but the filesystem is very nuanced. It's
much better to just write temporary files. Use mocks to amend how to write those files out.

When not to use a mock
======================

General rules for when to use a mock:
- Look for where your code talks to things that are not your code. You most likely want to mock that.
- Look for where a unit your code requires isolation from the rest of your code for a good test. You
  most likely want to mock that
- Never mock the file system

Summary
=======

- Mock to isolate your code from the outside world (and vice versa)
- Mock to inspect inner behavior
- Mock speed up unit tests
- Above all else, write tests!

Thank you!
==========

:Author: Brian Weber, SRE at Twitter
:Contact: bweber@twitter.com
:Twitter: @mistermocha
