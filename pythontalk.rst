Mocks for Testing
=================
Brian Weber, SRE, Twitter

How I learned about mocking
===========================

... or how I really learned how to test safely


What is mocking?
=================

``unittest.mock is a library for testing in Python. It allows you to replace parts of your system under test with mock objects and make assertions about how they have been used.``

*source: https://docs.python.org/3/library/unittest.mock.html*

This is a code block
====================

.. code-block:: python

   from this import that

   def foo(bar):
     log.debug("I am foo with a bar of {}".format(bar))

Another slide
=============

- One thing
- Another thing
- Oh yeah
- BTW...
