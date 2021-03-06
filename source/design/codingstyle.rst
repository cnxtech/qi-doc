.. _std-code-convention:

===================
 Coding Convention
===================

Backward Compatibility
======================

Qt API Design Principles
  http://wiki.qt-project.org/API_Design_Principles

Qt ABI thoughts
  http://labs.qt.nokia.com/2009/08/12/some-thoughts-on-binary-compatibility/

C++ Tips and Tricks for Mac OS X
  https://developer.apple.com/library/mac/#technotes/tn2185/_index.html

Potential Errors Passing CRT Objects Across DLL Boundaries
  http://msdn.microsoft.com/en-us/library/ms235460.aspx

Naming convention
=================

Headers go into <libname>/*

  - we want the same layout when headers are installed and are in the source tree
  - we want to prefix by the library name to avoid headers clash, remember that we want to install in /usr/include sometimes

Sources and private headers goes into <src>/*

  - That allows us to isolate public code that need a particular attention
  - It's easier to grep only in public headers

Private classes go into <src>/*

Private implementation headers should be named '<classname>_p.hpp'

  - this distinguishes public and private headers, otherwise we can have two files with the same name which is not pratical.
  - furthermore this denotates that the content should never be public, it's the private part of a class.

- Qt classes are not in a namespace
- Qt classes are prefixed with 'Qi'

Example
-------
For the public *foo* class and the private *oups* class in the *bar* library we have:

.. code-block:: console

  bar/foo.hpp

  src/foo.cpp
  src/foo_p.hpp
  src/oups.cpp
  src/oups.hpp
  src/oups_p.hpp (optional)

  qt/bar/qt/qifoo.h

  qt/src/qifoo.cpp
  qt/src/qifoo_p.h


This gives us the following objects:

STL:

.. code-block:: cpp

  #include <bar/foo.hpp>

  int main(void) {
     qi::Foo foo;
  }

QT:

.. code-block:: cpp

  #include <bar/qt/qifoo.h>

  int main(void) {
     QiFoo foo;
  }

Headers
=======

Public headers must not include public headers from other libraries. This
ensures binary compatibility.

Public headers must be enclosed within brackets <> when included.

On the other hand, private headers must be enclosed within double quotes "" when
included.

Header include statement
========================

* If a header is **not provided by the project**, then the include statement
  **must** be:

  .. code-block:: cpp

    #include <bar/foo.h>

* Else if a header is **provided by the project** and **gets installed**, so this is a
  *public header*, then the include statement **must** be:

  .. code-block:: cpp

    #include <bar/foo.h>

* Otherwise, this is a *private header* **provided by the project** and
  **only used in this project** (e.i. *never gets installed*), then the include
  statement **must** be:

  .. code-block:: cpp

    #include "bar/foo.h"

Export symbol
=============

All public functions and classes should be exported using <LIBNAME>_API macro. This macro should be unique to the library and never be used by others libraries.

.. code-block:: cpp

  #include <bar/api.hpp>

  class BAR_API Foo {
  };

For each library you will have to define <library>/api.hpp

.. code-block:: cpp

  #pragma once
  #ifndef _BAR_API_HPP_
  #define _BAR_API_HPP_

  #include <qi/macro.hpp>

  #define BAR_API QI_LIB_API(bar)

  #endif  // _BAR_API_HPP_


Please remember to export nested class

.. code-block:: cpp

  class BAR_API Foo
  {
  public:

    Foo();
    ~Foo();

    class BAR_API Bar // BAR_API is mandatory here
    {
    public:

      Bar();
      ~Bar();
    };

  };

Private Implementation
======================

- Use private implementation where applicable.
- Still reserve a pointer instead if you dont use it. (for future use, see
  example two).
- Classes should be named <classname>Private.
- A pointer '_p' should be added into the class.

When a class has a private implementation, the copy constructor *must* be either
implemented, either disabled - *ie.* defined in the private section of the class.


Example with Pimpl
------------------

bar/foo.hpp:

.. code-block:: cpp

  class FooPrivate;
  class Foo {
    FooPrivate *_p;
  };


Example without Pimpl
---------------------

.. code-block:: cpp

  class Foo {
  public:

  protected:
    //could be used to create a future pimpl if needed without breaking ABI
    void *_reserved;
    int   _mymember;
  };


Struct
======

You can expose struct but they should only contains POD. If a struct have a member which a class (or worst) a STL class, Windows wont be happy, and you will have to link
the exe and the dll with the same VC runtime, in the same configuration (release/debug). Prefer Pimpl in this case.

Exception
=========

http://stackoverflow.com/questions/4756944/c-dll-plugin-interface/4757105#4757105

Exceptions issues:

- may not be available on all platforms
- it's not really compatible with asynchronous design, where error reporting should be asynchronous too, but the error can catched at the caller place, and rethrow at the callee place. (qi::Future can help having both)
- exceptions increase the library size
- it's really hard to write exception-safe code. (RAII really help here)
- Exception catching of a user defined type in a binary other than the one which threw the exception requires a typeinfo lookup. (and rtti do not work well across dll boundary http://gcc.gnu.org/faq.html#dso)
- it break ABI: memory allocated in one place should be deallocated in the same place (remember that object do not have the same size in release/debug with MSVC), so if user catch a ref, this can crash.
- Avoiding leak is really hard (all function should handle exceptions, again RAII really help)

Example:

.. code-block:: c++

  A *a = new A();
  //this leak a A*
  functionthatthrow();
  delete a;

This could be fixed with a RAII smart pointer class:

.. code-block:: c++

  //this could be rewriten with smart pointer to avoid error
  boost::shared_ptr<A> a = boost::make_shared<A>();

  functionthatthrow();
  //a is cleanup here whatever happened.

even more vicious:

.. code-block:: c++

  //GenericObject that throw in operator= sometime
  class EvilObject;
  std::list<EvilObject> evilList;

  //simple function, that do not look evil, but can throw nevertheless,
  //but can you guess what?
  void functionthatdonotthrow(const EvilObject &eo) {
    evilList.push_back(eo);
  }

  void main() {
    EvilObject *eo = new EvilObject;
    //leak, but you can't guess that reading functionthatdonotthrow
    functionthatdonotthrow(*eo);
  }

Once you have exception you should use RAII everywhere to avoid leak.

So use exception with caution and prefer a simple hierarchy over a complicated one.
Do not hesitate to use std::runtime_error.
Never throw an exception not defined in your lib, except the one provided by the stdexcept header.

Iterators
=========

When naming an iterator, simply append "It" after the name of the container.

.. code-block:: c++

  std::vector<int> primeNumbers;
  std::vector<int>::iterator primeNumbersIt;


Enum
====

The name of the enumeration must be singular.

The enumeration values must be prefixed by the name of the enumeration followed by an underscore.

Precise each number and never reorder existing enumerations for ABI compatibility.

.. code-block:: c++

  class Message {
  public:

    enum Type {
      Type_Call = 0,
      Type_Error = 1,
      Type_Answer = 2,
      Type_Event = 3
    };

  };

Always prefer enumerations to booleans for readability.

.. code-block:: c++

  // bad: cannot understand just by reading the line
  Client ds("ip", true);
  // GOOD: easy to read, ok this is keepalive.
  Client ds("ip", Connection_KeepAlive);



Members
=======

- Private members names should be prefixed with underscores.

Arguments
=========

If the argument is IN-OUT then use pointer and avoid reference. The code that use the function is clearer to look at.

.. code-block:: c++

  int     a, b, result;
  bool    check;

  //the & show that the value can be modified
  check = computeButCanFail(a, b, &result);

  //bad... we dont know value will be modified
  check = computeButCanFail(a, b, result);

If the type is a POD (bool, char, short, int, float, double, etc...) use:

.. code-block:: c++

  void setValue(int i);

In all other case use const ref.

.. code-block:: c++

   void setValue(const MyClass &myclass);

Virtual
=======

All class with virtuals should have a virtual destructor to avoid leak.


Interface
=========

Always declare the destructor of an interface pure virtual.

(and provide an implementation to make it compile).

An interface should not be instanciable, so forcing the destrutor to be pure is good.

.. code-block:: c++

  class SocketInterface {
  public:
    //pure virtual destructor
    virtual ~SocketInterface() = 0;

    virtual void onReadyRead();
  };


Global
======

- Never define a global in a library that need code to run.
- always define global static

.. code-block:: c++

   static const std::string titi;       //bad because it call the constructor of std::string
   static std::string titi = "toto";    //bad because it call the constructor of std::string
   static const int i = somefunction(); //bad because it call somefunction
   std::string tutu;                    //very very bad because it's not static to the file and call the constructor of std::string

.. code-block:: c++

   static const std::string *titi = 0; // it's a pointer, so it does not call the std::string constructor
   static const int i = 0;
   static const float f = 2.;

** pointers
-----------

They should never be used to return data to users.
Implement fast copy constructor and operator=. Rely on swap semantic if needed.

Rational:
  Allocation should always be done in the same "space", a library should malloc and free his structure, user code too. Under windows structure do not have the same size between debug and release, this lead to release library not usable in debug build.

** pointer should only be used as input parameter, to pass an array of pointer.

.. code-block:: c++

  //BAD an object is created in the socket library, but should be released
  //in the client program
  Message *msg;
  socket.read(&msg);

.. code-block:: c++

  //Good, user provide a message to fill
  Message msg;
  socket.read(&msg);


Assert/Exit
===========

** Basically you should only use assert for something that according to you the developper when you write the assert CANNOT POSSIBLY BE FALSE.

- do not call assert when the error is not fatal.
- never call exit in a library.

Report error to the user of the library if possible instead. Users are then free to assert/exit as they want. A library should never crash a program delibarately.

assert is only active during debug, you may think that it is enough to use it, but Windows users use debug build (and some developer may too), and they do not want their program to crash because of a lib that do not handle errors correctly.

You should really be careful with assert, the goal is to catch error before a segfault, to point at the real error, instead of a random segfault. So use with caution.

A borderline example:

.. code-block:: c++

  inotify_event& evt = reinterpret_cast<inotify_event&>(buffer[i]);
  if (evt..mask & IN_CREATE) {...}
  else if (evt.mask & IN_STUFF) {...}
   <..>
  else assert(!"inotify event type unknown");

  //this one is wrong, if inotify api evolves some new message types may appear. This can happen and is not fatal to the code.
