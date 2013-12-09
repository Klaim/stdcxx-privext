Private Extension Methods
==========================================

* Document Number: NXXX=YY-ZZZZ
* Date: 2013-12-08
* Programming Language C++, Language Working Group
* Reply-to: Matthew Fioravante <fmatthew5876@gmail.com>

Introduction
=============================

This proposal adds a new mechanism for declaring and defining non-virtual 
private class methods and static private class methods outside of the
class definition.

Impact on the standard
=============================

This proposal is a core language extension. It does not require
any new keywords. It proposes one additional syntax by reusing an existing
keyword. The new feature does not break any legacy code. 
Some expressions which used to be a compilation error are now valid C++.
This feature can be thought of as a steping stone on the road to
modules in C++, but unlike modules will bring immediate benefit to
C++ developers.

Motivation: Problem Definition
================

Good class design follows the principle of encapsulation. We wish to
only expose the public aspects of our class in the interface 
and hide the implementation details.
The game of encapsulation is the game of hiding as many details
as possible, exposing only the minimum of details required for the users
of the interface. By minimizing the exposed information, we reduce
the complexity of our interface, making it easier to understand and use.
We also give ourselves much more freedom and flexibility with
implementation. Any class details not present in the interface can
be changed without affecting the users of the interface.

Analysis of Encapsulation in the C++ class model
--------------------------------

In C++, the class interface is manifest as the
class definition. This class definition is normally written in a header
file, in order to allow its use in multiple translation units. By
minimizing the details in the class definition, we achieve better
encapsultion and all of the benefits it brings.

The following table lists all of the aspects that make up a class. It
lists whether or not they are required to be in the class defintion.
We also mention which aspects are required by users of the class
and child classes which will inherit. Finally, the last column
describes when and where the aspect is required to be known
by the compiler in order to implement the language.

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

Let us examine this table and try to deconstruct which aspects should
be part of the class interface and which are implementation details.
First, we can remove all aspects which are used by either the user
or the inheritor as they are obviously part of the interface. This
leaves us with the following.

* Private data member defintions
* Private non-virtual method declarations
* Private static method declarations
* Friend declarations
* All member function definitions

The private data members are required by the compiler in order to
compute the size of the class object. The size is needed when
a new object is to be allocated on the stack or the heap. Therefore,
for practical reasons, we will also consider the private data members
as part of the class interface.

Friend declarations technically do not need to be known by the user,
the inheritor, or even the compiler at the class definition.
However, allowing friend declarations to be declared anywhere would 
make it very easy for users to abuse friends in order to break access
control. This proposal does not address any aspects of the controversal
friend feature and we will not speak of it again.

The leaves us with the private methods (non-virtual and static) and the
member function defintions, which are already not required to be present
in the definition. The users (user and inheritor) obviously do not need
to know of the signatures, much less the existance of private methods.
The compiler also does not need to know of the existance of private
methods until they are called. On all major platforms, changing
the non-virtual private methods (which are not called by inline functions)
does not affect the ABI of the class itself
\[[KDEABI](#KDEABI)\].

Thus, we conclude that non-virtual private methods declarations and static private method
declarations are not a part of the class interface. Therefore, following
the guidelines of encapsulation, they do not belong the class definition.

Practical Concerns
---------------------

High level discussions about encapsulation aside, there are some very real
practical problems that arise from requiring private method
declarations in the class definition.

* Whenever the class developer adds, removes, or changes the private method
    signatures, all users of the class *must* recompile. Large C++ applications
    already have prohibitively long compilation times. Waiting for compilation
    wastes a lot of programmer time and reduces productivity. Indeed
    many techniques have been developed such as PIMPL which sacrifice
    runtime efficiency in the name of reducing compile times. Whole
    books such as \[[Lakos01](#Lakos01)\] have been written
    with large sections dedicated to techniques for reducing compile times.
    Compilation time in C++ is a big problem that needs to be addressed.
    This proposal provides one way to greatly reducing compile times for many large projects.
    Of all of the issues given here, this one is the most dire.
* All of the symbols used in private method signatures must be exposed
    to the clients of the class. This matters for shared libraries
    where we wish to minimize symbol dependencies for load time and
    avoiding name clashes. While some platform dependent techniques
    such as visibility control can be used to mitigate the symbol
    pollution, this requires extra work from the programmer which he
    would not need to do at all if the private method signature was
    safetly encapsulated within the project's internal source files.
* File scope symbols (static keyword, anonymous namespaces) cannot be
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
    messy conditional compilation techniques such as `#ifdef`.


We propose that private non-virtual
methods and private static methods should be able to be declared outside
of the class definition. This change does not break encapsulation
because the methods are private, infact it improves encapsulation
for reasons previously stated.
We believe this feature could also be another step towards modules in C++.

As a side note, some programming languages provide a mixin feature
which allows programmers to add public methods and possibly data
members to classes. We are not proposing or even endorsing any form
of mixin here. As has been stated, private method signatures are
implementation details.

High Level Description
====================

The technical specification for this proposal is simple. Just allow the
programmer to declare and define additional class member functions
which are not present in the class definition. We call these additional
methods *private extension methods*.

Here is an example:

    //foo.hh
    class Foo {
      public:
        void pub1();
        void pub2();
      private:
        //private method provided in the class definition
        void _priv1();
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

    //A new private extension method, using the static keyword for file scope.
    static void Foo::_priv3() {
    }

    //Definition of Foo::pub1()
    void Foo::pub1() {
      _priv1();
      _priv3();
    }

Any class member function which has been declared but not found in the class definition will
be implicitly given private access control. For this reason, all
class member function declarations must require the class definition
to been previously seen.

The following would be a compilation error:

    void Foo::_priv(); //<-Error: class defintion not visible here!

    class Foo {
    };

The one issue that arises from the previous example is how to declare a static private
extension method. Within the class definition we prefix the
method with the static keyword. Prefixing an extension
method with the static keyword outside of class scope will
not define a static class method but instead define a private method
with static linkage.

In order to resolve this issue, we propose one more change. We
will allow the programmer to supply the static keyword after
the function declaration. Within the class definition, this will
have the same effect as putting the static keyword before the
declaration. Outside of the class definition, prefixing with
static will denote static linkage, while suffixing will
denote a static private extension method.

Some examples:

    class Foo {
      public:
        static void f1(); //<- static public method
        void f2() static; //<- static public method
      private:
        static void f3(); //<- static private method
        void f4() static; //<- static private method
    };

    void Foo::f5(); //<- private extension method
    static void Foo::f6(); //<- private extension method with static linkage
    void Foo::f7() static; //<- static private extension method
    static void Foo::f8() static; //<- static private extension method with static linkage

    static void bar(); //<-free function with static linkage
    void baz() static; //<-Compilation Error! Static suffix can only be used with class methods!

    void Foo::f1() static {
      /* f1()'s definition. Note that we can optionally include the static keyword here as a suffix.*/
    }

Technical Specification
=====================
    
In summary, this proposal makes the following changes to the standard:

* Allow user to declare class methods which are
    not present in the class definition. All of these methods will
    have private access control.
* In the class definition, allow the user to provide the static keyword
    after a method declaration, with the same meaning as if it appeared
    before the method declaration.
* If a static method is declared in the class definition, allow the user
    to optionally include the static keyword after the method signature
    in the method definition. This keyword will have no effect.
* If a non-static method is declared in the class definition, adding
    the static keyword after the method signature in the method defintion
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
    has seems somewhat natural as you are "explicitly" defining
    a new member function.
    This keyword however is meant to be used with
    constructors, do we really want another overloaded keyword like
    static in the language?
* private: The private keyword seems like a natural choice. However
    we don't like this keyword for one big reason, its confusing. 
    New users will undoubtably ask why you can write a private
    extension method but not a public or protected one. Someone in
    the future might even be tempted to write a proposal similar to
    this one allowing public and protected extension methods. Such a
    proposal would be a very bad idea as it would destroy encapsulation.
* [[attribute]]: This is almost like inventing a new keyword, except
    you're cheating by instead using a generalized attribute. We do
    not believe attributes should be used for back door keywords 
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

This would be a viable approach. Users who want to have file scoped
private extension methods can use anonymous namespaces.
We have opted against this approach for now as we believe it will lead
to confusion as the already emcumbered static keyword will now have
different meanings for member functions and free functions.

Acknowledgements
====================

Thank you to everyone on the std proposals forum for feedback and suggestions.

References
==================
* <a name="Lakos01"></a>Lakos, John. *Large-Scale C++ Software Design*, Addison-Wesley, July 1996, ISBN 0201633620.
* <a name="KDEABI"></a>[KDEABI] *Policies/Binary Compatibility Issues With C++ - KDE TechBase*, Available online at <http://techbase.kde.org/Policies/Binary_Compatibility_Issues_With_C++>.

