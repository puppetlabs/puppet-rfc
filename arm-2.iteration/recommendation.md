Recommendation
==============

The author of this proposal recommends that the ruby-java8 like proposal is used with:

1.  Lambda parameters expressed with `|$x|`
2.  Lambda parameters placed left of the lambda body i.e. `|$x| {e}`
3.  The lambda body may be a block with one or several
    expressions/statements
4.  Enumeration functions `each`, `select`, `reject`, `collect` and `reduce` are
    implemented.
5.  Enumeration picking is:  
    **Array** - one lambda arg picks each value, two args picks index and each value.  
    **Hash**  - one lambda arg picks each entry as a [key, value] array, two args picks key and value.
6.  Any other type of picking is done with functions; suggest `tuples(n=2)´ function to pick n elements.
7.  Literal arguments appear after the curried value by convention, but
    a function (e.g. `reduce`) may insert a value before the first when
    this is logically more sound than having it appear after the curry
    (in reduce the optionally given argument is the "start value").
8.  Calls to functions always pass the optional lambda as the last argument.



Wait with implementation of these options until use-cases are more known:

1. Function reference and uncompleted call (specifying arguments that
   are not curried from the enumerable)
2. Type/define reference (i.e. a call-by-position to call-by-name mapping)

Rationale
---------
The rationale for these recommendations are:

1.  No additional keywords or backwards incompatible changes to the
    language (only some relaxations and new features).
2.  Similar in syntax/semantics to constructs in ruby/java-8
3.  Open ended extensible design expandable via functions
4.  No magic variables
5.  No special cases - need to learn only a few basic principles
6.  Tries to avoid the "I don't get the variable declaration inside the
    braces" feedback (even if it still introduces what may look foreign
    to non developers, at least the idea is that it is easier to grasp -
    UX studies will tell).
7.  A bit more wordy than the pipe examples, but since it more explicit
    it should be easier to read & guess what is going on when first
    being exposed to the syntax.
8.  This proposal is known to be implementable (a working implementation
    exists of almost the entire recommendation).
9.  Does not block or make future more advanced features difficult to
    implement (function reference, uncompleted calls, automatic
    currying, closures).
10. In the IRC discussions to date on puppet-dev, the recommended style
    seems preferable over the pipe/operator based.
11. The "picking logic" mimics Ruby, but avoids having to have an additional function to get index in array
    (which is useful in several use cases; messages, picking from parallell lists, etc.). The more strict; always
    just pass each element is not very practical when iterating over hashes.
