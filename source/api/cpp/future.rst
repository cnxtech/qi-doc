.. _api-future:
.. cpp:namespace:: qi
.. cpp:auto_template:: True
.. default-role:: cpp:guess

qi::Promise, qi::Future
***********************

The promise and future concepts allow one to provide a result asynchronously.
The implementor creates a promise for the result and returns the future
contained in the promise. The caller can then use the result of the future
later.

You can see a :ref:`complete usage example<future_usage>` at the end of this
document.

Summary
-------

.. cpp:brief::

Detailed Description
--------------------

qi::Future
==========

A `Future` provides a way to wait for and get the result of an asynchronous
operation. It is the receiving end of a `qi::Future` - `qi::Promise` pair.

*Future* is templated by the type of the underlying value. *void* is permitted.

The different states of a Future
................................

A future can be in multiple states represented by `FutureState`:

- FutureState_None: The Future is not tied to a Promise, and will never change
  state.
- FutureState_Running: The future is tied to a Promise, and the asynchronous
  operation has not finished yet.
- FutureState_Canceled: The operation was successfully canceled.
- FutureState_FinishedWithError: The operation finished with an error.
- FutureState_FinishedWithValue: The operation finished and its return value is
  available.

Getting the state and waiting for a Future
..........................................

There are multiple ways to handle a future. The first is to use the
`qi::Future<T>::value` method. In that case, the call will block until the
Future leaves the *running* state.  Then if a value if available, it will be
returned. Otherwise a `FutureException` will be raised:

.. code-block:: cpp

  qi::Future<int> someOperation();

  int callIt()
  {
    int i = someOperation().value(); // wait, then get the value or throw
    return i;
  }

If you do not wish to wait forever, or want to handle Future error without
catching an exception, you can use `Future<T>::wait` (timeout):
this function waits at most the specified time in milliseconds, and return
a `FutureState`. You can then safely call `Future<T>::value` or
`Future<T>::error`, if future is in state `FutureState_FinishedWithValue` or
`FutureState_FinishedWithError` respectively:

.. code-block:: cpp

  qi::Future<int> f = someOperation();
  switch(f.wait(1000))
  {
     case FutureState_Running:
       std::cerr << "Still not ready" << std::endl;
       break;
     case FutureState_Canceled:
       std::cerr << "Canceled" << std::endl;
       break;
     case FutureState_FinishedWithError:
       std::cerr << "Error: " << f.error() << std::endl;
       break;
     case FutureState_FinishedWithValue:
       std::cerr << "Value: " << f.value() << std::endl;
       break;
  }

Future notification
...................

Alternatively, you can get notified of Future completion asynchronously using
`Future<T>::connect`. This function accepts a callback function or
functor with signature ``void (qi::Future<T> f)``.

The Future guarantees you that your callback function will be called once and
only once, when or if the Future leaves the *Running* state (that is, enters
one of *Canceled*, *FinishedWithError* or *FinishedWithValue*):

.. code-block:: cpp

  void myCallback(qi::Future<int> f)
  {
    qi::FutureState s = f.wait(); // will return immediately, Future has finished.
    switch(s) {...}
  }

  // ...
  qi::Future<int> f = someOperation();
  f.connect(&myCallback);

The callback is always invoked asynchronously (in the promise thread or in any
thread, depending on the promise type) unless specified otherwise in the
connect.

.. _future-connect:

connect() accepts extra arguments after the callback: values or placeholders
that will be bound to the call(similarly to how ``boost::bind`` works). If
the first argument is a ``boost::weak_ptr``, or inherits from `qi::Trackable`,
then the callback will not be called if the weak_ptr cannot be locked, or
if the Trackable was destroyed:

.. code-block:: cpp

  class Foo
  {
  public:
    void onOpFinished(qi::Future<int> op, int opNumber);
  };

  void safe_async_op(boost::shared_ptr<Foo> foo, int opNumber)
  {
    qi::Future<int> future = someOperation();
    // This version will keep foo alive at least until the Future finished
    future.connect(&Foo::onOpFinished, f, _1, opNumber);
    // This version is safe in case foo is destroyed before the Future finishes.
    future.connect(&Foo::onOpFinished, boost::weak_ptr<Foo>(f), _1, opNumber);
  }

Future cancellation
...................

An async operation that returns a Future can support cancellation.  To check if
a future you have can be canceled, use `isCancelable`.

If `isCancelable` returns true, you can try to abort the operation by calling
`cancel`. Depending on the operation and on the timing of your call, your
cancel request might be ignored (for example, if it is received too late and a
value is already available). But you can expect the Future to hastily leave the
*Running* state one way or an other.

The property *cancellable* of a future is defined at the construction of the
promise. You rarely need to check if a future is cancellable, as the
specification of the function which returned it should tell whether it is
cancellable or not.

qi::Promise
===========

.. _api-promise:

A `qi::Promise` is an object that can create and satisfy a `qi::Future`.
Like *Future*, it has shared semantics (all copies of a Promise represent the
same object). The next example illustrates it's basic use case:

.. code-block:: cpp

  qi::Future<int> myFunctionReturningAFuture()
  {
    qi::Promise<int> promise;
    // start an asynchronous operation, holding the promise
    // note that starting threads like that is bad practice, use qi::async
    boost::thread(someAsynchronousOp, promise);
    return promise.future();
  }

  void someAsynchronousOp(qi::Promise<int> promise)
  {
    try {
      int result = performSomeTask();
      promise.setValue(result);
    }
    catch(const std::exception& e)
    {
      promise.setError(e.what());
    }
  }


In plain English:

- Create a `Promise` and return the ``future()`` obtained with
  `Promise::future`.
- Transmit the *Promise* to the asynchronously executing code.
- Notify of successful completion with `Promise::setValue` or
  `Promise::setError`.
- Only one of the two functions above must be called, and only once per
  `Promise`.

Supporting cancellation
.......................

If your asynchronous operation can be canceled, you must provide a callback
with signature ``void(qi::Promise<T>)`` to the `Promise` constructor. You can
provide `qi::PromiseNoop` if you don't have any specific action to do upon
cancellation.

This callback will then be called if a cancellation request is received by a
connected `Future`. This callback is expected to ensure that the connected
`Future` hastily leaves the *Running* state, by calling one of
`Promise::setValue`, `Promise::setError` and `Promise::setCanceled`.
However this call does not have to be made synchronously in the cancellation
callback.

If you used `qi::PromiseNoop`, you can rely on `Promise::isCancelRequested` to
check at a given point if the user requested a cancellation.

You can see :ref:`an example with cancellation support<future_usage_cancel>` at
the end of this document.

Controlling callback execution
..............................

When one of the three state-changing functions listed above is called on
a *Promise*, callbacks registered to the connected *Future* will be
invoked. You can control whether this invocation is made synchronously,
or asynchronously using a thread from an internal thread pool, by passing
one of *FutureCallbackType_Sync* and *FutureCallbackType_Async* to the
*Promise* constructor.

qi::FutureSync
==============

`qi::FutureSync<T>` is a lightweight wrapper on top of
`qi::Future<T>` that will wait on the `Future` in its destructor
if the `Future` was ignored by the user.

It is intended to be used as a way to provide a default apparent
synchronous-blocking behavior to a function, that can be changed into
an asynchronous behavior by handling the resulting `FutureSync`.

Returning a FutureSync
......................

You can simply change the returned type from `Future` to `FutureSync` in the
:ref:`basic example<api-promise>`. The returned `Future` will transparently
convert to a `FutureSync`.

Calling a function returning a FutureSync
.........................................

`FutureSync` follow this simple rule: The destructor will call
`Future::wait` from its destructor, unless:

- It is copied into another `Future` or `FutureSync`
- `FutureSync::async` or any of the Future function is called (`wait`,
  `connect`, ...)

`FutureSync` also has a cast operator that allows you to use the returned value
transparently.

.. code-block:: cpp

  qi::FutureSync<int> someFunction();

  void test()
  {
    someFunction(); // will wait
    qi::FutureSync<int> f = someFunction(); // will wait at end of scope
    someFunction().async();                 // will not wait
    qi::Future<int> f2 = someFunction();    // will not wait
    someFunction().value();                 // will wait, because of value()
    int val = someFunction();               // will wait, does the same as
                                            // value(), may throw on error
  }

.. _future_usage:

Implementing an asynchronous function
=====================================

Simple implementation
.....................

Here is an example of an asynchronous function implementation that supports
cancellation.

Let's implement this class and make ``calculate()`` asynchronous.

.. code-block:: cpp

  class Worker {
    public:
      int calculate();
  };

First, ``calculate`` must return a future and we must create a function to do
the actual work.

.. code-block:: cpp

  #include <qi/future.hpp>

  class Worker {
    public:
      qi::Future<int> calculate();

    private:
      void doWork(qi::Promise<int> promise);
  };

For the sake of this example, we'll use a simple function to simulate work:

.. code-block:: cpp

  void Worker::doWork(qi::Promise<int> promise)
  {
    int acc = 0;
    for (int i = 0; i < 100; ++i)
    {
      qi::os::msleep(10); // working...
      acc += 1;
    }
    promise.setValue(acc);
  }

And then, we must call this function asynchronously and return the
corresponding future:

.. code-block:: cpp

  qi::Future<int> Worker::calculate() {
    qi::Promise<int> promise;
    qi::async(boost::bind(&Worker::doWork, this, promise));
    return promise.future();
  }

Now, ``calculate`` is asynchronous! But this isn't useful at all, our code is
more complex and this could have been done just by calling `qi::async`. What we
can do now is to support cancellation so that one can call `cancel()` on the
returned future to abort the action.

.. _future_usage_cancel:

Cancellation support
....................

Promises are cancellable when they are given a cancellation callback at
construction. You usually don't need this callback so you can just pass the
no-operation callback.

.. code-block:: cpp

  qi::Future<int> Worker::calculate() {
    qi::Promise<int> promise(qi::PromiseNoop<int>);
    qi::async(boost::bind(&Worker::doWork, this, promise));
    return promise.future();
  }

``doWork()`` can now check if the future has been cancelled.

.. code-block:: cpp

  void Worker::doWork(qi::Promise<int> promise)
  {
    int acc = 0;
    for (int i = 0; i < 100; ++i)
    {
      if (promise.isCancelRequested())
      {
        std::cout << "cancel requested" << std::endl;
        promise.setCanceled();
        return;
      }
      qi::os::msleep(10); // working...
      acc += 1;
    }
    promise.setValue(acc);
  }

Reference
---------

.. cpp:autoenum:: qi::FutureState

.. cpp:autoclass:: qi::Future

.. cpp:autoclass:: qi::Promise

.. cpp:autoclass:: qi::FutureSync

.. cpp:autofunction:: qi::PromiseNoop
