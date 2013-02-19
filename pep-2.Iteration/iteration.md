Iteration
=========

This document contains an exploration of options and a proposed solution
to the puppet redmine issue 
[#11331 - "foreach"](http://projects.puppetlabs.com/issues/11331)

Initially, this proposal for support of iteration was modeled after
Ruby's lambdas and Enumerables. Work then continued on an alternative
using "pipes". The "ruby like" proposal has been implemented, but not
(yet) the "pipe" based proposal.

Both the "ruby like" and the "pipe" proposal makes use of a
parameterized code block (a lambda), and this idea is presented
separately in this document as the other two build upon it.

At the end there is also mentions of other alternatives/suggestions that
were considered but not explored further.


Examples of the Recommended Implementation
==========================================

### Iterating over an array - creating resources

    $array.each |$x| { file { "/somewhere/$x": owner => $x } }

### Iterating over pairs in an array or hash

e.g. an array of `['usrname', 0777, ...]`, or hash of `{'username'=> 0777, ...}`

    $array.pairs |$x| {
      file {"/somewhere${$x[0]}":
        owner => $x[0],
        mode=>$x[1]
      }
    }

### Creating, Collecting, and Relating Resources

    $r1 = $a1.collect |$x| {file {"/somewhere/$x": owner => $x}}
    $r2 = $a2.collect |$x| {file {"/elsewhere/$x": owner => $x}}
    $r1 -> $r2

or

    $a1.collect |$x| { file {"/somewhere/$x": owner => $x}} ->
      $a2.collect |$x| { file {"/elsewhere/$x": owner => $x}}

### Filtering Data Before Creating a resource

    $a.select |$x| { $x =~ /com$/ }.each |$x| {
      file { "/somewhere/$x":
        owner => $x
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

    {|<parameters>| <statements or expression> }

This means that the block requires arguments to be passed to the block,
and that the passed arguments are bound to the local variables as
denoted by the parameters. Here is an example:

    {|$x,$y| ... } # lambda with two parameters    
    {|| ... }      # lambda with no parameters

Parameters before
-----------------

As an alternative to the lambda parameters being inside of the brace,
they can be moved to the left of the lambda body. (See discussion below
for the rationale to pick this syntax).

    |<parameters>| { <statements or expression> }

e.g.

    |$x| { $x * 2 }
    
    |$x| { file { "/somewhere/$x": ... } }

This also makes it possible to allow something else than a block
expression as the lambda body (e.g. a function, type or define
reference). Although tempting, having just a simple expression makes it
difficult to chain operations (the dot binds to the last operand). It is
probably best if the lambda body for general expressions are always in a
block to avoid users having to think about precedence since they will
typically get unhelpful errors if they get it wrong.

Consider:

    $a.collect |$x| $x + 2 . select ...

What is the select applied to? (A: the literal 2).

Function references
-------------------

It is useful to be able to directly pass a named function where a lambda
is allowed. The `&` operator can be used for this:

    &<NAME>

These two are then equivalent:

    $array.collect &upcase
    $array.collect |$x| { upcase($x) }

The proposal is to only allow a literal reference to a function at the
same locations as a lambda is allowed (i.e. to not support something
like `&$x` for a dynamic reference, and not accepting `&<NAME>` as an
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

    (x, y) -> { }

Using more puppet like syntax, this could be:

    ($x, $y) => { }    
    () => { }

This however means that calls where there are no parameters must use
parentheses, as a call is otherwise ambiguous. (This would be  a drastic
change to the puppet language).

A suggested solution to this is to use a syntactic marker such as `$`:

    $(x,y)=>{}    
    ${}

But `${}` clashes with old style "non double quoted interpolated string"
that was available in some versions of puppet. `$(x,y)` is preferred to
`$($x,$y)` simply because it is lighter, but then this is not
consistent with how parameters to classes and defines are expressed.
Also, taken in isolation a `${}` syntax may be confused with
interpolation in general.

Using some other free operator works:

    &(x,y) => {}    
    &{}

This style is used in the "pipe" proposal. because that proposal also
uses `&` as a function reference, and a lambda is an anonymous function
(although defined instead of referenced).

If variables should be preceded with `$` to be consistent with other
function like declarations it starts to look clunky:

    &($x,$y) => {}    
    &{}

Some languages (e.g. go) uses a keyword `func` to introduce an anonymous
function. We could do the same (but this adds a keyword), or we could
use the convention that the name `_` means anonymous. If used as an
operator, and dropping the fat arrow, a lambda would look like this:

    _($x,$y) {}
    _{}

That looks quite magical compared to the keyword variant:

    func($x,$y) {}    
    func($x,$y) {}
    func {}

It is possible to use the pipe operators (at least in the "Ruby like"
proposal):

    |$x, $y| => { }
    || => { }

and also works without the arrow:

    |$x, $y| { }    
    || { }

Further, it is of value to be able to write a lambda that is not a block
(such as a function reference). In this case the fat arrow separator may
look more appealing. Compare:

    |$x| => &upcase($x)
    |$x| &upcase($x)
    
    $array.each |$x| {file { "/tmp/$x": }}
    $array.each |$x| => {file { "/tmp/$x": }}

Naturally, both styles (without arrow for a braced block, and with arrow for a function reference) could be
used - maybe to set them apart, and to provider better error feedback (but this may be of debateable value).

### Automatic currying

You may have reacted examples like this:

    $array.collect |$x| => &upcase($x)

as it has redundant information, this could just as well have been
written

    $array.collect &upcase

Automatic currying with a lambda could look like this but needs the
magic variable 'it':

    $array.collect &{ upcase($_) }

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

There is already criticism against having too much special punctuation, and
the addition of greek letters the syntax starts to look like "math".

Discussion about Function Reference
-----------------------------------

As shown in the summary of the proposals for lambda, a function
reference could be created by using `&<NAME>. As there is no closure
requirement, a function reference can be allowed as an r-value, and thus
any expression can be used after the &  as long as it evaluates to the
name of a function. This means that the grammar could be:

    & <expression>

And that expression must evaluate to a string that is the name of the
function - ie. `&` is a function pointer.

This allows composition and functional oriented programming styles to be
used. This may however be far too complex for the target audience. There
are other obvious issues regarding static validation of such statements
as the correctness is not known until evaluation time.

The functional reference can allow specification of additional
arguments, i.e. an uncompleted function call, and the `it` variable
represents where the curried (value is inserted). A concrete binding for
`it` is needed; it could be spelled out as `$it`, or use a more special
`$_` syntax.

This example shows a call where the curried variable is in the second
position when the call is made.

    &myfunc(1, $_, "text")

Additional rules could be:

* if no arguments are specified, the call is made with the single 
  argument `$_`

* if arguments are specified, the `$_` is the first argument (unless
  it is included in the list)

or

* if arguments are specified, the `$_` must be included in order to
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

    $a.select |$x| {$x > 10}.collect |$x|{ $x*2 }

is easier to read than

    collect(select($a) |$x| { $x > 10}) |$x|{$x*2}

These two calling styles are referred to as "chained call style" and
"function call style".

This means that the proposal is not to add "iteration" per se to the
language, but to add the ability to write functions that may perform
iteration, or apply external logic to puppet constructs in a convenient
way.

Chained call style
------------------

    $a.each |$x| { ... }

Function call style
-------------------

    each($a) |$x| { ... }

Which naturally also works when there is no data e.g. getting data from
some default source, and it can directly apply the data to the lambda.

    defaultdata() |$x| { ... }

Which, if function call with lambda was not supported would have to be
written as:

    defaultdata().apply |$x| { ... }

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
it be better if operations were chained with a pipe `|` ?

One can simply exchange the dot operator with a `|` to get this horrible
mix of idioms:

    $a | collect {|$x| } | each {|$x| }

However, this is a good starting point for understanding the proposed
pipe notation (which is not the horrible line above).

In the pipe proposal,  the `|` operator means "collect", and the `|?` means
"select" (there is no reject, although one could be added with an
additional operator).  The LHS is always an array, if fed something
other than an array, it is turned into one. A magic variable referred to
as "it" (concrete syntax either `$it`, or something more special like
`$_`) is available in the RHS scope. This variable may be renamed by the
use of a lambda. In addition this proposal uses & as an operator to
produce a function reference (see separate discussion about Function
References).

A semicolon ends the pipe. This is needed when a pipe is used in
expressions and provides a visual clue where the pipes ends when the
last step in the pipe extends over multiple lines. (Although only
strictly required when the pipe is used as a value in an expressions it
is better if it is mandatory - the pipe ends here;).

Here is an example of upcasing elements of an array, the result is a new
array:

    $upcased = [a,b,c] | &upcase ;

produces `[A, B, C]` assigned to `$upcased`.

To summarize; arrays flow through the pipe element by element, applying
the element to a function that is expressed via function reference, as
an expression (which is shorthand for a lambda), or a lambda. The pipe
ends with a semicolon.

### Lambdas in the pipe

There are obvious difficulties to use the proposed lambda forms that are
based on using pipes to delimit the lambda parameters. Here are some
examples:

    f | || e  |   ... # ambiguous !
    f | {|| e } | ... # ok, ruby like lambdas
    f | &() e |   ... # ok, prefixed parameter list

The `&(){}` form is used going forward in the text (although the ruby like
syntax would work just as well).

### Collect pipe segment

A collect pipe segment allows one of the following as the RHS:

1.  A function reference `&func`, (or uncompleted function call) which is
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
behave like the UFO operators `<| |>` where bare word expressions are
turned into named access - i.e. that the following expressions have the
same meaning.

    |? attr == 3 ;    
    |? &(x) { $x['attr'] == 3 } ;
    |? &{ $_['attr'] == 3 } ;

The side effect is that bare words can not be used to represent strings,
they must always be quoted in a select segment. An alternative is to add
special select pipe segment (described in the next section).

### Examples using Pipe

Using pipes makes code compact.

    $a |? $_ > 10 | $_ * 2 ;

Which is equivalent to the proposed (more explicit) ruby/java8-like
alternative:

    $a.select |$x| {$x > 10}.collect |$x| { $x * 2 }

Pipe syntax can also use variable names (instead of `it`):

    $a |? &(x) $x > 10 | &(x) $x * 2 ;

When used to create resources:

    $a |? $_ =~ /com$/ |
      file { "/somewhere/$x":
        owner => $x
      } ;

### Ending the pipe

If there is the need to continue with a non pipe expression after the
pipe (say to allow addition of the produced array with another array,
there must be the need to terminate the array. A semicolon can be used
for this:

    expr | expr ; + expr

To make it simpler, all pipes should end with a semicolon.

### Special select pipe segment

It may be possible to use the UFO select to add this behavior to the
pipet. I.e. a segment written as `<| |>` would be a select with bare
word magic.

    <| attr == 3 |>

This comes with its own set of grammatical issues, and the operator `<|`
already has additional semantics.

### Reduction pipe segments

Since the operators are collect and select only, any reduction needs to
be performed by a function. The problem is that the semantics of the `|`
is collect - i.e. collecting the result of each reduction.

The solutions are:

1.  No support for reduction
2.  Modify the semantics; a function reference takes over the meaning of
    the operator. | is really an enumeration operator, and if the RHS is
    a function producing an enumerable this is used instead of collect.
    The introduction of such special cases is unfortunate.
3.  Introduce a reduce operator, say `|>`, but now two magic `it` variables
    are needed (this and that)
4.  Support reduction as a lambda function

Reduction as function call with lambda would then look like this:

    $array | reduce (1) &(memo, x) => {$memo + $x}

We now basically have both proposals at the same time (pipe and the ruby
like support) and the result is both hard to read and hard to
understand. Compare to:

    $array.reduce(1) |$memo, $x| {$memo + $x }

As you have observed, the pipe notation makes simple cases compact
(perhaps too compact as there are few clues to what is meant by the
operators), and complexity makes a funny face at us when we need to deal
with something like reduction.



Ideas for the future
--------------------

Ideally, the grammar should be relaxed to not differentiate between
expressions and statements, as this makes it possible to use constructs
such as:

    $a = case ...
    $a = if ...

Which are much nicer and clearer than embedding conditional assignments
to `$a` inside the nested blocks of these statements.

This is easily achievable with an expression based grammar; it is only a
matter of different validation to make the semantics match 3.1, and can
thus be turned on/off as an experimental feature.
