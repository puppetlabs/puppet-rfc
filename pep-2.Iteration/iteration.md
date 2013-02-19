"Iteration"
===========

This document contains an exploration of options and a proposed solution
to the puppet redmine issue \#11331 - "foreach"
[http://projects.puppetlabs.com/issues/11331](http://projects.puppetlabs.com/issues/11331)

Initially, this proposal for support of iteration was modeled after
Ruby's lambdas and Enumerables. Work then continued on an alternative
using "pipes". The "ruby like" proposal has been implemented, but not
(yet) the "pipe" based proposal.

Both the "ruby like" and the "pipe" proposal makes use of a
parameterized code block (a lambda), and this idea is presented
separately in this document as the other two build upon it.

At the end there is also mentions of other alternatives/suggestions that
were considered but not explored further.

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

1.  delimiters around parameters; i.e. |\$x| or something else?
2.  arrow or no arrow; i.e, |\$x| =\> {e}, or |\$x| {e}
3.  Use of \$ in variables inside the lambda parameters or not? Using
    \$ makes it consistent with other parameter declarations, but the
    \$ before the name is superfluous. (It is also superfluous in
    defines and parameterized classes and could be relaxed there). If
    you like \$() as delimiters (option i.) instead of || then including
    the \$ in the list for each variable seems superfluous).

4.  Do you like the ruby-style of using enumeration functions (each,
    collect, reject, reduce) or do you prefer an operator based
    approach?

4.  If you like operators, the "pipe" approach is what those involved in
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
5.  If you like the ruby-java8-like approach better you may have issues
    with the names of the functions:

1.  Should it be each (like in ruby), or foreach (as in java8, and some
    languages where this is a keyword). Does the "for" prefix help or
    confuse? Do you like java8's stream method name better?
2.  Do you like the ruby select and reject, or would you prefer there is
    only a select (with user having to put a not first to get
    reject functionality). Do you prefer the more generic term
    filter (used in java8)?

6.  If you like the ruby-java8-like approach better you may have issues
    with the "picking" from the enumerable:

1.  Would you like to see the start of the pipe/chain of iteration be
    based on LHS/RHS combination (i.e selecting hash keys, key-value
    pairs, pairs, triplets, etc. based on parameter declaration in the
    lambda) e.g. that \$a.each [\$x,\$y| =\> {} picks pairs, or ...
2.  always use an additional function to perform the picking (e.g.
    pairs, triplets, tuplets(n), keys, values) if picking is not "each
    element"? (Java-8 has a default stream method with possibility to
    add new such methods for various types of picking things).

5.  Do you like the ruby method call style (using dot notation) or not?

7.  You may like the dot notation because it allows a chain of calls
    with lambdas that reads correctly from left to right.
8.  You may dislike the function notation because the dot notation looks
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

Recommendation
--------------

The author of this proposal recommends that:

1.  The ruby-java8 like proposal is used with:

1.  lambda parameters expressed with |\$x|
2.  lambda parameters placed left of the lambda body i.e. |\$x| {e}
3.  the lambda body may be a block with one or several
    expressions/statements

1.  Wait with implementation of these options until use-cases are more
    known:

1.  function reference and uncompleted call (specifying arguments that
    are not curried from the enumerable)
2.  type/define reference (i.e. a call-by-position to call-by-name
    mapping)

4.  enumeration functions each, select, reject, collect and reduce are
    implemented.
5.  enumeration picking is each element (i.e. [key,value] for hash, and
    each element for array), and that any other picking is done with
    functions such as keys(), values() for hashes, and pairs(),
    triplets(), and tuples(n) for arrays.
6.  It is always a single argument that is curried (it may be an array)
7.  Literal arguments appear after the curried value by convention, but
    a function (e.g. reduce) may insert a value before the first when
    this is logically more sound than having it appear after the curry
    (in reduce the optionally given argument is the "start value").
8.  Calls to functions always pass the optional lambda as the last
    argument.

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

Examples of the Recommended Implementation
==========================================

### Iterating over an array - creating resources

\$array.each |\$x| { file { "/somewhere/\$x": owner =\> \$x } }

### Iterating over pairs in an array or hash

e.g. an array of ['usrname', 0777, …], or hash of {'username'=\> 0777,
…}

\$array.pairs |\$x| {

  file {"/somewhere/\${\$x[0]}":

    owner =\> \$x[0],

    mode=\>\$x[1]

  }

}

### Creating, Collecting, and Relating Resources

\$r1 = \$a1.collect |\$x| {file {"/somewhere/\$x": owner =\> \$x}}

\$r2 = \$a2.collect |\$x| {file {"/elsewhere/\$x": owner =\> \$x}}

\$r1 -\> \$r2

or

\$a1.collect |\$x| { file {"/somewhere/\$x": owner =\> \$x}} -\>

  \$a2.collect |\$x| { file {"/elsewhere/\$x": owner =\> \$x}}

### Filtering Data Before Creating a resource

\$a.select |\$x| { \$x =\~ /com\$/ }.each |\$x| {

  file { "/somewhere/\$x":

    owner =\> \$x

  }

}

Lambdas
=======

A lambda is an (optionally) parameterized code block that is evaluated
in a local inner scope of the point where the lambda is being used. It
is possible to set variables in this scope, but these are only visible
in the lambda's scope (and any nested inner local scopes). These
variables, just like all other variables are immutable. When the lambda
is invoked many times (as is the case with iteration), the block is
re-initialized, any variable values set in the previous iteration are
gone. Side effects of lambdas (such as creating a resource, or anything
else that ends up in the catalog) takes place in evaluation order.

Parameters inside
-----------------

The "ruby like" proposal is based on this syntax:

{|\<parameters\>| \<statements or expression\> }

This means that the block requires arguments to be passed to the block,
and that the passed arguments are bound to the local variables as
denoted by the parameters. Here is an example:

{|\$x,\$y| ... } \# lambda with two parameters

{|| ... }      \# lambda with no parameters

Parameters before
-----------------

As an alternative to the lambda parameters being inside of the brace,
they can be moved to the left of the lambda body. (See discussion below
for the rationale to pick this syntax).

|\<parameters\>| { \<statements or expression\> }

e.g.

|\$x| { \$x \* 2 }

|\$x| { file { "/somewhere/\$x": … } }

This also makes it possible to allow something else than a block
expression as the lambda body (e.g. a function, type or define
reference). Although tempting, having just a simple expression makes it
difficult to chain operations (the dot binds to the last operand). It is
probably best if the lambda body for general expressions are always in a
block to avoid users having to think about precedence since they will
typically get unhelpful errors if they get it wrong.

Consider:

\$a.collect |\$x| \$x + 2 . select …

What is the select applied to? (A: the literal 2).

Function references
-------------------

It is useful to be able to directly pass a named function where a lambda
is allowed. The & operator can be used for this:

&\<NAME\>

These two are then equivalent:

\$array.collect &upcase

\$array.collect |\$x| { upcase(\$x) }

The proposal is to only allow a literal reference to a function at the
same locations as a lambda is allowed (i.e. to not support something
like &\$x for a dynamic reference, and not accepting &\<NAME\> as an
r-value) - see "Discussion about Function Reference" below.

The addition of a function reference is optional, it is not required to
support "iteration".

Discussion about Lambda Syntax
------------------------------

When the Ruby like syntax was shown to non-programmers, there seems to
be an unnecessary cognitive complication due to the placement of the
parameters inside the braces - as this is different from say a regular
function declaration which people seems to understand.

Moving the lambda parameters outside of the body is somewhat
grammatically difficult (as we will see in a bit). In Java 8, lambdas
are written like this:

(x, y) -\> { }

Using more puppet like syntax, this could be:

(\$x, \$y) =\> { }

() =\> { }

This however means that calls where there are no parameters must use
parentheses, as a call is otherwise ambiguous. (This would be  a drastic
change to the puppet language).

A suggested solution to this is to use a syntactic marker such as \$:

\$(x,y)=\>{}

\${}

But \${} clashes with old style "non double quoted interpolated string"
that was available in some versions of puppet. \$(x,y) is preferred to
\$(\$x,\$y) simply because it is lighter, but then this is not
consistent with how parameters to classes and defines are expressed.
Also, taken in isolation a \${} syntax may be confused with
interpolation in general.

Using some other free operator works:

&(x,y) =\> {}

&{}

This style is used in the "pipe" proposal. because that proposal also
uses & as a function reference, and a lambda is an anonymous function
(although defined instead of referenced).

If variables should be preceded with \$ to be consistent with other
function like declarations it starts to look clunky:

&(\$x,\$y) =\> {}

&{}

Some languages (e.g. go) uses a keyword "func" to introduce an anonymous
function. We could do the same (but this adds a keyword), or we could
use the convention that the name "\_" means anonymous. If used as an
operator, and dropping the fat arrow, a lambda would look like this:

\_(\$x,\$y) {}

\_{}

That looks quite magical compared to the keyword variant:

func(\$x,\$y) {}

func(\$x,\$y) {}

func {}

It is possible to use the pipe operators (at least in the "Ruby like"
proposal):

|\$x, \$y| =\> { }

|| =\> { }

and also works without the arrow:

|\$x, \$y| { }

|| { }

Further, it is of value to be able to write a lambda that is not a block
(such as a function reference). In this case the fat arrow separator may
look more appealing. Compare:

|\$x| =\> &upcase(\$x)

|\$x| &upcase(\$x)

\$array.each |\$x| {file { "/tmp/\$x": }}

\$array.each |\$x| =\> {file { "/tmp/\$x": }}

### Automatic currying

You may have reacted examples like this:

\$array.collect |\$x| =\> &upcase(\$x)

as it has redundant information, this could just as well have been
written

\$array.collect &upcase

Automatic currying with a lambda could look like this but needs the
magic variable 'it':

\$array.collect &{ upcase(\$\_) }

Since the introduction of lambda and iteration will be new to the puppet
community, it is perhaps best to always be explicit as things are
already complex and magic enough without also adding automatic currying.

### UTF-8 options

It has been suggested in several conversations about operators and
lambdas that it would be of value to use operators in the wider UTF-8
character set. An obvious choice for lambda is then the greek lower case
lambda letter λ

λ(x,y) {}

λ{}

Irrespective of the merits and problems of using the lambda character,
using UTF-8 characters has several inherent problems; they are much more
difficult to type, and there may be encoding issues in users toolchains
that makes them drop or misinterpret such characters.

Discussion about Function Reference
-----------------------------------

As shown in the summary of the proposals for lambda, a function
reference could be created by using &\<NAME\>. As there is no closure
requirement, a function reference can be allowed as an r-value, and thus
any expression can be used after the &  as long as it evaluates to the
name of a function. This means that the grammar could be:

& \<expression\>

And that expression must evaluate to a string that is the name of the
function - ie. & is a function pointer.

This allows composition and functional oriented programming styles to be
used. This may however be far too complex for the target audience. There
are other obvious issues regarding static validation of such statements
as the correctness is not known until evaluation time.

The functional reference can allow specification of additional
arguments, i.e. an uncompleted function call, and the 'it' variable
represents where the curried (value is inserted). A concrete binding for
'it' is needed; it could be spelled out as \$it, or use a more special
\$\_ syntax.

This example shows a call where the curried variable is in the second
position when the call is made.

&myfunc(1, \$\_, "text")

Additional rules could be:

1.  if no arguments are specified, the call is made with the single
    argument \$\_
2.  if arguments are specified, the \$\_ is the first argument (unless
    it is included in the list)
3.  (or)
4.  if arguments are specified, the \$\_ must be included in order to
    pass it (this is less magical but also adds noise).

The addition of uncompleted calls is optional.

Closures
--------

It is proposed that lambdas are (at least initially) restricted to be
used in association with calls to functions/the runtime and not be
allowed as r-values as this requires a rewrite of scope, with potential
breakage as a consequence.

Since there are difficulties to support full closures (i.e. that lambda
is a r-value) they may only appear as an extra element in function
calls. It is the responsibility of the implementor of a function to
ensure that the scope given in the call is not referenced after the
function returns.

Ruby Like Proposal
==================

The proposal is based on the implementation of one general mechanism;
the ability to define a parameterized puppet code block (a.k.a Lambda),
and passing that to a function call. In addition to this it is
beneficial to also make calls chainable from left to right rather than
nesting them. While not all that difficult to read for regular function
calls, it becomes a real usability issue when the nesting includes
nested lambdas.

e.g.

\$a.select |\$x| {\$x \> 10}.collect |\$x|{ \$x\*2 }

is easier to read than

collect(select(\$a) |\$x| { \$x \> 10}) |\$x| {\$x\*2}

These two calling styles are referred to as "chained call style" and
"function call style".

This means that the proposal is not to add "iteration" per se to the
language, but to add the ability to write functions that may perform
iteration, or apply external logic to puppet constructs in a convenient
way.

Chained call style
------------------

\$a.each |\$x| { … }

Function call style
-------------------

each(\$a) |\$x| { … }

Which naturally also works when there is no data e.g. getting data from
some default source, and it can directly apply the data to the lambda.

        defaultdata() |\$x| { … }

Which, if function call with lambda was not supported would have to be
written as:

        defaultdata().apply |\$x| { … }

Which has the negative side effect that the defaultdata() function on
its own can not make use of efficient enumeration and must always return
an array rather than feeding "next" into the given lambda.

Pipe Expression Proposal (updated Feb 15, 2013)
===============================================

The proposal to use "pipes" is thought as an alternative to the
ruby/java8 proposal, and is the proposal that those working on ideas
around iteration managed to make most concrete while staying within the
constraints (no new keywords, etc.).

The rationale for this proposal is that it is believed that sysadmins
are used to thinking in terms of a unix command line pipe. Essentially
that is what a combination of dot operation and lambdas provide. Would
it be better if operations were chained with a pipe '|' ?

One can simply exchange the dot operator with a '|' to get this horrible
mix of idioms:

\$a | collect {|\$x| } | each {|\$x| }

However, this is a good starting point for understanding the proposed
pipe notation (which is not the horrible line above).

In the pipe proposal,  the | operator means "collect", and the |? means
"select" (there is no reject, although one could be added with an
additional operator).  The LHS is always an array, if fed something
other than an array, it is turned into one. A magic variable referred to
as "it" (concrete syntax either \$it, or something more special like
\$\_) is available in the RHS scope. This variable may be renamed by the
use of a lambda. In addition this proposal uses & as an operator to
produce a function reference (see separate discussion about Function
References).

A semicolon ends the pipe. This is needed when a pipe is used in
expressions and provides a visual clue where the pipes ends when the
last step in the pipe extends over multiple lines. (Although only
strictly required when the pipe is used as  a value in an expressions it
is better if it is mandatory - the pipe ends here;).

Here is an example of upcasing elements of an array, the result is a new
array:

\$upcased = [a,b,c] | &upcase ;

produces [A, B, C] assigned to \$upcased.

To summarize; arrays flow through the pipe element by element, applying
the element to a function that is expressed via function reference, as
an expression (which is shorthand for a lambda), or a lambda. The pipe
ends with a semicolon.

### Lambdas in the pipe

There are obvious difficulties to use the proposed lambda forms that are
based on using pipes to delimit the lambda parameters. Here are some
examples:

f | || e  | … \# ambiguous !

f | {|| e } | … \# ok, ruby like lambdas

f | &() e | … \# ok, prefixed parameter list

The &(){} form is used going forward in the text (although the ruby like
syntax would work just as well).

### Collect pipe segment

A collect pipe segment allows one of the following as the RHS:

1.  A function reference &func, (or uncompleted function call) which is
    called for each element, the result is collected and passed to the
    next segment in the pipe.
2.  A lambda, which is called for each element, the result is collected
    and passed to the next segment in the pipe
3.  Any other expression is evaluated and collected. The expression may
    reference the it variable. (i.e. this is an automatic lambda).

### Select pipe segment

The select pipe segment allows the same RHS expressions as collect, with
the following additions:

1.  The expression must be a boolean result
2.  The collected result passed to the next segment in the pipe consists
    of all elements for which the expression evaluated to true.

More magic can perhaps be added to the select pipe segment to make it
behave like the UFO operators \<| |\> where bare word expressions are
turned into named access - i.e. that the following expressions have the
same meaning.

|? attr == 3 ;

|? &(x) { \$x['attr'] == 3 } ;

|? &{ \$\_['attr'] == 3 } ;

The side effect is that bare words can not be used to represent strings,
they must always be quoted in a select segment. An alternative is to add
special select pipe segment (described in the next section).

### Examples using Pipe

Using pipes makes code compact.

\$a |? \$\_ \> 10 | \$\_ \* 2 ;

Which is equivalent to the proposed (more explicit) ruby/java8-like
alternative:

        \$a.select |\$x| {\$x \> 10}.collect |\$x| { \$x \* 2 }

Pipe syntax can also use variable names (intead of it):

\$a |? &(x) \$x \> 10 | &(x) \$x \* 2 ;

When used to create resources:

\$a |? \$\_ =\~ /com\$/ |

  file { "/somewhere/\$x":

    owner =\> \$x

  } ;

### Ending the pipe

If there is the need to continue with a non pipe expression after the
pipe (say to allow addition of the produced array with another array,
there must be the need to terminate the array. A semicolon can be used
for this:

expr | expr ; + expr

### Special select pipe segment

It may be possible to use the UFO select to add this behavior to the
pipet. I.e. a segment written as \<| |\> would be a select with bare
word magic.

\<| attr == 3 |\>

This comes with its own set of grammatical issues, and the operator \<|
already has additional semantics.

### Reduction pipe segments

Since the operators are collect and select only, any reduction needs to
be performed by a function. The problem is that the semantics of the |
is collect - i.e. collecting the result of each reduction.

The solutions are:

1.  No support for reduction
2.  Modify the semantics; a function reference takes over the meaning of
    the operator. | is really an enumeration operator, and if the RHS is
    a function producing an enumerable this is used instead of collect.
    The introduction of such special cases is unfortunate.
3.  Introduce a reduce operator, say |\>, but now two magic it variables
    are needed (this and that)
4.  Support reduction as a lambda function

Reduction as function call with lambda would then look like this:

\$array | &reduce (1) &(memo, x) =\> {\$memo + \$x}

We now basically have both proposals at the same time (pipe and the ruby
like support) and the result is both hard to read and hard to
understand. Compare to:

\$array.reduce(1) |\$memo, \$x| {\$memo + \$x }

As you have observed, the pipe notation makes simple cases compact
(perhaps too compact as there are few clues to what is meant by the
operators), and complexity makes a funny face at us when we need to deal
with something like reduction.

Other Considered Alternatives^[[a]](#cmnt1)^ and Options
========================================================

Function call style with lambda as parameter
--------------------------------------------

foreach(\$a, {|\$x| … } )

This is not as nice textually (a block nested in the argument list).

Requires either handling the lambda as a special case (grammatically
only appearing last in an argument list), or handle as non evaluated
expression. If not handled as a special case in the grammar, the
consequence is that code like this can be written:

\$a = {|\$x| … }

\#... much later

foreach(\$y, \$a)

The big problem here is that this requires closures (lambda bound in
other context than where it is called).

Most alternatives require introduction of new keyword(s)
--------------------------------------------------------

Anything that introduces new keywords may require users to modify their
existing code.

If "foreach" is made into a keyword there are much fewer restrictions on
what the syntax would look like.

Naturally, a specific "foreach" implementation would not be
open/extensible and thus does not provide support for things like
collect, select, reject, and reduce.

It would be possible to use an already existing keyword or operator, but
using it a different way. This is usually more magic. (Not explored
further).

Using magic variables
---------------------

(requires foreach to be a keyword)

foreach \$a {

  \$loop

}

This requires that foreach is a keyword (it would otherwise be a
function with \$a as a parameter) followed by a block. This would then
require that a function call can be followed by a block, and that magic
variables are introduced in such a block. Basically it is the same as a
lambda only that variables are not declared. Generalized to other
functions than foreach, it would be an ambiguity in the grammar for

NAME { … }

as this is either a function call with a hash as a parameter, or a
function call with a block. Extensive grammar lookahead is required if
the start of the block is not easily recognized with lexical lookahead.
(In the proposal that is "{ |")

No matter what is picked (\$loop, \$\_ etc), the magic variable is
magic. This style also makes it more difficult to specify what is wanted
on each iteration, one value at a time or two or three, the keys from a
hash, the values, or the entries? If only one at a time is supported, a
'next' operation is required to be able to skip iterations, and if a
pair is required the intermediate state needs to be kept outside of the
inner block. (Lots of problems).

Other forms of declaration of loop variable than using || inside block
----------------------------------------------------------------------

(requires foreach to be a keyword).

foreach \$a (\$x) {}

foreach(\$a, \$x) { }

foreach \$a=\>\$x { }

foreach \$a =\> \$x, \$y, \$z { }

Can imagine saving typing by auto-shadowing the lhs if it is a variable
(but this naturally fails if lhs is a general expression). Anyway, here
is an example:

\$a = [1, 2, 3]

foreach \$a { notice \$a } \# =\> notices '1'

Disadvantages:

1.  new keyword
2.  auto-shadowing is just as magic as magic loop variables

Non general code block
----------------------

The idea is that looping in itself is not the goal, but the application
of a set onto something (e.g. a resource type). ie. essentially an
"apply" operator.

apply(a\_collection, to\_named\_thing)

Problem here is what to pick from a\_collection (each, pairs, triplets,
keys, entries), and how to apply them to the named thing. If the named
thing is a function, one could pick as many things from a collection as
there are arguments in the named function (to\_named\_thing). If however
the named thing is a resource type, arguments are passed by name, so
some kind of parameterized block is required.

so - either a keyword like apply, or an operator.

Using keyword apply
-------------------

apply \$a (\$x, \$y) =\> file { "prefix\_\$x": ensure =\> \$y}

Keyword apply can be extended with name - e.g.

        apply each \$a(\$x, \$y) =\> file { "prefix\_\$x": ensure =\>
\$y}

The name could be a symbolic mapping mode in english, e.g. each, pairs,
triplets, keys, key\_and\_value, etc. if a more human readable form is
wanted. Examples:

        apply each(\$x) \$a =\> file { "prefix\_\$x": ensure =\>
present}

apply pairs(\$x,\$y) \$a =\> file { "prefix\_\$x": ensure =\> \$y}

or some combination of this and other proposals

apply \$a(\$x) =\> file { "prefix\_\$x": ensure =\> present}

apply \$a.each(\$x) =\> file { "prefix\_\$x": ensure =\> present}

apply \$a.pairs(\$x,\$y) =\> file { "prefix\_\$x": ensure =\> \$y}

The advantage here is that it is explicit how the user wants to pick
things from the collection.

Still, if the iteration is wanted more than once, it has to be repeated,
or a RHS block is required. With a RHS block, this \*is\* the same as
the lambda proposal earlier, only with explicit naming of the picking
method. (The same could be achieved with different functions foreach,
foreach\_key, etc. and using the prefered/proposed solution).

### Operator example

Example uses \* overloading, and needs either magic variables, or a
declaration of the mapping

\$a \* file { "prefix\_\$a": ensure =\> present }

\$a \* |\$x| file { "prefix\_\$x": ensure =\> present }

The only difference from earlier general foreach function and lambda is
that logic is restricted in what it may apply to. The operator way is
quite magic.

Adapting Call by Name(updated Feb 15, 2013^[[b]](#cmnt2)^)
----------------------------------------------------------

A define (or a resource type) is almost like a function, but it is
"called" with named argument passing instead of positional argument
passing.

A special form of lambda can act as an adapter between these two call
styles. A lambda declared with a RHS being a type/define reference would
adapt the positional input to the output. Syntax:

|\<parameters\>| &\<NAME\>

Here is an example, calling the user defined resource type something
with this syntax, and with an equivalent full syntax:

\$a.each |\$name| &something

\$a.each {|\$name| something { name =\> \$name } }

If additional arguments are needed, they are passed as "default values"
e.g. these are equivalent

\$a.each |\$name, \$foo=10| &something

\$a.each {|\$name| something { name =\> \$name, foo =\> 10 } }

Since the RHS of default values are evaluated, it is possible to bind
the 'it' variable to any other variable. This opens up a set of issues
regarding the position to name mapping. The ruby like approach is that
the combination of LHS and RHS lambda determines what is fed into the
lambda (function introspects the RHS parameters), how should that work
when defaults are used, and there is a desire to define a mapping?

1.  Strict positioning; parameters with default are always last in the
    parameter list, parameters at the beginning receive values from the
    LHS function (this is how this is implemented in the experimental
    proposal)
2.  User may use a magic 'it' variable (but several may be needed and
    this starts to get complex, \$\_\_ ? or \$\_1, \$\_2, or \$\_[1],
    etc.)
3.  If the referenced element is call by position (i.e. a function), the
    arguments are simply passed in the order they are stated.

This is probably not useful in practice as each element needs a unique
title that is derived from the input, and the additional complexity of
"yet more syntax" means this is probably not worth exploring.

Automatic expansion of Collections^[[c]](#cmnt3)^
-------------------------------------------------

Idea being that enumeration happens as a consequence of a special
assignment - e.g.

\$each = \<expression\>; file { path =\> \$each }

This is similar to the "apply" idea; apply a collection to a resource
but it allows the specification of the name of the single value that can
be picked.

This style is not extensible, it can not be used for different types of
enumeration. It can be expanded to allow picking multiple variables
(with same type of semantics as introspection of the lambda parameters)
- e.g.

\$key, \$value = \<expression\> ; …

This is more magical, not extensible, and only solves "looping for the
purpose of applying result to a resource".

After updating the "pipe" and "using type as a function" above, I think
the gist of what this proposal is about is achieved while also being
extensible...

An Implementation of the Proposal
=================================

The implementation of the proposal implements new AST object to capture
the concepts: Named Access, Method Call, and Lambda.

A NamedAccess is EXPR . NAME

A MethodCall is a call applied to NamedAccess taking arguments

A Lambda is a parameterized code block that defines a local scope

The implementation only uses named access for method calls, but could be
used to lookup an object that may or may not be invokable. I.e.
foo.bar is now always a method call, not "just a lookup".

The existing "ephemeral" scope was expanded to also support being a
"local scope". This means that variable assignment always take place in
the most nested "local scope" or if none exists in the scope itself. The
effect is that variables shadow outer variables while being available to
inner local/match scopes. Access to any ephemeral variables (local or
match) is prevented as all external access is via qualified variables,
or when the scope is in a state where ephemeral scopes can not be
present (their scopes have already vanished), there is also protection
in the scope implementation where the :origin option of the request is
checked, and ephemeral lookup is then not done.

One wart in the 3.1.x based implementation is the requirement to prefix
an expression in the lambda with a syntax marker. In the implementation
"=" was choosen.

i.e.

\$a.select {|\$x| = \$x =\~ /awsome/ }

One remaining problem is that STATEMENT |  EXPRESSION is ambiguous in
puppet's current grammar (most likely because of the non-parenthesised
function call allowed for statements), and thus, to be able to have the
lambda return a value an expression needs to be turned into a statement
- i.e. a "return".

This is why the "=" is needed in the first 3.1 based implementation.

UPDATE:

As a test, a grammar for puppet that changed it to be expression based
was implemented, and that proved to be very doable by simply using a
different approach than in the current grammar.

This experiment (egrammar.ra) is currently found here:
https://github.com/hlindberg/puppet/blob/pops/lib/puppet/pops/impl/parser/egrammar.ra
for anyone interested. This work can be manually applied to the real
grammar.

To enable use of this approach, it is important to add validation of the
parsed result. That work is mostly what remains (most (if not all) rules
are easily found in Geppetto btw, so there is a list to work from.

The egrammar now as support for the following different styles

function λ

function () λ

x.function λ

x.function() λ

Where λ is one of

{| PARAMETERS | STATEMENTS\_OR\_EXPR }

| PARAMETERS| { STATEMENTS\_OR\_EXPR }

| PARAMETERS| =\> { STATEMENTS\_OR\_EXPR }

The intent for supporting all of these is to allow UX studies with one
and the same version.

Ideas for the future
--------------------

Ideally, the grammar should be relaxed to not differentiate between
expressions and statements, as this makes it possible to use constructs
such as:

\$a = case …

\$a = if …

Which are much nicer and clearer than embedding conditional assignments
to \$a inside the nested blocks of these statements.

This is easily achievable with an expression based grammar; it is only a
matter of different validation to make the semantics match 3.1, and can
thus be turned on/off as an experimental feature.

[[a]](#cmnt_ref1)Luke Kanies:

I'd like to at least think about whether we could do an approach that
didn't require full functionality in the attached block.

E.g., I know that arrays could be done with string expansion, but could
hashes?  Could we get the other functions out of it?

* * * * *

Henrik Lindberg:

I am not envisioning full functionality either; i.e. classes and define
not allowed in block which leaves resources and expressions. Also
probably not allowed to collect (that is instad another form of
iteration/predicate - only an idea at this stage).

* * * * *

Luke Kanies:

Ah.  I assumed that block was full code.  If it can only hold resources,
that changes it quite a bit, I think.  That is, if we're going for that
reduced use case, I think we can get there with less features added to
the language.

Any chance we could have a phone call about this or something?  Maybe we
discuss it at the Wednesday morning (PDX time) meeting?

* * * * *

Henrik Lindberg:

Andy and I are speaking at a Ruby meetup (wednesday) and then PuppetCamp
Stockholm (Thurs). Friday is first opportunity.

I understand what you are after (or at least I think so :) - supporting
the bare minimum first, but without getting painted into a corner wrt
other possibilities. Must be backwards compatible, and should use syntax
palatable by real/typical users. There is one proposal that is
implemented, and the "pipe" + "type" looks like it is also implementable
(but requires a few more iterations before completed). Then we have two
proposals which both can be reduced to a minimal case, while also being
extensible in the future.

* * * * *

Henrik Lindberg:

regarding "full code" - allowing classes to be created (and some other
types of statements would be very bad). The way to achieve this is to
assert what was parsed and validate the types of statements/expressions.
(This makes it flexible, and easy to experiment with going forward).

* * * * *

Luke Kanies:

I think this is not good.  I think the blocks either need to support the
full scope of the language, or they need to be a new syntax that makes
it clear that they're not full language.

That's why I like the string expansion version - it's obviously limited.

* * * * *

Henrik Lindberg:

This thread took place some time ago, and the document is now reworked
with a lot more detail. However - The blocks are "full language", but
just as for other types of conditional logic, it should not be allowed
to create classes or defines. There is no technical reason to limit
other types of expressions/statements.

Suggest that this particular discussion thread is resolved and that we
start over with a full review. ok?

* * * * *

Luke Kanies:

Yeah, I feel a bit lost in these docs.  Starting with a clean slate
seems reasonable.

[[b]](#cmnt_ref2)Henrik Lindberg:

added "using type as a function"

* * * * *

Henrik Lindberg:

Updated to use function reference style

[[c]](#cmnt_ref3)Luke Kanies:

Can I see some examples of this?  I don't understand.

* * * * *

Henrik Lindberg:

This came from your comment in the other document; maybe I misunderstood
the intent...

* * * * *

Luke Kanies:

Ah, ok.  Thanks.
