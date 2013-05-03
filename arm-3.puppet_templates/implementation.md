Implementation
==============

A reference implementation of this ARM is currently available at:
https://github.com/hlindberg/puppet/tree/feature/master/templates

The implementation is based on the `--parser future` and mostly adds
logic to the lexer (broken out into a separate EppScanner) that the lexer uses.

The implementation is based on the work done for ARM 4, Heredoc. And thus, support
for heredoc is included in the Template reference implementation. (This is not an absolute
requirement - but would otherwise require splitting up the Heredoc support into a generic part
(support lexing modes and automatic token at start; this would be difficult without concrete modes).

The implementation is done in the same style as the rest of `--parser future`. Producing a model that is
validated and transformed to AST for evaluation. New "EPP" versions of parser related classes consists of
small derived classes that basically just starts the parser in the epp mode.