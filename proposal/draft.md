Private Extension Methods
==========================================

* Document Number: NXXX=YY-ZZZZ
* Date: 2013-12-08
* Programming Language C++, Language Evolution Working Group
* Reply-to: Matthew Fioravante <fmatthew5876@gmail.com>

Introduction
=============================

This proposal adds a new mechanism for declaring non-virtual 
private class methods and static private class methods outside of the
class definition.

Impact on the standard
=============================

This proposal is a core language extension. It does not require
any new keywords. It proposes one additional syntax by reusing an existing
keyword in a manner congruent with the keyword's original purpose.
The new feature does not break any legacy code. 
Some expressions which used to be a compilation error are now valid C++.

Motivation: Problem Definition
================

Good class design follows the principle of encapsulation.
The game of encapsulation is the game of hiding as many details
as possible, exposing only the minimum of details required for the users
to use the interface. By minimizing the exposed information, we reduce
the complexity of our interface, making it easier to understand and use.
We also give ourselves much more freedom and flexibility with
implementation. Any class details not present in the interface can
be changed without affecting the users of the interface.

Analysis of the quality of encapsulation enabled by the C++ class model
--------------------------------

We will perform an analysis of encapsulation provided by the C++
class mechanism.
In C++, a system interface can take the form of a class definition.
This class definition is normally written in a header
file, in order to allow its use in multiple translation units.
In order to maximize encapsulation, we must minimize the number
of class details in the interface to the bare minimum required by
the users of the class.

The following table lists all of the aspects that make up a class. It
lists whether or not they are required to be in the class definition.
The next 2 columns describe how each aspect is used by the programmer
using the class directly and the child class inheriting from it. The
last column describes when the compiler needs the aspect to be visible
in order to implement the language.

<table border="1">
	<tr><th>#</th><th>Aspect</th><th>Required in class definition?</th><th>User</th><th>Inheritor</th><th>Compiler</th></tr>
	</tr><tr>
		<td>1</td>
		<td>Public data member definitions</td>
		<td>Y</td><td>use</td><td>use</td><td>sizeof(), method definitions</td>
	</tr><tr>
		<td>2</td>
		<td>Protected data member definitions</td>
		<td>Y</td><td></td><td>use</td><td>sizeof(), method definitions</td>
	</tr><tr>
		<td>3</td>
		<td>Private data member definitions</td>
		<td>Y</td><td></td><td></td><td>sizeof(), method definitions</td>
	</tr><tr>
		<td>4</td>
		<td>Public virtual method declarations</td>
		<td>Y</td><td>call</td><td>override, call</td><td>vtable</td>
	</tr><tr>
		<td>5</td>
		<td>Protected virtual method declarations</td>
		<td>Y</td><td></td><td>override, call</td><td>vtable</td>
	</tr><tr>
		<td>6</td>
		<td>Private virtual method declarations</td>
		<td>Y</td><td></td><td>override</td><td>vtable</td>
	</tr><tr>
		<td>7</td>
		<td>Public non-virtual method declarations</td>
		<td>Y</td><td>call</td><td>call</td><td>only at call site</td>
	</tr><tr>
		<td>8</td>
		<td>Protected non-virtual method declarations</td>
		<td>Y</td><td></td><td>call</td><td>only at call site</td>
	</tr><tr>
		<td>9</td>
		<td>Private non-virtual method declarations</td>
		<td>Y</td><td></td><td></td><td>only at call site</td>
	</tr><tr>
		<td>10</td>
		<td>Public static method declarations</td>
		<td>Y</td><td>call</td><td>call</td><td>only at call site</td>
	</tr><tr>
		<td>11</td>
		<td>Protected static method declarations</td>
		<td>Y</td><td></td><td>call</td><td>only at call site</td>
	</tr><tr>
		<td>12</td>
		<td>Private static method declarations</td>
		<td>Y</td><td></td><td></td><td>only at call site</td>
	</tr><tr>
		<td>13</td>
		<td>friend declarations</td>
		<td>Y</td><td></td><td></td><td>only at friend definition</td>
	</tr><tr>
		<td>14</td>
		<td>All member function definitions</td>
		<td>N</td><td></td><td></td><td>only for inlining</td>
	</tr>
</table>

Let us examine this table and try to deconstruct which aspects
required in the class definition are actually
part of the class interface and which are implementation details.
First, the public and protected aspects must be a part
of the interface because they are directly exposed to class user and
the child class who inherits.
Private virtual methods are also part of the interface to the child class
because the child class may choose to override them.

Now we are left with the following:

* Private data member definitions
* Friend declarations
* Private non-virtual method declarations
* Private static method declarations

Private data members are not technically part of the interface as
they cannot be accessed by users or child classes.
Here we run into practical considerations. The compiler needs to know
the size of the entire object in order to create instances of it.
The size cannot be computed without seeing all of the data members,
regardless of access control. Developers who wish to increase encapsulation
by hiding the private data members can use the PIMPL idiom
at the expense of run time efficiency.

Friend declarations also are not technically part of the interface.
However, allowing friend declarations to be declared anywhere would 
make it very easy for users to abuse friends in order to break access
control. This proposal does not address any aspects of the
friend feature and we will not speak of it again.

The leaves us with the non-virtual private methods and static private methods.
The direct users and child classes cannot call private methods so they
do not need to see their signatures, much much less know of their existence.
The compiler also does not need to know of the existence of private
methods until they are called. On all major platforms, changing
the non-virtual private methods (which are not called by inline functions)
does not affect the ABI of the class itself
\[[KDEABI](#KDEABI)\].

We conclude that non-virtual private method and static private method
declarations are not a part of the class interface and thus should not
be required in the class definition.

Practical Concerns
---------------------

High level discussions about encapsulation aside, there are some very real
practical problems that arise from requiring private method
declarations in the class definition.

* Whenever the class developer adds, removes, or changes the private method
    signatures, all users of the class *must* recompile. Large C++ applications
    already have prohibitively long compilation times. Waiting for compilation
    wastes a lot of programmer time and reduces productivity.
    Whole books such as \[[Lakos01](#Lakos01)\] have been written
    with large sections dedicated to techniques for reducing compile times.
    Compilation time in C++ is a big problem that needs to be addressed.
    This proposal is one major step in the reduction of programmer
    time wasted waiting for compilation during development.
    Of all of the issues given here, this one is the most dire.
* All of the symbols used in private method signatures must be exposed
    to the clients of the class. This matters for shared libraries
    where we wish to minimize symbol dependencies to optimize library load time and
    avoid symbol name clashes. While some platform dependent techniques
    such as visibility control can be used to mitigate the symbol
    pollution, this requires extra work from the programmer which he
    would not need to do at all if the private method signature was
    safely encapsulated within the project's internal source files.
* File scope symbols (i.e static and anonymous namespaces) cannot be
    used in private method signatures, even if those private methods
    are only called within one translation unit. File scope symbols
    are another good technique for reducing the symbols exported by
    a binary and can also enable compiler optimizations.
* Some classes may have multiple implementations which can be swapped
    out at compile time. One example is a cross platform file IO
    library such as iostream. It is not unreasonable to expect one
    platform to require different private method signatures than
    another. Because these signatures are required to be in the
    definition, we are then forced to also include all of the
    messy conditional compilation details into the header files.


We propose that private non-virtual
methods and private static methods should be able to be declared outside
of the class definition. This change does not break encapsulation
because the methods are private, in fact it improves encapsulation
for reasons previously stated.
We believe this feature could also be another step towards modules in C++.

As a side note, some programming languages provide a mixin feature
which allows programmers to reopen a class and extend its interface
arbitrarily.
We are not proposing or even endorsing any form
of mixin here. Adding, removing, or changing private methods does
not change the interface of the class.

High Level Description
====================

The basic idea behind this proposal is simple. Just allow the
programmer to declare additional class non-virtual private methods
and static private methods
which are not present in the class definition. We call these additional
methods *private extension methods*.

Here is an example:

    //foo.hh
    class Foo {
      public:
        Foo();
        void pub1();
        void pub2();
      private:
        //private method declared within the class definition
        void _priv1();
        
        int _i;
        int _j;
    };

    //Forward declaration of a private extension method
    void Foo::_priv2();

    //Inlined public method calls the private extension method
    inline void Foo::pub2() { _priv2(); }

<!-- -->

    //foo.cc
    //Definition of _priv1()
    void Foo::_priv1() {
      /* function body */
    }

    //A new file scoped private extension method.
    static void Foo::_priv3() {
      _i = 3;
    }

    //Another file scoped private extension method.
    namespace {
      static void Foo::_priv4() {
        _priv3();
        _j = _i++;
      }
    }

    //Definition of Foo::pub1()
    void Foo::pub1() {
      _priv1();
      _priv4();
    }

    //A private extension method constructor
    Foo::Foo(int i, int j) : _i(i), _j(j) {}

    //Public constructor delegates to the private extension constructor
    Foo::Foo() : Foo(0, 0) {}

Any class method which has been declared but not found in the class definition will
be implicitly given private access control. For this reason, all
class method declarations will require the class definition
to been previously seen.

The following would be a compilation error:

    void Foo::_priv(); //<-Error: class definition not visible here!

    class Foo {
    };

The one issue that arises from the previous example is how to declare a static private
extension method. Within the class definition we prefix the
method with the static keyword. Prefixing an extension
method with the static keyword outside of class scope will
not define a static class method but instead define a private extension method
with static linkage.

In order to resolve this issue, we propose one more change. We
will allow the programmer to supply the static keyword after
the function declaration. Within the class definition, this will
have the same effect as putting the static keyword before the
declaration. Outside of the class definition, prefixing with
static will denote file scope, while suffixing will
denote a static private extension method.

Some examples:

    class Foo {
      public:
        static void f1(); //<- static public method
        void f2() static; //<- static public method
      private:
        static void f3(); //<- static private method
        void f4() static; //<- static private method

        void f5(); //<- private method
        void f6(); //<- private method
    };

    void Foo::f7(); //<- private extension method
    static void Foo::f8(); //<- file scope private extension method
    void Foo::f9() static; //<- static private extension method
    static void Foo::f10() static; //<- file scope static private extension method

    static void bar(); //<-file scope free function
    void baz() static; //<-Compilation Error! Static suffix can only be used with class methods!

    void Foo::f1() static {
      /* f1()'s definition. Note that we can optionally include the static keyword here as a suffix.*/
    }
    static void Foo::f5() { //<-Compilation Error! Class methods declared in the class definition cannot have file scope!
    }
    void Foo::f6() static { //<-Compilation Error! f6() was declared to be a non-static class method!
    }

Technical Specification
=====================
    
In summary, this proposal makes the following changes to the standard:

* Allow the programmer to declare class methods which are
    not present in the class definition. All of these methods will
    have private access control.
* In the class definition, allow the programmer to provide the static keyword
    after a method declaration, with the same meaning as if it appeared
    before the method declaration.
* If a static method is declared in the class definition, allow the programmer
    to optionally include the static keyword after the method signature
    in the method definition. This keyword will have no effect.
* If a non-static method is declared in the class definition, adding
    the static keyword before or after the method signature in the method definition
    will result in a compiler error.
* If a class method is declared and is not present in the class definition, 
    the user may add the static keyword before the method signature to give
    the method file scope.
    All declarations and definitions must be the same with regards to
    the static keyword prefix, otherwise a compiler error results.
* If a class method is declared and is not present in the class definition, 
    the user may add the static keyword after the method signature to declare
    a new private static class method.
    All declarations and definitions must be the same with regards to
    the static keyword suffix, otherwise a compiler error results.

Alternate Syntax
===================

There are many possible ways to encode this new feature in syntax. One
downside of the current approach is that previously if the user wrote
a typo when writing a member function name in the definition they
would get a helpful compilation error. With this proposal, they will
simply define a private extension method, and not see any error until
later during the linking stage. We believe the benefits of our
proposal outweigh this minor cost. Some other schemes for syntax
are included below for consideration.

Require a keyword
------------------------------

Another approach is to require a keyword in front of each private
extension method declaration and definition. This addresses the above
mentioned typo issue but has other drawbacks as will be described shortly.
It would look like this:

    class Foo {
    };
    
    keyword void Foo::_priv1(); //<-private extension method
    void Foo::_priv2(); //Compilation Error! _priv2() not found in class definition. Private extension methods require keyword 'keyword'!

The question of course is which keyword to use. Here are some ideas:

* explicit: The explicit keyword could be reused here and it even
    seems somewhat natural as you are "explicitly" defining
    a new member function. All of the current uses of this keyword
    are related to type conversions and it would be better to keep
    it this way for consistency and ease of understanding.
    Do we really want another overloaded keyword like
    static in the language?
* private: The private keyword seems like a natural choice but it
    is a bad choice because it is confusing.
    New users will undoubtedly ask why you can write a private
    extension method but not a public or protected one. Someone in
    the future might even be tempted to write a proposal similar to
    this one allowing public and protected extension methods. 
* [[attribute]]: This is almost like inventing a new keyword, except
    you're cheating by instead using a generalized attribute.
    Attributes should be used for back door keywords 
    to implement new language features.
* Invent a new keyword: This probably doesn't even need to be said.
    New keywords are something that should only be introduced if
    there is a really good reason. This is a small proposal that
    does not merit a new keyword.

Static extension methods
-------------------------

In order to the resolve the ambiguity of the static keyword we proposed
to allow specifying static after the function signature. Another
possibility is to just change the meaning of the static prefix 
for member functions to denote a static method.

For example:

    class Foo {
    };
    
    static void Foo::priv1(); <-Static private extension method

This would be a viable approach and has even less impact on the
standard then the current proposal. Users who want to have file scoped
private extension methods can still use anonymous namespaces.
We have opted against this approach for now as we believe it will lead
to confusion as the already encumbered static keyword will now have
different meanings for member functions and free functions.

Acknowledgements
====================

Thank you to everyone on the std proposals forum for feedback and suggestions.

References
==================
* <a name="Lakos01"></a>[Lakos01] Lakos, John. *Large-Scale C++ Software Design*, Addison-Wesley, July 1996, ISBN 0201633620.
* <a name="KDEABI"></a>[KDEABI] *Policies/Binary Compatibility Issues With C++ - KDE TechBase*, Available online at <http://techbase.kde.org/Policies/Binary_Compatibility_Issues_With_C++>.

