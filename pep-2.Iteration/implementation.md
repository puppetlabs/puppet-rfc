Implementation
==============
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

    $a.select {|$x| = $x =~ awsome }

One remaining problem is that `STATEMENT | EXPRESSION` is ambiguous in
puppet's current grammar (most likely because of the non-parenthesised
function call allowed for statements), and thus, to be able to have the
lambda return a value an expression needs to be turned into a statement
- i.e. a "return".

This is why the "=" is needed in the first 3.1 based implementation.

UPDATE:

As a test, a grammar for puppet that changed it to be expression based
was implemented, and that proved to be very doable by simply using a
different approach than in the current grammar.

This experiment (egrammar.ra) is currently
found [here](https://github.com/hlindberg/puppet/blob/pops/lib/puppet/pops/impl/parser/egrammar.ra)
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

    {| PARAMETERS | STATEMENTS_OR_EXPR }    
    | PARAMETERS| { STATEMENTS_OR_EXPR }
    | PARAMETERS| => { STATEMENTS_OR_EXPR }

The intent for supporting all of these is to allow UX studies with one
and the same version.
