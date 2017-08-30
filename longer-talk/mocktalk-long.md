---
author: Brian Weber
creator: Twitter SRE
title: Unit Testing with Mock
date: PyBay Conference - August 12, 2017
output:
  slidy_presentation:
    highlight: kate
    css: mystyle.css
    
---

## In this talk...

- Motivations
- Overview of the `mock` library
- Examples to show when & how to use

## Why I gave this talk

I had to write a script that talked to...

- command-line
- git
- fetch/upload packages
- ldap/kerberos
- email
- **staging production deploys**

... How do I test this?

## This talk is kinda about unit testing

- **Unit**: Just this small part
- **Integration**: When all the parts talk to each other and included parts
- **Acceptance**: When the whole app talks to everything else

A unit test executes a small part (unit) of your code and confirms the desired outputs and effects.

### Tests execute your code

## This talk is kinda about unit testing

Unit tests should be:

- Fast
- Cheap
- Safe

... so that I can run them all the time

### Tests execute your code

## Pop quiz! {.bigger .center}

### How would you test this

## Code runs a shell command?

```python
def wipe_directory(path):
    p = Popen(
        # yes you're gonna delete everything
        ['rm', '-rf', path])
    if p.wait():
        raise Exception('We had a fail')
```

## Code calls a web API?

```python
def delete_everything():
    r = requests.post('http://example.com/',
        # yes, this deletes things too, right?
        data={'delete': 'everything', 'autocommit': 'true'})

    if r.status_code == 200:
        print('All things have been deleted')
        return True

    else:
        print('Got an error: {}'.format(r.headers))
        return False
```

## SQL through another library?

```python
class DBWriter(object):
    counter = 0

    def __init__(self):
        self.db = DBLibrary()

    def commit_to_db(self, sql):
        self.counter += 1
        self.db.commit(sql)

    # Write something to the database
    def save(self, string):
        sql = "INSERT INTO mytable SET mystring = '{}'".format(string)
        self.commit_to_db(sql)

    # And delete something too!
    def drop(self, string):
        sql = "DELETE FROM mytable WHERE mystring = '{}'".format(string)
        self.commit_to_db(sql)
```

## Mocks to the rescue!! {.bigger .center}

## Use mocks for safer unit tests

`unittest.mock` is a library for testing in Python. It allows you to replace parts of your system under test with mock objects and make assertions about how they have been used.

_Source: https://docs.python.org/3/library/unittest.mock.html_

Available in the standard library as of python 3.3, also available on https://pypi.python.org/pypi/mock for older versions

## Why should I use a mock?

- **Unit test safely:** stop the state-changing parts so you can actually run tests
- **Write better code:** a lovely side-effect of writing thorough tests
- **Isolation:** create walls between your code and "not-your-code" for safer tests

## How do I use a mock? {.center .bigger}

## Direct replacement

Just create instances and put them where you want them

```python
# Part of stdlib in python >= 3.3

from unittest.mock import Mock
from mycode import MyClass

def test_myclass():
    my_object = MyClass()
    my_object.sub_method = Mock()
    my_object.visible_method()
    my_object.sub_method.assert_called_with("foo")
```

## Subclassing to replace

```python
class TestMyClass(MyClass):
    sub_method = mock.Mock(...)

def test_myclass():
    my_object = TestMyClass()
    my_object.visible_method()
    my_object.sub_method.assert_called_with("foo")
```

## Using patch

Use `patch` to install a mock somewhere not directly accessible

```python
def count_the_shells():
    p = Popen(['ps', '-a'], stdout=PIPE)
    if p.wait():
        raise Exception('We had a fail')
    count = 0
    for proc in p.stdout.readlines():
        if "-bash" in proc:
            count += 1
    return count

# patch in the namespace
@mock.patch('subprocess.Popen')
def test_count_the_shells(mocked_popen):
    # compare against test output in a file
    mocked_popen.return_value.stdout = open('testps.out')
    mocked_popen.return_value.wait.return_value = False
    assert count_the_shells() == 4
```

## Using patch

Use `patch` to install a mock somewhere not directly accessible

```python
def count_the_shells():
    p = Popen(['ps', '-a'], stdout=PIPE, stderr=PIPE)
    if p.wait():
        raise Exception('We had a fail')
    count = 0
    for proc in p.stdout.readlines():
        if "-bash" in proc:
            count += 1
    return count

# patch in the namespace
def test_count_the_shells(mocked_popen):
    with mock.patch('subprocess.Popen') as mocked_popen:
        # compare against test output in a file
        mocked_popen.return_value.stdout = open('testps.out')
        mocked_popen.return_value.wait.return_value = False
        assert count_the_shells() == 4
```

## Which pattern of replacement is right?

- Which one matches your code patterns?
- Which one matches your team's patterns?
- Which one helps you write better, bug-free code?

**It's up to you!**

## Mocks are plastic

Calls to unassigned attributes just return a shapeless mock

```python
>>> mock = Mock()
>>> mock.this_is_never_assigned('hello')
<Mock name='mock.this_is_never_assigned()' id='4422797328'>
```
You must teach a mock how to behave.


_Definition of plastic (https://www.merriam-webster.com/dictionary/plastic)_

1. _formative, creative plastic forces in nature_
2. _capable of being molded or modeled; capable of adapting to varying conditions_

## Return value

Declaring a `return_value` can assign an expected value

```python
>>> mock.my_attr.return_value = "This space for rent"
>>> mock.my_attr()
'This space for rent'
>>>
```

Respond to that desired return code instead of actually calling an endpoint for one!

## Side effect

Use `side_effect` to do things that don't explicity have a static return value

```python
>>> def add_to(x):
...     mymock.counter += x
...
>>> mymock.add_to.side_effect = add_to
>>> mymock.add_to(5)
>>> mymock.add_to(3)
>>> mymock.counter
8
```

## MagicMock

Contains default implementations of magic (dunder) methods

```python
>>> mock = mock.Mock()
>>> int(mock)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: int() argument must be a string or a number, not 'Mock'

>>> magic = mock.MagicMock()
>>> int(magic)
1
```

## Spec and Autospec {.bigger .center}

Really pretend to be another thing!

## Using `spec`

Instantiating with `spec`:

- respects `type` checks.
- responds only to attributes defined in the `spec` object

```python
>>> from collections import OrderedDict
>>> mymock = Mock(spec=OrderedDict)
>>> isinstance(mymock, OrderedDict)
True
>>> type(mymock)
<class 'mock.Mock'>
```

The `spec` argument must be a class or an object/instance.

## Using `spec`

Only attributes defined in the spec are respected.

```python
>>> mymock = Mock(spec=OrderedDict)
>>> a = mymock.this_does_not_exist()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/opt/twitter/lib/python2.7/site-packages/mock.py", line 658, in __getattr__
    raise AttributeError("Mock object has no attribute %r" % name)
AttributeError: Mock object has no attribute 'this_does_not_exist'

>>> mymock.this_does_not_exist = "this exists now"
>>> print(mymock.this_does_not_exist)
this exists now
```

The mock object can still be altered though. You may want this.

## `spec_set` is safer

```python
>>> mymock = Mock(spec_set=OrderedDict)
>>> mymock.this_does_not_exist = "o no you didn't"
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/opt/twitter/lib/python2.7/site-packages/mock.py", line 761, in __setattr__
    raise AttributeError("Mock object has no attribute '%s'" % name)
AttributeError: Mock object has no attribute 'this_does_not_exist'
```

## `autospec` matches signatures

```python
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
```

Note that `autospec` returns a `MagicMock` by default as well! Use this when possible!

## Avoid typos in test code!

Using `spec` and its variants prevent typos in assertion calls

```python
>>> mock = Mock(name='Thing', return_value=None)
>>> mock(1, 2, 3)
>>> mock.assret_called_once_with(4, 5, 6)
```

```python
>>> from urllib import request
>>> mock = Mock(spec=request.Request)
>>> mock.assret_called_with
Traceback (most recent call last):
 ...
AttributeError: Mock object has no attribute 'assret_called_with'
```

## Don't over-model your code!

- Shared libraries could change - spec/autospec help with this
- It's easy to re-write large sections of code in mocks
- Remember the edges of your code - that's where mocks live

## Introspection

- `called` - boolean, true if ever called
- `call_count` - integer, number of times called
- `call_args` - mock.call() object with args from last call
- `call_args_list` - list of mock.call() with all args ever used
- `method_calls` - track calls to methods and attributes, and their descendents
- `mock_calls` - list of all calls to the mock object

## Introspection

```python
>>> mymock = Mock()
>>> mymock.foo()
<Mock name='mock.foo()' id='4465705552'>
>>> mymock.foo(1)
<Mock name='mock.foo()' id='4465705552'>
>>> mymock.foo("bar")
<Mock name='mock.foo()' id='4465705552'>
>>> mymock.foo.call_count
3
>>> mymock.foo.call_args
call('bar')
>>> mymock.foo.call_args_list
[call(), call(1), call('bar')]
>>> mymock.method_calls
[call.foo(), call.foo(1), call.foo('bar')]
>>>
```

## Built-in tests

- `assert_called_once` - if called exactly once
- `assert_called_with` - specific args used in the last call
- `assert_called_once_with` - specific args are used exactly once
- `assert_any_call` - specific args used in any call ever
- `assert_has_calls` - like `any_call` but with multiple calls
- `assert_not_called` - has never been called

## Built-in tests

```python
>>> mock = Mock()
>>> mock.bar("foo")
<Mock name='mock.bar()' id='4382462544'>
>>> mock.bar.assert_called_with("foo")


>>> mock.bar.assert_called_with("baz")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/opt/twitter/lib/python2.7/site-packages/mock.py", line 835, in assert_called_with
    raise AssertionError(msg)
AssertionError: Expected call: bar('baz')
Actual call: bar('foo')
```

## A few examples

## Calling an API

```python
def get_example():
    r = requests.get('http://example.com/')
    return r.status_code == 200
```

How do we...

- Block the outbound call
- Track behavior of the `requests` object
- Model the `status_code` attribute

## Calling an API

```python
def get_example():
    r = requests.get('http://example.com/')
    return r.status_code == 200
```

```python
def test_get_example_passing():
  mocked_req_obj = mock.Mock()
  mocked_req_obj.status_code = 200

  with mock.patch('requests.get', autospec=True) as mocked_get:
    mocked_get.return_value = mocked_req_obj
    assert(get_example())

    mocked_get.assert_called()
    mocked_get.assert_called_with('http://example.com/')
```

## Calling an API

```python
def get_example():
    r = requests.get('http://example.com/')
    return r.status_code == 200
```

```python
def test_get_example_failing():
  mocked_req_obj = mock.Mock()
  mocked_req_obj.status_code = 400

  with mock.patch('requests.get', autospec=True) as mocked_get:
    mocked_get.return_value = mocked_req_obj
    assert(not get_example())

    mocked_get.assert_called()
    mocked_get.assert_called_with('http://example.com/')
```

## Calling an API - improved!

```python
def get_example(session=None):
    if not session:
        session = requests.Session
    r = session.get('http://example.com/')
    return r.status_code == 200
```

```python
def test_get_example_failing():
  mocked_get = mock.Mock('requests.Session', autospec=True)
  mocked_get.return_value.status_code = 200

  assert(get_example())

  mocked_get.assert_called()
  mocked_get.assert_called_with('http://example.com/')
```



## A really simple class

```python
class AddStore(object):
    stored_number = 0

    def add_to(x):
        self.stored_number += x

    def show():
        return self.stored_number

    def __repr__():
        return "AddStore({}".format(self.stored_number)
```

What edges are there here?

## A really simple class

*NONE*

Nothing in that class talks to anything external, so no need to mock anything.

Only mock when you absolutely must.

## Database library

```python
class DBWriter(object):
    counter = 0

    def __init__(self):
        self.db = DBLibrary()

    def _commit_to_db(self, sql):
        self.counter += 1
        self.db.commit(sql)

    def save(self, string):
        sql = "INSERT INTO mytable SET mystring = '{}'".format(string)
        self.commit_to_db(sql)

    def drop(self, string):
        sql = "DELETE FROM mytable WHERE mystring = '{}'".format(string)
        self.commit_to_db(sql)
```

## Two approaches

## Patch locally defined function

```python
# mock_commit == DBWriter._commit_to_db
@mock.patch('dbwriter.DBWriter._commit_to_db', autospec=True)
def test_save(mock_commit):
    # reimplement
    def fake_commit(self, sql):
        self.counter += 1

    # assign side_effect to the function
    mock_commit.side_effect = fake_commit

    # run the code
    writer = DBWriter()
    writer.save("Hello World")

    # introspection
    mock_commit.assert_called_with(writer,
        "INSERT INTO mytable SET mystring = 'Hello World'")
    assertEquals(writer.counter, 1)
```

## Patch locally defined function

Pros:

- Insulates the DB
- Checks internal behavior

Cons:

- We're rewriting our code
- No inspection of the use of `DBWriter` instance

## Patch external library

```python
@mock.patch('dbwriter.DBLibrary', autospec=True)
def test_save(mock_dblib):
    writer = DBWriter()
    writer.save("Hello World")
    mock_dblib.return_value.commit.assert_called_with(writer,
        "INSERT INTO mytable SET mystring = 'Hello World'")
```

## Patch external library

Pros:

- Insulates the DB
- Less lines of code
- Introspection into external library use
- Full code exercise

Cons:

- No introspection of calls to `self`-based functions

## Which one should we use? {.bigger .center}

## BOTH!

Unit tests are cheap to run! When in doubt, write both tests, right?

## A new challenger appears!

Subclass and patch

```python
class TestDBWriter(DBWriter):
    def __init__(self):
        self.db = mock.Mock('dblibrary.DBLibrary', autospec=True)

def test_save(mock_dblib):
    writer = DBWriter()
    writer.save("Hello World")
    mock_dblib.return_value.commit.assert_called_with(writer,
        "INSERT INTO mytable SET mystring = 'Hello World'")
```

## Repeat instantiation

```python
from some.library import AnotherThing

class MyClass(object):
    def __init__(self, this, that):
        self.this = AnotherThing(this)
        self.that = AnotherThing(that)

    def do_this(self):
        self.this.do()

    def do_that(self):
        self.that.do()

    def do_more(self):
        got_it = self.this.get_it()
        that_too = self.that.do_it(got_it)
        return that_too
```

## Two approaches

## Patch `this` and `that` directly

```python
def test_my_class():
    my_obj = MyClass("fake this", "fake that")
    my_obj.this = Mock(spec_set='some.library.AnotherThing')
    my_obj.that = Mock(spec_set='some.library.AnotherThing')

    my_obj.do_this()
    my_obj.this.do.assert_called()
    my_obj.do_that()
    my_obj.that.do.assert_called()
```

## Patch `this` and `that` directly

Pros:

- Assurances that `this` and `that` are unique

Cons:

- `AnotherThing` is instantiated twice, and then replaced. Wasteful and _potentially unsafe_.

## Mock "factory"

```python
@patch('yourcode.AnotherThing', autospec=True)
def test_my_class(mock_thing):
    def fake_init(*args):
        return Mock(*args, spec_set='some.library.AnotherThing')
    mock_thing.side_effect = fake_init

    my_obj = MyClass("fake this", "fake that")
    my_obj.this.called_with("fake this")
    my_obj.that.called_with("fake that")
```

## Mock "Factory"

In this case, it's better because we can implement logic in the `side_effect` to ensure each
instance is unique.

## To Mock or Not To Mock? {.bigger .center}

## You should mock:

- Calls to a service's API endpoint
- Stateful databases
- Command-line execution
- Anytime your app talks to anything that is not your app
- Just at the edges and corners

## You should not mock:

- ... more than you need, use sparingly!
- The filesystem: too many edges
- Acceptance & integration tests: defeats the purpose

## How this helped me:

- I can make changes safely
- Others can contribute safely
- Continuous integration

## Everything I learned is all wrong

- Patching is wrong!
- Instantiating and replacing is wrong!
- Your code that you're testing is wrong!

## Everything I learned is all wrong

- Patching is ~~wrong!~~ _necessary when it's necessary_
- Instantiating and replacing is ~~wrong!~~ _just one technique_
- Your code that you're testing ~~is wrong!~~ _can be improved while you test_

## In closing

If you're writing tests, you're doing something right!

For more info:

- https://docs.python.org/3/library/unittest.mock.html
- https://dev.to/mistermocha

## Thank you!

- Brian Weber
- twitter/github: @mistermocha
- bweber@twitter.com

*We're hiring!*


