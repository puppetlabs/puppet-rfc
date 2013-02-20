Spinoff
=======

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
