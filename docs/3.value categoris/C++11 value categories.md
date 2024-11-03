C++11 introduced new value categories. Anders Schau Knatten explains his way of thinking about them.

Did you used to have some sort of intuition for what ‘lvalue’ and ‘rvalue’ mean? Are you confused about glvalues, xvalues and prvalues, and worry that lvalues and rvalues might also have changed? This article aims to help you develop a basic intuition for all five of them.

First, a warning: This article does not give a complete definition of the five value categories. Instead, I give a basic outline, which I hope will help to have in the back of your mind the next time you need to look up the actual details of one of them.

Back before C++11, there were two value categories, lvalue and rvalue. The basic intuition was that lvalues were things with identities, such as variables, and rvalues were expressions evaluating to temporaries (with no identity). Consider these definitions:

```c++
Widget w;
Widget getWidget();
```


If we now use the expression w anywhere, it evaluates to the object w , which has identity. If we use the expression getWidget() , it evaluates to a temporary return value with no identity. Let’s visualise it like this:

![img](./img/C++11%20value%20categories/0.png)

However, along came rvalue references and move semantics. On the surface, the old lvalue/rvalue distinction seems sufficient: Never move from lvalues (people might still be using them), feel free to move from rvalues (they’re just temporary anyway). Let’s add movability to our diagram:

![img](./img/C++11%20value%20categories/1.png)

Why did I put ‘Can’t move’ and ‘lvalue’ in red in that diagram? It turns out that you might want to move from certain lvalues! For instance, if you have a variable you won’t be using anymore, you can std::move() it to cast it to an rvalue reference. A function can also return an rvalue reference to an object with identity.

So as it turns out, whether something has identity, and whether something can be moved from, are orthogonal properties ! We’ll solve the problem of moving from lvalues soon, but first, let’s just change our diagram to reflect our new orthogonal view of the world:

![img](./img/C++11%20value%20categories/2.png)

Clearly, there’s a name missing in the lower left corner here. (We can ignore the top right corner, temporaries which can’t be moved from is not a useful concept.)

C++11 introduces a new value category ‘xvalue’, for lvalues which can be moved from. It might help to think of ‘xvalue’ as ‘eXpiring lvalue’, since they’re probably about to end their lifetime and be moved from (for instance a function returning an rvalue reference).

In addition, what was formerly called ‘rvalue’ was renamed to ‘prvalue’, meaning ‘pure rvalue’. These are the three basic value categories:

![img](./img/C++11%20value%20categories/3.png)

But we’re not quite there yet, what’s a ‘glvalue’, and what does ‘rvalue’ mean these days? It turns out that we’ve already explained these concepts! We just haven’t given them proper names yet.

![img](./img/C++11%20value%20categories/4.png)

A glvalue, or ‘generalized lvalue’, covers exactly the ‘has identity’ property, ignoring movability. An rvalue covers exactly the ‘can move’ property, ignoring identity. And that’s it! You now know all the five value categories.

If you want to go into further detail about this topic, cppreference has a very good article [ cppreference ].

This article was first published on Anders Schau Knatten’s blog, C++ on a Friday , on 9 March 2019 at https://blog.knatten.org/2018/03/09/lvalues-rvalues-glvalues-prvalues-xvalues-help/

Reference
[cppreference] Value categories: https://en.cppreference.com/w/cpp/language/value_category

Anders Schau Knatten makes robot eyes at Zivid, where he strives for simplicity, stability and expressive code. He’s also the author of CppQuiz and @AffectiveCpp, which strive for none of the above.