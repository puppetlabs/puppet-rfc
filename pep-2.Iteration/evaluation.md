Decision/Evaluation Support
===========================

In order to help with the reading and evaluation/decision making, this
is one way of breaking down the reading/decisions. There is a
recommendation after the numbered decisions if you just want to look at
"what the author of the document prefers".

1.  The two proposals are based on the requirement that the solution
    should not break existing code (i.e. introduction of new keywords,
    changes to naming rules, breaking API changes are not ok). This to
    enable introducing this feature in the 3.x puppet stream.

    1.  If you do not care about backwards compatibility some of the non
        explored ideas may be more appealing to you, or you may have other
        ideas to present.
        
2.  As both of the proposals are built on lambdas being added to the
    language, this is naturally the first decision point.

    1.  If you do not like the idea of a lambda, one of the listed
        alternatives needs to be elaborated (or feel free to suggest a
        concrete implementable proposal)
    2.  If you after reading about the ideas still feel uncomfortable, you
        should know it is possible to restrict the types of
        expressions/statements available when in a lambda. As an example, it
        should probably not be allowed to create new classes and defines
        since these are not allowed to be conditional in regular puppet
        logic. Also know that most statements that currently do not produce
        an r-value can easily be made to do so; a resource creation can
        return a reference to the created resource, etc. The set of allowed
        expressions in a lambda have not yet been detailed pending an
        overall decision if lambdas are going in or not.

3.  There are two lambda styles to choose from; the ruby "parameters
    first in a block" like syntax, and the java-8 like "parameters
    outside block" syntax.

    1.  You may like the ruby-like syntax because it is just that, and it
        looks like something someone using ruby immediately would
        understand. Although this syntax gets the job done, it may make
        introduction of some options (applying to a named function by
        reference, and turning an apply on define/type into a lambda) more
        difficult to add later.
    2.  You may like the java-8 like syntax because you think it is clearer
        to non rubyists that there is a left to right flow of data, to a
        parameter declaration, to logic, and it looks more like a classic
        function declaration, only without a name. Incidentally this makes
        it easier to add additional features as "being a lambda" is now not
        tied to the opening {| tokens as in the ruby-like case.
    3.  There are several ways to tweak the java-8 like style to fit puppet.
        Which one is preferred? (If you prefer the "pipe" proposal, option
        i.) must clearly be "something else" (since using | as delimiter
        will not work, and you need to read the discussion about the various
        explored options).

        1.  delimiters around parameters; i.e. `|$x|` or something else?
        2.  arrow or no arrow; i.e, `|$x| => {e}, or |$x| {e}`
        3.  Use of `$` in variables inside the lambda parameters or not? Using
            `$` makes it consistent with other parameter declarations, but the
            `$` before the name is superfluous. (It is also superfluous in
            defines and parameterized classes and could be relaxed there). If
            you like `$()` as delimiters (option i.) instead of `||` then including
            the `$` in the list for each variable seems superfluous).

4.  Do you like the ruby-style of using enumeration functions (`each`,
    `collect`, `reject`, `reduce`) or do you prefer an operator based
    approach?

    1.  If you like operators, the "pipe" approach is what those involved in
        discussing alternatives have been able to make most concrete. We
        think you may like the pipes because "they use a pipe idiom that
        should be familiar to sysadmins", and "simple, typical use cases
        look simple". In contrast you may dislike them because as soon as
        complexity goes up there is exposure to computer science constructs
        that are just as hard (or even harder) to understand than the
        "ruby/java8"-like proposal. You may also dislike it because it comes
        with some special cases, and is sprinkled with magic variables. It
        may also seem like mixed idioms and is not like anything seen
        elsewhere.
        
    2.  If you like the ruby-java8-like approach better you may have issues
        with the names of the functions:

        1.  Should it be each (like in ruby), or foreach (as in java8, and some
            languages where this is a keyword). Does the `for` prefix help or
            confuse? Do you like java8's stream method name better?
        2.  Do you like the ruby select and reject, or would you prefer there is
            only a select (with user having to put a not first to get
            reject functionality). Do you prefer the more generic term
            filter (used in java8)?
            
    3.  If you like the ruby-java8-like approach better you may have issues
        with the "picking" from the enumerable:

        1.  Would you like to see the start of the pipe/chain of iteration be
            based on LHS/RHS combination (i.e selecting hash keys, key-value
            pairs, pairs, triplets, etc. based on parameter declaration in the
            lambda) e.g. that `$a.each |$x,$y| => {}` picks pairs, or ...
        2.  always use an additional function to perform the picking (e.g.
            `pairs`, `triplets`, `tuplets(n)`, `keys`, `values`) if picking is not "each
            element"? (Java-8 has a default stream method with possibility to
            add new such methods for various types of picking things).

5.  Do you like the ruby method call style (using dot notation) or not?

    1.  You may like the dot notation because it allows a chain of calls
        with lambdas that reads correctly from left to right.
    2.  You may dislike the function notation because the dot notation looks
        like invoking methods on "data", and you are not familiar with the
        concept of "objects".

6.  Do you like the idea that function call and method call are
    basically the same thing?

    1.  You may dislike it because it is two ways to do the same
    2.  You may like it because some functions may not have a LHS to apply a
        dot method lambda to, and hey the functions are unaware of how they
        were called.
    3.  You may like it because it enables being able to indirectly apply a
        function by name to data
    4.  You may be ok with having both, but would like to see restrictions
        on combination of function call with lambda to avoid users creating
        nested lambda calls and other constructs that are hard to read, but
        is fine with having straight forward function calls with lambdas.
