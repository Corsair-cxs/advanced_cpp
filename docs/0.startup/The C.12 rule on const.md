# The C.12 rule on const

Coding guidelines are the Strunk & White for programming languages: famous, influential, in demand, but mediocre. A look at the C.12 rule on const usage from the C++ Core Guidelines.

This article is part of a series of [articles on programming idioms](http://bajamircea.github.io/coding/cpp/2022/10/17/idioms.html).

# The C.12 rule

Consider the [C.12 rule in the C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-constref). Its contents says:

***C.12: Don’t make data members const or references\***

***Reason\***

*They are not useful, and make types difficult to use by making them either uncopyable or partially uncopyable for subtle reasons.*

**Example; bad**

```
class bad {
    const int i;    // bad
    string& s;      // bad
    // ...
};
```

*The `const` and `&` data members make this class “only-sort-of-copyable” – copy-constructible but not copy-assignable.*

***Note\***

*If you need a member to point to something, use a pointer (raw or smart, and `gsl::not_null` if it should not be `null`) instead of a reference.*

***Enforcement\***

*Flag a data member that is `const`, `&`, or `&&`.*

# Critique of the rule

One issue with coding guidelines is that they are not necessarily explicit about the idioms they refer to.

To the regular data idiom this rule applies.

But we’ve seen in glorious details that in the mockable interfaces idiom we have a good reason to use references for the member interfaces or `const` for the member string. We’ve also seen the `lower_bound_predicate` using both const and reference.

In this particular case a combination of the fact that it refers to member variables as data members and the fact that it is in a section that mentions concrete/regular classes suggests that the rule is supposed to apply to regular data classes from the regular idiom.

**Literal interpretation of coding guidelines without applying thorough reasoning is a dangerous thing. Regard with suspicion the developer that quotes the coding rule as the single justification instead of providing a compelling rational explanation.**

# Notable public C++ coding standards

Bjarne Stoustrup produced a [C++ coding standard for the Joint Strike Fighter](https://www.stroustrup.com/JSF-AV-rules.pdf), basically for embedded systems on an airplane. C++ exceptions were not allowed due to the lack of guarantees required for such a system. Many users thought “but my system is different” so the standard was ignored by large parts of the C++ community.

[Google then published their C++ Style Guide](https://google.github.io/styleguide/cppguide.html). They did not use exceptions either. In this case due to being cautious of introducing them in a large, legacy code that did not use C++ exceptions.

Bjarne Stroustrup and Herb Sutter merged code guidelines from the organisations where they worked, Morgan Stanley and Microsoft, and others into the [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuideline), which then continued as a publicly available guideline, from where the rule above is taken.

# Parallels to Strunk & White

Strunk & White is a language style guide that is often familiar to US college/university students and it’s supposed to contain advice on good writing style for such students. Surprisingly it’s almost unknown in Britain.

Like many programming coding guidelies, it’s popular and written in a similar simplistic style of rules like “always to this” or “never do this”, for example use the active voice not the passive voice.

The advice does not address the questions “then why passive voice exists, where is it useful?” and they hilariosly contradict themselves by using the passive voice to provide the advice:

> Many a tame sentence … can be made lively and emphatic by substituting a transitive in the active voice

And that creates a wrong frame of mind. There are better books on language style out there that for have more complex, but more effective rules e.g. identify what is known, old information, vs. what is newly introduced to the reader and the passive voice can be used to make that transition smoothly rather than jumping back between new and old.

# Conclusion

Having C++ coding standards is better than not having any. They are often not explicit about the idioms they support or the fact that multiple idioms might coexist in a large code base. They are often phrased as over simplified rules and often lack context.

Thy are not state of the art, it’s not where ideas and sound judgement come from.

# References

- Strunk & White: [The Elements of Style](https://en.wikipedia.org/wiki/The_Elements_of_Style)
- [Pullum on Passives](https://www.youtube.com/watch?v=ZrRKJrTPwYg)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-constref)
- There are all kinds of types: [C++ class taxonomy](http://bajamircea.github.io/coding/cpp/2015/11/05/class-taxonomy.html)
- [Series of articles on programming idioms](http://bajamircea.github.io/coding/cpp/2022/10/17/idioms.html)