An Implementation of the Proposal
=================================

The implementation of the proposal implements new AST object to capture
the concepts: Method Call, Lambda, and BlockExpression

A MethodCall is a call applied to EXPR . NAME taking arguments

An AST:: Lambda is a parameterized AST::BlockExpression that defines a local scope

Local Scope
-----------
The existing "ephemeral" scope was expanded to also support being a
"local scope". This means that variable assignment always take place in
the most nested "local scope" or if none exists in the scope itself. The
effect is that variables shadow outer variables while being available to
inner local/match scopes. Access to any "ephemeral" variables (local or
match) is prevented as all external access is via qualified variables,
or when the scope is in a state where ephemeral scopes can not be
present (their scopes have already vanished), there is also protection
in the scope implementation where the `:origin` option of the request is
checked, and ephemeral lookup is then not done.

3.1.x Non Expression Based Grammar
----------------------------------
One wart in the 3.1.x based implementation is the requirement to prefix
an expression in the lambda with a syntax marker. In the implementation
"=" was choosen.

i.e.

    $a.select |$x| { = $x =~ awsome }

The problem is that `STATEMENT | EXPRESSION` is ambiguous in
puppet's current grammar (most likely because of the non-parenthesized
function call allowed for statements), and thus, to be able to have the
lambda return a value an expression needs to be turned into a statement
- i.e. a "return".

This is why the "=" is needed in the first 3.1 based implementation,
and why the implementation efforts were changed into the proposed Expression Based
Grammar.

Expression Based Grammar
------------------------

The expression based grammar generalizes everything into expressions
and defines the grammar by using two techniques: Racc precedence (to resolve most shift/reduce conflicts),
and left refactoring by defining a rule chain from lowest to highest precedence. (This is the grammar style
used in the Geppetto Xtext/Antlr based grammar).

The expression based grammar is (currently) found
[here](https://github.com/hlindberg/puppet/blob/pops/lib/puppet/pops/impl/parser/egrammar.ra)
for anyone interested.

To enable use of this approach, it is important to add validation of the
parsed result. That work is mostly what remains (most (if not all) rules
are easily found in Geppetto btw, so there is a list to work from.

The egrammar now has support for the following different styles

    function () λ
    x.function λ
    x.function() λ

Where λ is one of

    {| PARAMETERS | STATEMENTS_OR_EXPR }    
    | PARAMETERS| { STATEMENTS_OR_EXPR }
    | PARAMETERS| => { STATEMENTS_OR_EXPR }

The intent for supporting all of these is to allow UX studies with one
and the same version.

Support for

    function λ    

proved to be very difficult - it is ambiguous without also making lambdas be expressions (which implies Closures),
and requires model rewrite / transformation once a sequence of expressions have been produced by the parser (or rather 
as a final step in parsing). Model rewriting is already used in order to handle the (also) ambiguous non-parenthesized function
calls.

Expression Based Grammar Spin Offs
----------------------------------

It is now possible to make conditional "statements" produce "rvalues". Thus allowing constructs like:

    $a = if $x == 10 { "hello" } else { goodbye }
    $a = case $x { 10,20,30: { "hello" } default: { "goodbye" }
    
etc.




