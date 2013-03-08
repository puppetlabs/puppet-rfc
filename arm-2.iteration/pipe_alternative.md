(ARM-2) Alternative - Pipe Expression Proposal
==============================================

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
