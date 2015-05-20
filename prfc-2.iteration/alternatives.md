Other Considered Alternatives and Options
=========================================
This document contains considered alternatives for the ARM-2.Iteration proposal.

It is based both on work done by the authors of this proposal and ideas discussed in various
forums. If these ideas seems half-baked, it is probably because that is precisely what they are.

Function call style with lambda as parameter
--------------------------------------------

    foreach($a, {|$x| ... } )

This is not as nice textually (a block nested in the argument list).

Requires either handling the lambda as a special case (grammatically
only appearing last in an argument list), or handle as non evaluated
expression. If not handled as a special case in the grammar, the
consequence is that code like this can be written:

    $a = {|$x| … }
    #... much later
    foreach($y, $a)

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

    foreach $a {    
      $loop
    }

This requires that foreach is a keyword (it would otherwise be a
function with \$a as a parameter) followed by a block. This would then
require that a function call can be followed by a block, and that magic
variables are introduced in such a block. Basically it is the same as a
lambda only that variables are not declared. Generalized to other
functions than foreach, it would be an ambiguity in the grammar for

    NAME { ... }

as this is either a function call with a hash as a parameter, or a
function call with a block. Extensive grammar lookahead is required if
the start of the block is not easily recognized with lexical lookahead.
(In the proposal that is `{|`)

No matter what is picked (`$loop`, `$_` etc), the magic variable remains
magic. This style also makes it more difficult to specify what is wanted
on each iteration, one value at a time or two or three, the keys from a
hash, the values, or the entries? If only one at a time is supported, a
'next' operation is required to be able to skip iterations, and if a
pair is required the intermediate state needs to be kept outside of the
inner block. (Lots of problems).

Other forms of declaration of loop variable than using || inside block
----------------------------------------------------------------------

(requires foreach to be a keyword).

    foreach $a ($x) {}
    foreach($a, $x) { }
    foreach $a=>$x { }
    foreach $a => $x, $y, $z { }

Can imagine saving typing by auto-shadowing the lhs if it is a variable
(but this naturally fails if lhs is a general expression). Anyway, here
is an example:

    $a = [1, 2, 3]
    foreach $a { notice $a } # => notices '1'

Disadvantages:

1.  new keyword
2.  auto-shadowing is just as magic as magic loop variables

Non general code block
----------------------

The idea is that looping in itself is not the goal, but the application
of a set onto something (e.g. a resource type). ie. essentially an
"apply" operator.

    apply(a_collection, to_named_thing)

Problem here is what to pick from a_collection (each, pairs, triplets,
keys, entries), and how to apply them to the named thing. If the named
thing is a function, one could pick as many things from a collection as
there are arguments in the named function (to_named_thing). If however
the named thing is a resource type, arguments are passed by name, so
some kind of parameterized block is required.

so - either a keyword like apply, or an operator.

Using keyword apply
-------------------

    apply $a ($x, $y) => file { "prefix_$x": ensure => $y}

Keyword apply can be extended with name - e.g.

    apply each $a($x, $y) => file { "prefix_$x": ensure => $y}

The name could be a symbolic mapping mode in english, e.g. each, pairs,
triplets, keys, key_and_value, etc. if a more human readable form is
wanted. Examples:

    apply each($x) $a => file { "prefix_$x": ensure => present}
    apply pairs($x,$y) $a => file { "prefix_$x": ensure => $y}

or some combination of this and other proposals

    apply $a($x) => file { "prefix_$x": ensure => present}
    apply $a.each($x) => file { "prefix_$x": ensure => present}
    apply $a.pairs($x,$y) => file { "prefix_$x": ensure => $y}

The advantage here is that it is explicit how the user wants to pick
things from the collection.

Still, if the iteration is wanted more than once, it has to be repeated,
or a RHS block is required. With a RHS block, this *is* the same as
the lambda proposal earlier, only with explicit naming of the picking
method. (The same could be achieved with different functions foreach,
foreach_key, etc. and using the prefered/proposed solution).

### Operator example

Example uses `*` overloading, and needs either magic variables, or a
declaration of the mapping

    $a * file { "prefix_$a": ensure => present }    
    $a * |$x| file { "prefix_$x": ensure => present }

The only difference from earlier general foreach function and lambda is
that logic is restricted in what it may apply to. The operator way is
quite magic.

Adapting Call by Name(updated Feb 15, 2013
------------------------------------------

A define (or a resource type) is almost like a function, but it is
"called" with named argument passing instead of positional argument
passing.

A special form of lambda can act as an adapter between these two call
styles. A lambda declared with a RHS being a type/define reference would
adapt the positional input to the output. Syntax:

    |<parameters>| &<NAME>

Here is an example, calling the user defined resource type something
with this syntax, and with an equivalent full syntax:

    $a.each |$name| &something
    $a.each {|$name| something { name => $name } }

If additional arguments are needed, they are passed as "default values"
e.g. these are equivalent

    $a.each |$name, $foo=10| &something
    $a.each {|$name| something { name => $name, foo => 10 } }

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
    this starts to get complex, `$__`, or `$_1`, `$_2`, or `$_[1]`,
    etc.)
3.  If the referenced element is call by position (i.e. a function), the
    arguments are simply passed in the order they are stated.

This is probably not useful in practice as each element needs a unique
title that is derived from the input, and the additional complexity of
"yet more syntax" means this is probably not worth exploring.

Automatic Expansion of Collections
----------------------------------

Idea being that enumeration happens as a consequence of a special
assignment - e.g.

    $each = <expression>; file { path => $each }

This is similar to the "apply" idea; apply a collection to a resource
but it allows the specification of the name of the single value that can
be picked.

This style is not extensible, it can not be used for different types of
enumeration. It can be expanded to allow picking multiple variables
(with same type of semantics as introspection of the lambda
parameters) - e.g.

    $key, $value = <expression> ; ...

This is more magical, not extensible, and only solves "looping for the
purpose of applying result to a resource".
