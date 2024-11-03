## Universal References in C++11 -- Scott Meyers

By Blog Staff | Nov 1, 2012 01:07 PM | Tags: [intermediate](https://isocpp.org/blog/tag/intermediate) [advanced](https://isocpp.org/blog/tag/advanced)

# ![img](./img/Universal%20References%20in%20C++11/meyers.png)Universal References in C++11

*T&& Doesn’t Always Mean “Rvalue Reference”*

by Scott Meyers

 

> Related materials:
>
> - A video of Scott’s C&B talk based on this material [is available on Channel 9](http://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Scott-Meyers-Universal-References-in-Cpp11).
> - A black-and-white PDF version of this article [is available in Overload 111](https://t.co/ou7sg6AUFX).

 

Perhaps the most significant new feature in C++11 is rvalue references; they’re the foundation on which move semantics and perfect forwarding are built. (If you’re unfamiliar with the basics of rvalue references, move semantics, or perfect forwarding, you may wish to read [Thomas Becker’s overview](http://thbecker.net/articles/rvalue_references/section_01.html) before continuing.)

Syntactically, rvalue references are declared like “normal” references (now known as *lvalue references*), except you use two ampersands instead of one. This function takes a parameter of type rvalue-reference-to-`Widget`:

```cpp
void f(Widget&& param);
```

Given that rvalue references are declared using “`&&`”, it seems reasonable to assume that the presence of “`&&`” in a type declaration indicates an rvalue reference. That is not the case:

```cpp
Widget&& var1 = someWidget;      // here, “&&” means rvalue reference auto&& var2 = var1;              // here, “&&” does not mean rvalue reference template<typename T>void f(std::vector<T>&& param);  // here, “&&” means rvalue reference template<typename T>void f(T&& param);               // here, “&&”does not mean rvalue reference
```

In this article, I describe the two meanings of “`&&`” in type declarations, explain how to tell them apart, and introduce new terminology that makes it possible to unambiguously communicate which meaning of “`&&`” is intended. Distinguishing the different meanings is important, because if you think “rvalue reference” whenever you see “`&&`” in a type declaration, you’ll misread a lot of C++11 code.

The essence of the issue is that “`&&`” in a type declaration sometimes means rvalue reference, but sometimes it means *either* rvalue reference *or* lvalue reference. As such, some occurrences of “`&&`” in source code may actually have the meaning of “`&`”, i.e., have the syntactic *appearance* of an rvalue reference (“`&&`”), but the *meaning* of an lvalue reference (“`&`”). References where this is possible are more flexible than either lvalue references or rvalue references. Rvalue references may bind only to rvalues, for example, and lvalue references, in addition to being able to bind to lvalues, may bind to rvalues only under restricted circumstances.[1] In contrast, references declared with “`&&`” that may be either lvalue references or rvalue references may bind to *anything*. Such unusually flexible references deserve their own name. I call them *universal references*.

The details of when “`&&`” indicates a universal reference (i.e., when “`&&`” in source code might actually mean “`&`”) are tricky, so I’m going to postpone coverage of the minutiae until later. For now, let’s focus on the following rule of thumb, because that is what you need to remember during day-to-day programming:

> If a variable or parameter is declared to have type **`T&&`** for some **deduced type** `T`, that variable or parameter is a *universal reference*.

The requirement that type deduction be involved limits the situations where universal references can be found. In practice, almost all universal references are parameters to function templates. Because the type deduction rules for `auto`-declared variables are essentially the same as for templates, it’s also possible to have `auto`-declared universal references. These are uncommon in production code, but I show some in this article, because they are less verbose in examples than templates. In the [Nitty Gritty Details section](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers#NittyGrittyDetails) of this article, I explain that it’s also possible for universal references to arise in conjunction with uses of `typedef` and `decltype`, but until we get down to the nitty gritty details, I’m going to proceed as if universal references pertained only to function template parameters and `auto`-declared variables.

The constraint that the form of a universal reference be `T&&` is more significant than it may appear, but I’ll defer examination of that until a bit later. For now, please simply make a mental note of the requirement.

Like all references, universal references must be initialized, and it is a universal reference’s initializer that determines whether it represents an lvalue reference or an rvalue reference:

- If the expression initializing a universal reference is an lvalue, the universal reference becomes an lvalue reference.
- If the expression initializing the universal reference is an rvalue, the universal reference becomes an rvalue reference.

This information is useful only if you are able to distinguish lvalues from rvalues. A precise definition for these terms is difficult to develop (the C++11 standard generally specifies whether an expression is an lvalue or an rvalue on a case-by-case basis), but in practice, the following suffices:

- If you can take the address of an expression, the expression is an lvalue.
- If the type of an expression is an lvalue reference (e.g., `T&` or `const T&`, etc.), that expression is an lvalue. 
- Otherwise, the expression is an rvalue. Conceptually (and typically also in fact), rvalues correspond to temporary objects, such as those returned from functions or created through implicit type conversions. Most literal values (e.g., `10` and `5.3`) are also rvalues.

Consider again the following code from the beginning of this article:

```cpp
Widget&& var1 = someWidget;auto&& var2 = var1;
```

You can take the address of `var1`, so `var1` is an lvalue. `var2`’s type declaration of `auto&&` makes it a universal reference, and because it’s being initialized with `var1` (an lvalue), `var2` becomes an lvalue reference. A casual reading of the source code could lead you to believe that `var2` was an rvalue reference; the “`&&`” in its declaration certainly suggests that conclusion. But because it is a universal reference being initialized with an lvalue, `var2` becomes an lvalue reference. It’s as if `var2` were declared like this:

```cpp
Widget& var2 = var1;
```

As noted above, if an expression has type lvalue reference, it’s an lvalue. Consider this example:

```cpp
std::vector<int> v;...auto&& val = v[0];               // val becomes an lvalue reference (see below)
```

`val` is a universal reference, and it’s being initialized with `v[0]`, i.e., with the result of a call to `std::vector<int>::operator[]`. That function returns an lvalue reference to an element of the vector.[2]  Because all lvalue references are lvalues, and because this lvalue is used to initialize `val`, `val` becomes an lvalue reference, even though it’s declared with what looks like an rvalue reference.

I remarked that universal references are most common as parameters in template functions. Consider again this template from the beginning of this article:

```cpp
template<typename T>void f(T&& param);               // “&&” might mean rvalue reference
```

Given this call to f,

```cpp
f(10);                           // 10 is an rvalue
```

`param` is initialized with the literal `10`, which, because you can’t take its address, is an rvalue. That means that in the call to `f`, the universal reference `param` is initialized with an rvalue, so `param` becomes an rvalue reference – in particular, `int&&`.

On the other hand, if `f` is called like this,

```cpp
int x = 10;f(x);                            // x is an lvalue
```

`param` is initialized with the variable `x`, which, because you can take its address, is an lvalue. That means that in this call to `f`, the universal reference param is initialized with an lvalue, and param therefore becomes an lvalue reference – `int&`, to be precise.

The comment next to the declaration of `f` should now be clear: whether `param`’s type is an lvalue reference or an rvalue reference depends on what is passed when `f` is called. Sometimes param becomes an lvalue reference, and sometimes it becomes an rvalue reference. `param` really is a *universal reference*.

Remember that “`&&`” indicates a universal reference *only where type deduction takes place*. Where there’s no type deduction, there’s no universal reference. In such cases, “`&&`” in type declarations always means rvalue reference. Hence:

```cpp
template<typename T>void f(T&& param);               // deduced parameter type ⇒ type deduction;                                 // && ≡ universal reference template<typename T>class Widget {    ...    Widget(Widget&& rhs);        // fully specified parameter type ⇒ no type deduction;    ...                          // && ≡ rvalue reference}; template<typename T1>class Gadget {    ...    template<typename T2>    Gadget(T2&& rhs);            // deduced parameter type ⇒ type deduction;    ...                          // && ≡ universal reference}; void f(Widget&& param);          // fully specified parameter type ⇒ no type deduction;                                 // && ≡ rvalue reference
```

There’s nothing surprising about these examples. In each case, if you see `T&&` (where `T` is a template parameter), there’s type deduction, so you’re looking at a universal reference. And if you see “`&&`” after a particular type name (e.g., `Widget&&`), you’re looking at an rvalue reference.

I stated that the form of the reference declaration must be “`T&&`” in order for the reference to be universal. That’s an important caveat. Look again at this declaration from the beginning of this article:

```cpp
template<typename T>void f(std::vector<T>&& param);     // “&&” means rvalue reference
```

Here, we have both type deduction and a “`&&`”-declared function parameter, but the form of the parameter declaration is not “`T&&`”, it’s “`std::vector<t>&&`”. As a result, the parameter is a normal rvalue reference, not a universal reference. Universal references can only occur in the form “`T&&`”! Even the simple addition of a `const` qualifier is enough to disable the interpretation of “`&&`” as a universal reference:

```cpp
template<typename T>void f(const T&& param);               // “&&” means rvalue reference
```

Now, “`T&&`” is simply the required *form* for a universal reference. It doesn’t mean you have to use the name `T` for your template parameter:

```cpp
template<typename MyTemplateParamType>void f(MyTemplateParamType&& param);  // “&&” means universal reference
```

Sometimes you can see `T&&` in a function template declaration where `T` is a template parameter, yet there’s still no type deduction. Consider this `push_back` function in `std::vector`:[3]

```cpp
template <class T, class Allocator = allocator<T> >class vector {public:    ...    void push_back(T&& x);       // fully specified parameter type ⇒ no type deduction;    ...                          // && ≡ rvalue reference};
```

Here, `T` is a template parameter, and `push_back` takes a `T&&`, yet the parameter is not a universal reference! How can that be?

The answer becomes apparent if we look at how `push_back` would be declared outside the class. I’m going to pretend that `std::vector`’s `Allocator` parameter doesn’t exist, because it’s irrelevant to the discussion, and it just clutters up the code. With that in mind, here’s the declaration for this version of `std::vector::push_back`:

```cpp
template <class T>void vector<T>::push_back(T&& x);
```

`push_back` can’t exist without the class `std::vector<T>` that contains it. But if we have a class `std::vector<T>`, we already know what `T` is, so there’s no need to deduce it.

An example will help. If I write

```cpp
Widget makeWidget();             // factory function for Widgetstd::vector<Widget> vw;...Widget w;vw.push_back(makeWidget());      // create Widget from factory, add it to vw
```

my use of `push_back` will cause the compiler to instantiate that function for the class `std::vector<Widget>`. The declaration for that `push_back` looks like this:

```cpp
void std::vector<Widget>::push_back(Widget&& x);
```

See? Once we know that the class is `std::vector<Widget>`, the type of `push_back`’s parameter is fully determined: it’s `Widget&&`. There’s no role here for type deduction.

Contrast that with `std::vector`’s `emplace_back`, which is declared like this:

```cpp
template <class T, class Allocator = allocator<T> >class vector {public:    ...    template <class... Args>    void emplace_back(Args&&... args); // deduced parameter types ⇒ type deduction;    ...                                // && ≡ universal references};
```

Don’t let the fact that `emplace_back` takes a variable number of arguments (as indicated by the ellipses in the declarations for `Args` and `args`) distract you from the fact that a type for each of those arguments must be deduced. The function template parameter `Args` is independent of the class template parameter `T`, so even if we know that the class is, say, `std::vector<Widget>`, that doesn’t tell us the type(s) taken by `emplace_back`. The out-of-class declaration for `emplace_back` for `std::vector<Widget>` makes that clear (I’m continuing to ignore the existence of the `Allocator` parameter):

```cpp
template<class... Args>void std::vector<Widget>::emplace_back(Args&&... args);
```

Clearly, knowing that the class is `std::vector<Widget>` doesn’t eliminate the need for the compiler to deduce the type(s) passed to `emplace_back`. As a result, `std::vector::emplace_back`’s parameters are universal references, unlike the parameter to the version of `std::vector::push_back` we examined, which is an rvalue reference.

A final point is worth bearing in mind: the lvalueness or rvalueness of an expression is independent of its type. Consider the type `int`. There are lvalues of type `int` (e.g., variables declared to be `int`s), and there are rvalues of type `int` (e.g., literals like `10`). It’s the same for user-defined types like `Widget`. A `Widget` object can be an lvalue (e.g., a `Widget` variable) or an rvalue (e.g., an object returned from a `Widget`-creating factory function). The type of an expression does not tell you whether it is an lvalue or an rvalue.
Because the lvalueness or rvalueness of an expression is independent of its type, it’s possible to have *lvalues* whose type is *rvalue reference*, and it’s also possible to have *rvalues* of the type *rvalue reference*:

```cpp
Widget makeWidget();                       // factory function for Widget Widget&& var1 = makeWidget()               // var1 is an lvalue, but                                           // its type is rvalue reference (to Widget) Widget var2 = static_cast<Widget&&>(var1); // the cast expression yields an rvalue, but                                           // its type is rvalue reference  (to Widget)
```

The conventional way to turn lvalues (such as `var1`) into rvalues is to use `std::move` on them, so `var2` could be defined like this:

```cpp
Widget var2 = std::move(var1);             // equivalent to above
```

I initially showed the code with `static_cast` only to make explicit that the type of the expression was an rvalue reference (`Widget&&`).

Named variables and parameters of rvalue reference type are lvalues. (You can take their addresses.) Consider again the `Widget` and `Gadget` templates from earlier:

```cpp
template<typename T>class Widget {    ...    Widget(Widget&& rhs);        // rhs’s type is rvalue reference,    ...                          // but rhs itself is an lvalue}; template<typename T1>class Gadget {    ...    template <typename T2>    Gadget(T2&& rhs);            // rhs is a universal reference whose type will    ...                          // eventually become an rvalue reference or};                               // an lvalue reference, but rhs itself is an lvalue
```

In `Widget`’s constructor, `rhs` is an rvalue reference, so we know it’s bound to an rvalue (i.e., an rvalue was passed to it), but `rhs` itself is an lvalue, so we have to convert it back to an rvalue if we want to take advantage of the rvalueness of what it’s bound to. Our motivation for this is generally to use it as the source of a move operation, and that’s why the way to convert an lvalue to an rvalue is to use `std::move`. Similarly, `rhs` in `Gadget`’s constructor is a universal reference, so it might be bound to an lvalue or to an rvalue, but regardless of what it’s bound to, `rhs` itself is an lvalue. If it’s bound to an rvalue and we want to take advantage of the rvalueness of what it’s bound to, we have to convert `rhs` back into an rvalue. If it’s bound to an lvalue, of course, we don’t want to treat it like an rvalue. This ambiguity regarding the lvalueness and rvalueness of what a universal reference is bound to is the motivation for `std::forward`: to take a universal reference lvalue and convert it into an rvalue only if the expression it’s bound to is an rvalue. The name of the function (“`forward`”) is an acknowledgment that our desire to perform such a conversion is virtually always to preserve the calling argument’s lvalueness or rvalueness when passing – *forwarding* – it to another function.

But `std::move` and `std::forward` are not the focus of this article. The fact that “`&&`” in type declarations may or may not declare an rvalue reference is. To avoid diluting that focus, I’ll refer you to the references in the [Further Information section](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers#FurtherInformation) for information on `std::move` and `std::forward`.

## Nitty Gritty Details

The true core of the issue is that some constructs in C++11 give rise to references to references, and references to references are not permitted in C++. If source code explicitly contains a reference to a reference, the code is invalid:

```cpp
Widget w1;...Widget& & w2 = w1;               // error! No such thing as “reference to reference”
```

There are cases, however, where references to references arise as a result of type manipulations that take place during compilation, and in such cases, rejecting the code would be problematic. We know this from experience with the initial standard for C++, i.e., C++98/C++03.

During type deduction for a template parameter that is a universal reference, lvalues and rvalues of the same type are deduced to have slightly different types. In particular, lvalues of type `T` are deduced to be of type `T&` (i.e., lvalue reference to `T`), while rvalues of type `T` are deduced to be simply of type `T`. (Note that while lvalues are deduced to be lvalue references, rvalues are not deduced to be rvalue references!) Consider what happens when a template function taking a universal reference is invoked with an rvalue and with an lvalue:

```cpp
template<typename T>void f(T&& param); ... int x; ... f(10);                           // invoke f on rvaluef(x);                            // invoke f on lvalue
```

In the call to `f` with the rvalue `10`, `T` is deduced to be `int`, and the instantiated `f` looks like this:

```cpp
void f(int&& param);             // f instantiated from rvalue
```

That’s fine. In the call to `f` with the lvalue `x`, however, `T` is deduced to be `int&`, and `f`’s instantiation contains a reference to a reference:

```cpp
void f(int& && param);           // initial instantiation of f with lvalue
```

Because of the reference-to-reference, this instantiated code is *prima facie* invalid, but the source code– “`f(x)`” – is completely reasonable. To avoid rejecting it, C++11 performs “reference collapsing” when references to references arise in contexts such as template instantiation.

Because there are two kinds of references (lvalue references and rvalue references), there are four possible reference-reference combinations: lvalue reference to lvalue reference, lvalue reference to rvalue reference, rvalue reference to lvalue reference, and rvalue reference to rvalue reference. There are only two reference-collapsing rules:

- An rvalue reference to an rvalue reference becomes (“collapses into”) an rvalue reference.
- All other references to references (i.e., all combinations involving an lvalue reference) collapse into an lvalue reference.

Applying these rules to the instantiation of f on an lvalue yields the following valid code, which is how the compiler treats the call:

```cpp
void f(int& param);              // instantiation of f with lvalue after reference collapsing
```

This demonstrates the precise mechanism by which a universal reference can, after type deduction and reference collapsing, become an lvalue reference. The truth is that a universal reference is really just an rvalue reference in a reference-collapsing context.

Things get subtler when deducing the type for a variable that is itself a reference. In that case, the reference part of the type is ignored. For example, given

```cpp
int x; ... int&& r1 = 10;                   // r1’s type is int&& int& r2 = x;                     // r2’s type is int&
```

the type for both `r1` and `r2` is considered to be `int` in a call to the template `f`. This reference-stripping behavior is independent of the rule that, during type deduction for universal references, lvalues are deduced to be of type `T&` and rvalues of type `T`, so given these calls,

```cpp
f(r1); f(r2);
```

the deduced type for both `r1` and `r2` is `int&`. Why? First the reference parts of `r1`’s and `r2`’s types are stripped off (yielding `int` in both cases), then, because each is an lvalue, each is treated as `int&` during type deduction for the universal reference parameter in the call to `f`.

Reference collapsing occurs, as I’ve noted, in “contexts such as template instantiation.” A second such context is the definition of `auto` variables. Type deduction for `auto` variables that are universal references is essentially identical to type deduction for function template parameters that are universal references, so lvalues of type `T` are deduced to have type `T&`, and rvalues of type `T` are deduced to have type `T`. Consider again this example from the beginning of this article:

```cpp
Widget&& var1 = someWidget;      // var1 is of type Widget&& (no use of auto here) auto&& var2 = var1;              // var2 is of type Widget& (see below)
```

`var1` is of type `Widget&&`, but its reference-ness is ignored during type deduction in the initialization of `var2`; it’s considered to be of type `Widget`. Because it’s an lvalue being used to initialize a universal reference (`var2`), its deduced type is `Widget&`. Substituting `Widget&` for `auto` in the definition for `var2` yields the following invalid code,

```cpp
Widget& && var2 = var1;          // note reference-to-reference
```

which, after reference collapsing, becomes

```cpp
Widget& var2 = var1;             // var2 is of type Widget&
```

A third reference-collapsing context is `typedef` formation and use. Given this class template,

```cpp
template<typename T>class Widget {    typedef T& LvalueRefType;    ...};
```

and this use of the template,

```cpp
Widget<int&> w;
```

the instantiated class would contain this (invalid) typedef:

```cpp
typedef int& & LvalueRefType;
```

Reference-collapsing reduces it to this legitimate code:

```cpp
typedef int& LvalueRefType;
```

If we then use this `typedef` in a context where references are applied to it, e.g.,

```cpp
void f(Widget<int&>::LvalueRefType&& param);
```

the following invalid code is produced after expansion of the `typedef`,

```cpp
void f(int& && param);
```

but reference-collapsing kicks in, so `f`’s ultimate declaration is this:

```cpp
void f(int& param);
```

The final context in which reference-collapsing takes place is the use of `decltype`. As is the case with templates and `auto`, `decltype` performs type deduction on expressions that yield types that are either `T` or `T&`, and `decltype` then applies C++11’s reference-collapsing rules. Alas, the type-deduction rules employed by `decltype` are not the same as those used during template or `auto` type deduction. The details are too arcane for coverage here (the [Further Information section](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers#FurtherInformation) provides pointers to, er, further information), but a noteworthy difference is that `decltype`, given a named variable of non-reference type, deduces the type `T` (i.e., a non-reference type), while under the same conditions, templates and `auto` deduce the type `T&`. Another important difference is that `decltype`’s type deduction depends only on the `decltype` expression; the type of the initializing expression (if any) is ignored. Ergo:

```cpp
Widget w1, w2; auto&& v1 = w1;                  // v1 is an auto-based universal reference being                                 // initialized with an lvalue, so v1 becomes an                                 // lvalue reference referring to w1. decltype(w1)&& v2 = w2;          // v2 is a decltype-based universal reference, and                                 // decltype(w1) is Widget, so v2 becomes an rvalue reference.                                 // w2 is an lvalue, and it’s not legal to initialize an                                 // rvalue reference with an lvalue, so                                 // this code does not compile.
```

## Summary

In a type declaration, “`&&`” indicates either an rvalue reference or a *universal reference* – a reference that may resolve to either an lvalue reference or an rvalue reference. Universal references always have the form `T&&` for some deduced type `T`.

*Reference collapsing* is the mechanism that leads to universal references (which are really just rvalue references in situations where reference-collapsing takes place) sometimes resolving to lvalue references and sometimes to rvalue references. It occurs in specified contexts where references to references may arise during compilation. Those contexts are template type deduction, `auto` type deduction, `typedef` formation and use, and `decltype` expressions.

## Acknowledgments

Draft versions of this article were reviewed by Cassio Neri, Michal Mocny, Howard Hinnant, Andrei Alexandrescu, Stephan T. Lavavej, Roger Orr, Chris Oldwood, Jonathan Wakely, and Anthony Williams. Their comments contributed to substantial improvements in the content of the article as well as in its presentation.

## Notes

[1] I discuss rvalues and their counterpart, lvalues, later in this article. The restriction on lvalue references binding to rvalues is that such binding is permitted only when the lvalue reference is declared as a reference-to-`const`, i.e., a `const T&`.

[2] I’m ignoring the possibility of bounds violations. They yield undefined behavior.

[3] `std::vector::push_back` is overloaded. The version shown is the only one that interests us in this article.

## Further Information

[C++11](http://en.wikipedia.org/wiki/C%2B%2B11), Wikipedia.

[Overview of the New C++ (C++11)](http://www.artima.com/shop/overview_of_the_new_cpp), Scott Meyers, Artima Press, last updated January 2012.

[C++ Rvalue References Explained](http://thbecker.net/articles/rvalue_references/section_01.html), Thomas Becker, last updated September 2011.

[decltype](http://en.wikipedia.org/wiki/Decltype), Wikipedia.

[“A Note About decltype,”](http://drdobbs.com/blogs/cpp/231002789) Andrew Koenig, Dr. Dobb’s, 27 July 2011.