(ARM-4) Puppet Heredoc
======================

Summary
-------
This proposal defines Heredoc for the Puppet Language. Puppet Heredoc defines the ability to
inline blocks of text, provide detailed escape processing and trimming of margin and
trailing newline as well as providing a mechanism to syntax check the textual content.

Goals
-----
The main goal is to make it easier to deal with blocks of verbatim text, especially when
containing escape sequences.

Non Goals
---------
These are non goals for this proposal:
* Defining content types
* Defining a plugin mechanism - the proposal is based on using functions, this is however not ideal.
* Enhanced positioning APIs (tracking positions in source via munging of escapes, via interpolation to checking with
  potentially an external parser)
* Ruby Heredoc syntax compatibility (there are several problems with the Ruby Heredoc syntax)

Success Metrics
---------------
No practical success metrics identified.

Motivation
----------
It is currently complicated to write certain types of long strings in the puppet language.
Specifically, this is of value for Powershell [Issue #13196](http://projects.puppetlabs.com/issues/13196).

Alternatives include using templates or external files. For relative small/medium sized elements (and if there
are many) management of external snippets is an unwanted extra complexity.

Compared to competitors where definitions are written in Ruby it is difficult to process blocks of
text and deal with strings at a character level in the Puppet DSL. This proposal makes some common tasks
much easier (handle a block of text, define a margin, and provide detailed escape processing).

Description
===========

Heredoc is a Ruby language feature that allows arbitrary text to be
written in the source file between a HEREDOC start marker and its
matching end (Ruby HEREDOC is not described in detail in this document).
There are several issues related to Ruby HEREDOC:

1.  It is not possible to specify removal of indented space in the
    result. The `<<-` syntax only makes indentation of the end marker
    possible and the user must call gsub to remove initial whitespace. This is difficult to get right,
    and is brittle (change of indentation may break the construct).
2.  It is not possible to get rid of the final `\n` without gsub substitution
3.  Since the heredoc operator `<<` is also used for the left shift
    binary operator it may not appear in all places where a string may
    appear.
4.  The Heredoc text is interpolated, and if verbatim ruby interpolation
    should appear in the output, these must be escaped.
5.  There is no detailed control over escapes.

Puppet Heredoc
--------------

This proposal presents a solution that does not have the same problems as Ruby's heredoc (shown above).

This proposal is built on using the `@()` operator to indicate "here doc, until given end tag", e.g.

    $a = @(END)
    This is the text that gets assigned to $a.
    And this too.
    END

Further, the operator allows specification of the semantics of the
contained text, as follows:

<table>
<tr>
  <td><tt>@(END)</tt></td><td>no interpolation and no escapes</td></tr>
<tr>
  <td><tt>@("END")</tt></td>
  <td>double quoted string semantics for interpolation and no escapes</td>
</tr>
</table>

It is possible to optionally specify a **syntax** that allows validation of the heredoc text, as in this example specifying
that this text is a puppet template.

    $a = @(END:epp)
    This is Embedded PuPpet with the result of an expression <% 1 + 2 %>
    END

And finally, it is possible to specify the processing of escaped characters. By default both 
the `@(END)` form as well as the `@("END")` form has no escapes and text is verbatim. It is possible to turn on all
escapes, or individual escapes by ending the heredoc specification with a `/`. If followed by only a `/` all escapes are turned on,
and if followed by individual escapes, only the specified escapes are turned on.
In this example, the user specifies that `\t` should be turned into tab characters:

    $a = @(END/t)
    There is a tab\tbefore 'before'
    END

The Heredoc Tag
---------------

The heredoc tag may consist of any text (letters, digits, whitespace and underscore) but not any other characters.
It may optionally contain a `:` (colon) followed by the name of the syntax used in the text. The name of the syntax allows
letters, digits, period, plus (as name part separator) and underscore - the name is case insensitive and it is used to enable syntax checking and for
tools to perform syntax coloring and provide user with help/validaton etc. It may also optionally be followed by a specification
of characters that have special meaning when escaped.

    @( <endtag> [:<syntax>] [/<escapes>] )

where

  * <endtag> is any text not containing :, /, ) or \r, \n (This is the text that indicates the end of the text)
  * <syntax> is [a-z][a-zA-Z0-9_]*, and is the name of a syntax
  * <escapes> zero or more of the letters t, s, r, n, L, $ where
    * \t is replaced by a tab character
    * \s is replaced by a space
    * \r is replaced by carriage-return
    * \n is replaced by a new-line
    * \L replaces escaped end of line (\r?\n) with nothing
    * \$ is replaced by a $ and prevents any interpolation
  * if <escapes> is empty (no characters) all escapes t, s, r, n, L, and $ are turned on.

The three parts in the heredoc tag may be surrounded by whitespace for readability.

End Marker
----------

The end marker consists of the heredoc endtag (specified at the opening `@`) and
optional operators to control trimming.

The end marker must appear on a line of its own. It may have leading and
trailing whitespace, but no other text (it is then not recognized as the
end of the heredoc). In contrast to Ruby heredoc, there is no need to
specify at the opening that the end tag may be indented.

The end marker may optionally start with one of the following operators:

<table>
<tr>
  <td><tt>-</tt></td>
  <td>indicates that trailing whitespace (including new line)
  is trimmed from the <i>last line</i>.</td>
</tr>
<tr>
  <td><tt>|</tt></td>
  <td>indentation marker, any left white-space on each line in the text that matches
  the white-space to the left of the position of the `|` char
  is trimmed from the text on all lines in the heredoc. Typically this is a sequence of spaces. A mix of tabs and spaces
  may be used, but there is no tab/space conversion, the same sequence used to indent lines should be used
  to indent the pipe char (thus, it is not possible to trim "part of a tab").</td>
</tr>
<tr>
  <td><tt>|-</tt></td>
  <td>combines indentation and trimming of trailing whitespace</td>
</tr>
</table>

The optional start may be separated from the end marker by whitespace
(but not newline). These are legal end markers:

    -END    
    - END
    |END
    | END
    |           END
    |- END

The sections [Indentation](#indentation), and [Trimming](#trimming)
contains examples and further details.

Indentation
-----------

The position of the | marker before the end tag controls how much
leading whitespace to trim from the text.

    0.........1.........2.........3.........4.........5.........6
    $a = @(END)
      This is indented 2 spaces in the source, but produces
      a result flush left with the initial 'T'
        This line is thus indented 2 spaces.
      | END

Without the leading pipe operator, the end tag may be placed anywhere on
the line. This will include all leading whitespace.

    0.........1.........2.........3.........4.........5.........6
    $a = @(END)
      This is indented 2 spaces in the source, and produces
      a result with left margin equal to the source file's left edge.
        This line is thus indented 4 spaces.
                                            END

When the indentation is right of the beginning position of some lines
the present left whitespace is removed, but not further adjustment is
made, thus altering the relative indentation.

    0.........1.........2.........3.........4.........5.........6
    $a = @(END)
      XXX
        YYY
       | END

Results in the string `XXX\n YYY\n`

### Tabs in the input and indentation

The left trimming is based on using the whitepsace to the left of the pipe character
as a pattern. Thus, a mix of space and tabs may be used, but there is no tab to/from space
conversion so it is not possible to trim part of a "tab". (Note: It is considered best practice to
always use spaces for indentation).

Trimming
--------

It is possible to trim the trailing whitespace from the last line (the
line just before the end tag) by starting the end tag with a minus '-'.
When a '-' is used this is the indentation position.

    0.........1.........2.........3.........4.........5.........6
    $a = @(END)
      This line will not be terminated by a new line
      -END
    $b = @(END)
      This line will not be terminated by a new line
      |- END

This is equivalent to:

    $a =  '  This line will not be terminated by a new line'
    $b =  'This line will not be terminated by a new line'

It is allowed to have whitespace between the - and the tag, e.g.

    0.........1.........2.........3.........4.........5.........6
    $a = @(END)     
      This line will not be terminated by a new line
      - END

Spaces allowed in the tag
-------------------------

White space is allowed in the tag name. Spaces are insignificant between words.

    0.........1.........2.........3.........4.........5.........6
    $a = @(Verse 8 of The Raven)
      Then this ebony bird beguiling my sad fancy into smiling,
      By the grave and stern decorum of the countenance it wore,
      `Though thy crest be shorn and shaven, thou,' I said, `art sure no craven.
      Ghastly grim and ancient raven wandering from the nightly shore -
      Tell me what thy lordly name is on the Night's Plutonian shore!'
      Quoth the raven, `Nevermore.'
      | Verse 8 of The Raven

The comparison of opening and end tag is performed by first removing any quotes, then
trimming leading/trailing whitespace, and then comparing the endtag against the text

Escapes
-------
By default escapes in the text (e.g. \n) is treated as verbatim text. The rationale for this, is that
heredoc is most often used for verbatim text, where any escapes makes it very tedious to correctly mark up
special characters. There are however usecases where it is important to have detailed control over escapes.

The set of allowed escapes are fixed to: t, s, r, n, $, and L (as shown earlier). The 'L' is special in that it allows the
end of lines to be escaped (with the effect of joining them)

    $a = @(END/L)
    First line, \
    also on first line in result
    |- END

Produces

    First line, also on first line in result


Without the specification of `/L` the result would have been

    First line,
    also on first line in result

When an escape is on for any character, it is always possible to escape the backslash to prevent replacement from
taking place.

    $a = @(END/L)
    First line, \\
    on second line
    |- END

An empty escapes specification turns on all escapes.

Multiple Heredoc on the same line
---------------------------------

It is possible to use more than one heredoc on the same line as in this
example:

    foo(@(FIRST), @(SECOND))
      This is the text for the first heredoc
        FIRST
      This is the text for the second
        SECOND

If however, the line is broken like this:

    foo(@(FIRST),
    @(SECOND))

Then the heredocs must appear in this order:

    foo(@(FIRST),
      This is the text for the first heredoc
      FIRST
    @(SECOND))
      This is the text for the second
      SECOND

Additional semantics
--------------------

The `@(tag)` is equivalent to a right value (i.e. a string), only that its
content begins on the next line of not already consumed heredoc text. The
shuffling around of the text is a purely lexical exercise.

Thus, it is possible to continue the heredoc expression, e.g. with a
method call.

    $a = @(END).upcase
      I am not shouting. At least not yet...
      | END

(It is not possible to continue the expression after the end-tag).


Syntax Checking of Heredoc Text
-------------------------------
Syntax checking of a heredoc text is made possible by allowing the heredoc to be tagged with the name
of a syntax/language. Here are some examples:

    @(END:javascript)
    @(END:ruby)
    @(END:property_file)
    @(END:yaml)
    @(END:json)
    @(END:myschema+json)
    @("END":epp)

This way, the checking can take place server side before the content
is (much later) used with possibly very hard to detect problems as a
result.

Implementation could be that a function called `check_<LANG>_syntax` is
called with the result. Thus making the set of languages extensible and
customizable. The syntax name characters `+` and `.` are translated to two and one underscore
character respectively. (See more about `+` below). A period is allowed since many mime types
have this in their names (e.g. vnd.apple.installer+xml).

The parser should ideally call such functions during parsing for non interpolated strings
and the compiler should perform the syntax check when interpolated expressions have been evaluated.
(This turned out to be somewhat tricky in practice - the reference implementation defers checking to evaluation
for all heredoc).

A `+` character is allowed as a name separator (in the style of RFC2046-Mime Media Types). The intent is to
describe syntax that is encoded in formats such as xml or json where the content must be valid json/xml and be valid
in the specified dialect/"schema" (e.g. xslt+xml).

By convention, the most specific syntax should be placed first. When mapping syntax to a function name, each `+` is converted
to two underscores. An attempt is first made with the full name (e.g. 'xslt__xml'), and if no such function exists, the leftmost
name segment is dropped, and a new attempt is made to find a function (e.g. 'xml'). The first found function is given the
task of validating the string. If no function is found, the string is not validated.

The full syntax string is passed to the validation function to allow mapping of segments to schema names.


Testing and Evaluation
----------------------

A reference implementation has been made of this proposal. 
It is based on the --future parser available since Puppet 3.2.

This implementation is currently available here: https://github.com/hlindberg/puppet/tree/feature/master/heredoc

When evaluating this, here are some things to consider:
* What do you think of the syntax?
* Is the described implemented features what you expect? Is something you commonly want to do missing?
* Do you find the lack of detailed mapped source position information a big problem and something that important to implement?
* Do you think it was the right decision to treat both non and interpolated string variants alike (i.e. no escapes by default)?
* Are you ok with the use of punctuation/syntax in the heredoc tag, or would you have liked something more explicit like
  (until=>marker, escapes =>nst, syntax=>xxx) or something similar?
* Do you think existing string capabilities in Puppet are enough and heredoc support is not needed?
* [Support spaces in tags](heredoc.md#spaces-allowed-in-the-tag) or not?  
  Allowing spaces in the tags offers self documentation, but could potentially
  be more difficult to read/understand.
* Should the default indentation be based on start of tag instead of pipe, and pipe if
  it is present? [Indentation](heredoc.md#indentation) - the proposal uses the ERB way ('nothing
  happens unless you say so').
* Is the suggested handling of tabs enough? [Tabs in input](heredoc.md#tabs-in-the-input-and-indentation)
* Should there be a way to right trim all lines?
* Should there be a way to join all lines (without space/nl, or with specificed chars)?
* Do you like using the @ operation for heredoc, or would you like to see something else?
* Do you like the syntax checking feature?


Alternatives and Recommendation
-------------------------------

The main goal is to make it easier to deal with blocks of text. An alternative is to support this in
a more direct fashion as a string with more elaborate delimiters keeping the text in-line (i.e. not starting on the following
line). The same escape and margin handling could be allowed (somehow).

Other alternatives include using more explicit (wordy) specification of the heredoc tag; say something like
(until=>marker, escapes =>nst, syntax=>xxx), thereby also making it possible to pass more parameters to syntax validation.

Risks and Assumptions
---------------------

The implementation is aproximately 90% in the lexer which is already covered with detailed tests. Any issues are
going to be local, easily reproducible and fixed.

The complexity of the lexer naturally increases, but the implementation has already broken out and generalized some
of the concepts that should make it easier to understand the lexer logic.

Dependencies
------------

There are no dependencies (except the 3.2 --parser future). The implementation of this introduces changes in the lexer that
are of value when implementing ARM-3 puppet templates.

The use of heredoc syntax checking based on functions is not 

Impact
------

How will this work impact other parts of the platform, the product,
and the contributors working on them?  Omit any irrelevant items.

- Compatibility:
  The implementation is backwards compatible.

- User experience:
  This brings new capabilities that it is assumed are easier to use than the alternatives. At the same time,
  heredoc is inherently somehwat complex to grasp at fist site.

- Portability:
  Implementation is aware of things like different line endings.

- Documentation:
  The documentation impact is local; simply a new feature.

- Spin-offs/Future work:
  A real plugin system for puppet extensions would be beneficial for adding syntax checkers instead of relying on functions.

