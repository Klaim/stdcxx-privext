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
The compiler also does not need to aware of the private
methods signatures until they are called. On all major platforms, changing
the non-virtual private methods (which are not called by inline functions)
does not affect the ABI of the class itself
\[[KDEABI](#KDEABI)\].

We conclude that non-virtual private method and private static member function
declarations are not a part of the class interface and thus should not
be required in the class definition.

Practical Concerns
---------------------

High level discussions about encapsulation aside, there are some very real
practical problems that arise from requiring private method
declarations in the class definition.

* Whenever the class developer adds, removes, or modifies a private method
    signature, all users of the class *must* recompile. Large C++ applications
    already have prohibitively long compilation times. Waiting for compilation
    wastes a lot of programmer time and reduces productivity.
    Whole books such as \[[Lakos01](#Lakos01)\] have been written
    with large sections dedicated to techniques for reducing compile times.
    Long compilation time in C++ is a big problem that needs to be addressed.
    This proposal is one major step in the reduction of programmer
    time wasted waiting for compilation during development.
    Of all of the issues given here, this one is the most dire.
* Unessessary dependencies are introduced into the class header file. 
    All of the symbols used in private method signatures must be exposed
    to the clients of the class. This matters for shared libraries
    where we wish to minimize symbol dependencies to optimize library load time and
    avoid symbol name clashes. While some platform dependent techniques
    such as visibility control can be used to mitigate the symbol
    pollution, this requires extra work from the programmer which he
    would not need to do at all if the private method signature along
    with all of it's symbol dependencies were
    safely encapsulated within the project's internal source files.
* Symbols with internal linkage (i.e keyword `static` and anonymous namespaces) cannot be
    used in private method signatures, even if those private methods
    are only called within one translation unit. This artificially 
    restricts the programmer from passing and returning intrnally linked
    symbols to and from their private methods. Internal linkage is
    another good technique for reducing the set of symbols exported by
    a binary and can also enable compiler optimizations.
* Some classes may have multiple implementations which can be swapped
    out at compile time. One example is a cross platform file IO
    library such as iostream. It is not unreasonable to expect 
    each platform to require a different set of private method
    signatures to implement its behavior.
    Because these signatures are required to be in the
    definition, we are then forced to also include all of the
    messy conditional compilation details into the header files.

Problem Solution
------------

We propose that private non-virtual
methods and private static methods should be able to be declared outside
of the class definition. 
Not only does this change not break encapsulation, it actually improves
encapsulation because unessessary implementation details are being removed from the interface.
Unlike PIMPL or other related encapsulation techniques, this new feature has
no run time overhead.
We believe the implementation of this feature could also be another step towards modules in C++.

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
class methods *private extension methods (PEM)*  and *private static extension member functions (PSEMF)*.

Declaring private extension methods
----------------------------

In order to declare a PEM, we simply declare
a new class member function outside of the class definition 
and prefix it with the `private` keyword:

foo.hh

    class Foo {
      public:
        int pubf(); //<-public member function
      private:
        int _i;

        int _privf(); //<-private member function
    };

foo.cc

    //Private extension method _priv_extf()
    private void Foo::_priv_extf() {
      _i++;
    }

    //Definition of pubf(). Calls our extension method.
    int Foo::pubf() {
      _priv_extf();
      return _privf();
    }

    //Definition of _privf()
    int Foo::_privf() {
       return _i * 2;
    }

We can also declare PEMs in header files.
This allows us to separate the class implementation into multiple
source files and share PEMs between them.
The next example shows the possibilities.

foo.hh (public header file, exposed to library users)

    class Foo {
      public:
        int pub1();

        int pubx();
        int puby();
      private:
        int _x;
        int _y;
        
        int _priv1();
    };

    //Declare a PEM in the class header file.
    //In this case, the effect is the same as declaring it within the class
    //definition. Using an extension method here instead of declaring in the
    //class definition is merely a stylistic choice.
    Foo::_priv2();

    //pub2() is inlined and calls _priv1() and _priv2(), so we need to see
    //both of them at this point. This can be accomplished by including them
    //within the class definition or declaring an extension method as shown
    //by the examples of _priv1() and _priv2().
    inline int pub2() { return _priv1() + _priv2(); }

foo\_impl.hh (private header file)

    //Here our private header file defines more extension methods
    private int Foo::_priv3();
    inline private int Foo::_priv4() { return _priv3() + 42; }

foo\_x.cc (private implementation file)

    //Another PEM defined only in this translation unit.
    //We should probably give this method internal linkage (see below).
    private int Foo::_transform_x() {
      return _x * x + 50;
    }

    //implementation of pubx()
    int Foo::pubx() {
      return _transform_x() * _priv4();
    }

foo\_y.cc (private implementation file)

    //implementation of puby()
    int Foo::puby() {
      return _y + _priv4();
    }

Private extension constructors
--------------

We can also declare additional private constructors:

    class Foo {
      public:
        Foo();
    };

    //PEM constructor
    private Foo::Foo(int i, int j) : _i(i), _j(j) {}

    //Public constructor delegates to the private extension constructor
    Foo::Foo() : Foo(0, 0) {}

Note that we cannot declare private extension default constructors, copy constructors, copy assignment,
move constructor, move assignment, or destructors. All of the following are errors:

    class Foo {};

    private Foo::Foo(); //Error: Cannot declare a PEM default constructor!
    private Foo::Foo(const Foo&); //Error: Cannot declare a PEM copy constructor!
    private Foo& Foo::operator=(const Foo&); //Error: Cannot declare a PEM copy assignment operator!
    private Foo::Foo(Foo&&); //Error: Cannot declare a PEM move constructor!
    private Foo& Foo::operator=(Foo&&); //Error: Cannot declare a PEM move assignment operator!
    private Foo::~Foo(); //Error: Cannot declare a PEM destructor!

Class Definition visibility and the private keyword
----------------------

Any class method which has been declared but not found in the class definition 
requires the `private` keyword, otherwise a compiler error will ensue.
Likewise any method definition which has been previously
declared in the class definition must not be prefixed by the `private` keyword.
For this reason, all class method declarations will require the class definition
to been previously seen.

The following are compilation errors:

    private void Foo::_p1(); //<-Error: class definition not visible here!

    class Foo {
      void _p2();
    };

    void Foo::_p3(); //<-Error: PEMs must use the private keyword!
    private void Foo::_p2() {} //<-Error: member function definitions cannot use the private keyword!

Static Member Functions
-------------------------------

As was discussed earlier, static private member functions are also 
implementation details and do not belong in the class definition.
We can define static private extension member functions by adding the `static`
keyword *after* the `private` keyword:

    class Foo {
      public:
        static int sf();
    };

    private void Foo::_f1(); //<-PEM

    private static int Foo::_f2() { return 42; } //<-PSEMF
    int Foo::sf() { return _f2(); }

The `static` keyword must appear *after* `private`. The reason will be shown in the next section.

Internal Linkage
--------------------------

Most private extension methods are likely to be used only within one translation unit (TU). 
Symbols used within only one TU can be given internal linkage. This reduces the number of symbols
in the entire application and allows more aggressive optimizations. For example, the compiler can
inline all calls and completely remove the function body from the compiled binary.

### The Static keyword

We would like to enable internal linkage for PEMs. This is also
enabled by the `static` keyword, in the same manner as other symbols. To give
internal linkage to a PEM, we include the `static` keyword
*before* the `private` keyword.

    class Foo {
    };

    private void Foo::_f1(); //<-PEM
    private static void Foo::_f2(); //<-PEM with internal linkage
    static private void Foo::_f3(); //<-PSEMF
    static private static void Foo::_f4(); //<-PSEMF with internal linkage

### Anonymous namespaces
    
The other method used in C++ to enable internal linkage is the use of anonymous name spaces.
The anonymous namespace presents something of a conundrum for PEMs.
Class methods always exist in the same namespace as the class itself. Declaring a PEM
within an anonymous namespace would seem to be defining a method of a class within a different namespace
as that of the class itself. We are opting to allow this feature, but are willing to forego it should
the committee decide against it. We already have the static keyword for internal linkage so it would
not be a huge loss if anonymous namespaces were not supported.

    class Foo {
    };

    namespace {
      private int Foo::_f1()
    };

We will require that the anonymous namespace is a member of the namespace of the class.

    namespace A {
      class X {};    
    }

    namespace A {
      namespace {
        private X::_a(); //<-Ok, PEM for A::X.
      }
      namespace B {
        namespace {
          private X::_b(); //Error: Tried to declare a PEM for non-existent class A::B::X.
        }
      }
    }

    namespace C {
      private X::_c(); //Error: Tried to declare a PEM for non-existent class C::X
    }

    private X::_d(); //Error: Tried to declare a PEM for non-existent class ::X

An ambiguity can result if the same name is used in the parent namespace and anonymous namespace.
This can be resolved using a full qualification.

    namespace A {
      class X {};
      namespace {
        class X {};

        private X::_a(); //Declares a PEM for X within the anonymous namespace, with internal linkage.

        private A::X::_a(); //Declares a PEM for A::X with internal linkage.
      }
    }

Class Templates and PEMs
--------------------

Class templates are supported in the natural way and do not require any special rules or caveats.

    template <typename T>
    class X {};

    template <typename T>
    private void X<T>::_f1(); //<-PEM for class template X

    template <>
    private void X<int>::_f2(); //<-Specialization of PEM for X<int>

Explicit instantiation of a class template will instantiate all of it's member functions.
This will recursively instantiate only the PEMs (and additional PEMs called by these,
in a recursive manner) called by those member functions.
This procedure follows the same rules as explicit instantiation
of a class template which calls free function templates.

    template <typename T>
    class X {
      public:
        void f(); 
    };

    template <typename T>
    private void X<T>::_f1() { //<-PEM for class template
      /* stuff */
    }

    template <typename T>
    private void X<T>::_f2() { //<-PEM for class template
      /* stuff */
    }

    //templated f() calls both _f1() and _f2()
    template <typename T>
    private void X<T>::f() {
      _f1();
      _f2();
    }

    //Some specializations of f()
    template <> void X<char>::f() { _f2(); }
    template <> void X<int>::f() { _f1(); }
    template <> void X<float>::f() { }
    template <> void X<double>::f() { _f1(); }
 
    //A specialization of _f1()
    template <>
    private void X<double>::_f1() { _f2(); }

    //Explicit instantiations
    template class X<char>;  /* Will instantiate the following:
                              * X<char>::f();
                              * X<char>::_f2();
                              */
    template class X<short>; /* Will instantiate the following:
                              * X<short>::f();
                              * X<short>::_f1();
                              * X<short>::_f2();
                              */
    template class X<int>;   /* Will instantiate the following:
                              * X<int>::f();
                              * X<int>::_f1();
                              */
    template class X<float>; /* Will instantiate the following:
                              * X<float>::f();
                              */
    template class X<double>;/* Will instantiate the following:
                              * X<double>::f();
                              * X<double>::_f2();
                              * X<double>::_f1();
                              */


Technical Summary
=====================
    
In summary, this proposal makes the following changes to the standard:

* Allow the programmer to declare additional class methods outside of the
    class definition. These so called *private extension method*
    declarations must be prefixed by the `private` keyword.
    These methods will have private access control.
* The programmer may specify the `static` keyword *after* the `private` keyword to declare a private static extension member function.
* The programmer may specify the `static` keyword *before* the `private` keyword to declare a private extension method with internal linkage.
* The programmer may specify the `static` keyword twice, once  *before* and once *after* the `private` keyword to declare a private static extension member function with internal linkage.
* The programmer may declare a private extension method or private static extension member function within an anonymous namespace to give the method internal linkage. This namespace *must* be a member of the namespace which contains the given class.

Counter Arguments
==================

We will now address some arguments against this proposal.

Violating Access Control
------------------------

One immediate and common objection to this proposal is that it may break encapsulation by allowing
the programmer to subvert access control. By definition, a PEM has private access control and thus
they cannot be called outside of the class scope. There is however, one exploit
discovered by Richard Smith where merely the existence of an additional class method
which is never called allows a violation of access control.

    class A {
      int n;
    public:
      A() : n(42) {}
    };
    
    template<typename T> struct X {
      static decltype(T()()) t;
    };
    template<typename T> decltype(T()()) X<T>::t = T()();
    
    int A::*p;
    private int A::expose_private_member() { // note, not called anywhere
      struct DoIt {
        int operator()() {
          p = &A::n;
          return 0;
        }
      };
      return X<DoIt>::t; // odr-use of X<DoIt>::t triggers instantiation
    }
    
    int main() {
      A a;
      return a.*p; // read private member
    }
    
This example would be a newly introduced standards conforming way to violate access
control if this proposal were to be accepted.

This is not actually a problem. This example is artificially contrived to exploit
access control. It is not a common error that would be made by novies nor
is it an obvious or easy to use tool to abuse access control and write poor interfaces.
Indeed, many methods of violating access control already exist
within the current language \[[GotW076](GotW076)\]. We agree with the author of that article.

"The issue here is of protecting against Murphy vs. protecting against Machiavelli... that is, protecting against accidental misuse (which the language does very well) vs. protecting against deliberate abuse (which is effectively impossible). In the end, if a programmer wants badly enough to subvert the system, he'll find a way," \[[GotW076](GotW076)\].

Modules will solve this problem
-------------------

Maybe they will, maybe they won't. Modules are still not very well defined. We do not know what modules
will look like when and if they finally arrive. This proposal aims to solve a real problem
in language now rather than the unknown future. It has low implementation overhead. This proposal could also be seen as a step
towards implementing a more complete solution with modules.

There are already current workarounds
--------------------

Likely the most compelling argument against the proposal is that there are already a set of current workarounds.
Friends and/or nested classes can be used to implement a partial variant of PEM in the current language.
For this workaround, nested classes are superior to friends because they can be further extended with additional
nested sub classes where as all friends have to be declared in the original class definition.
An example is provided.

Public header file:

    class X {
      public:
        void doWork();

      private:
        int _i;

        struct XHelper;
    };

Private implementation:

    struct X::XHelper {
      static void doWorkHelper(X& x) { //<-PEM
        x._i = 42;
      }

      struct XHelper2;
    };

    struct X::XHelper::XHelper2 {
      static void doMoreWorkHelper(X& x) { //<-PEM
        x._i++;
      }
    };

    void X::doWork() {
      XHelper::doWorkHelper(x);
      XHelper::XHelper2::doMoreWorkHelper(x);
    }

Pratically, this achieves most of the benefits of PEM, but it has some drawbacks:

* We still leak the `XHelper` symbol (implementation) to the header file (interface).
* All of the PEMs implemented by the helper cannot access the data members of X
    directly. They must use C style syntax `x._i = 42` instead of the more natural `_i = 42` provided by the implicit `this` pointer.
* The set of PEMs are restricted to the class definition of `XHelper`. We cannot arbitrarily introduce new PEMS without modifying `XHelper` or creating yet another subclass of `XHelper`.
* `XHelper` and all of it's methods cannot have internal linkage.
* `XHelper` static member function signatures cannot use symbols with internal linkage unless `XHelper` itself resides in only one translation unit.
* This technique is obscure and not well known, a first class language feature would be more accessible to new users.
* The fact that this technique was discovered shows a need for PEMs in the community.

Alternatives and Additions
===================

The current approach uses the private keyword to declare a PEM or PSEMF.
What are the consequences of this syntax?

Pros:

 - Completely backwards compatible
 - Does not invent new keywords or unusual syntax
 - Neatly resolves the 2 uses of static (internal linkage vs static class method)
 - Private keyword has a natural meaning here

Cons:

 - Novice programmers will inevitably ask why they cannot declare public and protected extension methods.

Might there be a better approach? We will discuss some alternatives.

Use a different keyword
-------------------------

We could use a keyword other than private. 
Here are some possible candidates, we believe them all to be inferior to private.

* explicit: The explicit keyword could be reused here and it even
    seems somewhat natural as you are "explicitly" defining
    a new member function. All of the current uses of this keyword
    are related to type conversions and it would be better to keep
    it this way for consistency and ease of understanding.
    Do we really want another overloaded keyword like
    static in the language?
* [[attribute]]: This is almost like inventing a new keyword, except
    you're cheating by instead using a generalized attribute.
    Attributes should not be used for back door keywords 
    to implement new language features.
* Invent a new keyword: This probably doesn't even need to be said.
    New keywords are something that should only be introduced if
    there is a really good reason. This is a small proposal that
    does not merit a new keyword.



Remove the keyword
------------------------------

Another approach is to avoid the use of a keyword entirely. This creates a problem
with the double meaning of the static keyword. We resolve it by moving the
static keyword after the entire function signature, similar to const.

    class Foo {};

    void Foo::_f1(); //<-PEM
    void Foo::_f2() static; //<-PSEMF
    static void Foo::_f3(); //<-PEM with internal linkage
    static void Foo::_f4() static; //<-PSEMF with internal linkage

Pros:

 - Completely backwards compatible
 - Avoids the confusion private vs public/protected extension methods.

Cons:

 - Previously if the user made a typo when defining a class method, he would get a compiler error. Now he will silently declare a new private extension method. This error is likely to be caught during link time however.


Reopening the class scope
-------------------------

One possible addition to this proposal would be to allow reopening the class private scope
to add not only new member functions but also typedefs and nested types. This is somewhat
reminiscent of the style of the `extern "C"` feature.
Credit goes to Péter Radics for the initial idea for the idea of reopening the class scope
and Vicente J. Botet Escriba for the handy syntax using private.

    class Foo {};

    private Foo { //<-Reopen Foo's private scope
      
      void _f(); //<-PEM
      static void _g(); //<-PSEMF

      int _x; //<-A static data member. We cannot add data members to Foo! 
              //Maybe this should be a compiler error?
      static int _y; //<-Another static data member.

      typedef float real; //<-A typedef, which is private to Foo

      class Bar; //<-Forward declaration of a nested class Foo::Bar

      class Baz {}; //<-Define a nested class Foo::Baz

      friend class Gaz; //<-Error: Cannot declare extended friends! This is a horrible break in encapsulation!
    };

    static private Foo { //<-Ropen Foo's private scope again, this time with internal linkage
      /* stuff */
    };

    namespace {
       private Foo { //<-Reopen yet again with internal linkage using anonymous namespace.
         /* stuff */
       };
    };

Internal Linkage by Default
----------------------

Several people have suggested that perhaps PEMs should always have internal linkage by default, requiring
an extra keyword for external linkage. We could use the `extern` keyword to denote external linkage. 
This would also allow the `static` keyword to freely move before and after the `private` keyword.
We also avoid the need for anonymous namespaces.

The syntax for this idea might look like the following:

    class Foo {};

    private void Foo::_f1(); //<-PEM with internal linkage
    extern private void Foo::_f2(); //<-PEM with external linkage
    private static void Foo::_f3(); //<-PEM //<-PSEMF with internal linkage
    static private void Foo::_f3(); //<-PEM //<-PSEMF with internal linkage
    extern private static void Foo::_f3(); //<-PEM //<-PSEMF with external linkage
    extern static private void Foo::_f3(); //<-PEM //<-PSEMF with external linkage

Acknowledgements
====================

Thank you to everyone on the std proposals forum for feedback and suggestions.

References
==================
* <a name="Lakos01"></a>[Lakos01] Lakos, John. *Large-Scale C++ Software Design*, Addison-Wesley, July 1996, ISBN 0201633620.
* <a name="KDEABI"></a>[KDEABI] *Policies/Binary Compatibility Issues With C++ - KDE TechBase*, Available online at <http://techbase.kde.org/Policies/Binary_Compatibility_Issues_With_C++>.
* <a name="GotW076"></a>[Gotw076] Sutter, Herb. *GotW #76: Uses and Abuses of Access Rights*, Available online at <http://www.gotw.ca/gotw/076.htm>.

