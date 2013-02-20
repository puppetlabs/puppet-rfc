(ARM-4) Puppet Heredoc
======================

Background
----------
It is currently complicated to write certain types of long strings in the puppet language.
Specifically, this is of value for Powershell [Issue #13196](http://projects.puppetlabs.com/issues/13196).

This proposal defines Puppet Heredoc.

Ruby Heredoc
------------

HEREDOC is a Ruby language feature that allows arbitrary text to be
written in the source file between a HEREDOC start marker and its
matching end. There are several issues related to Ruby HEREDOC:

1.  It is not possible to specify removal of indented space in the
    result. The `<<-` syntax only makes indentation of the end marker
    possible and the user must call gsub to remove initial whitespace.
2.  It is not possible to get rid of the final `\n` without substitution
3.  Since the heredoc operator `<<` is also used for the left shift
    binary operator it may not appear in all places where a string may
    appear.
4.  The Heredoc text is interpolated, and if verbatim ruby interpolation
    should appear in the output, these must be escaped.

Proposal for Puppet Heredoc
===========================

To overcome the problems above, this proposal is built on using a new
operator @ to indicate "here doc, until given end tag", e.g.

    $a = @END
    This is the text that gets assigned to $a.
    And this too.
    END

Further, the operator allows specification of the semantics of the
contained text, as follows.

<table>
<tr>
  <td><tt>@END</tt></td><td>no interpolation and no escapes</td></tr>
<tr>
  <td><tt>@"END"</tt></td>
  <td>double quoted string semantics (interpolation and escapes except that  double quote <tt>"</tt> may appear without escape).</td>
</tr>
<tr>
  <td><tt>@'END'</tt></td>
  <td>single quoted string semantics (no interpolation, fewer escapes, accepts single quote <tt>'</tt> without escape).</td>
</tr>
<tr>
  <td><tt>@%END%</tt></td>
  <td>EPP template semantics (no interpolation or escapes),
  signals that the text should be a legal template, but does not evaluate
  the template (it is still a string result, only containing a template).
  See <a href="#combining-heredoc-with-inline-template">Combining heredoc with inline template</a>.</td>
<tr>
</table>

End Marker
----------

The end marker consists of the heredoc tag specified at the opening ´@´ and
operators to control trimming.

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
  <td>indentation marker, white-space to the left of this
  position is trimmed from the text on all lines in the heredoc.</td>
</tr>
<tr>
  <td><tt>|-</tt></td>
  <td>combines indentation and trimming of trailing whitespace</td>
</tr>
</table>

The optional start may be separated from the end marker ny whitespace
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
    $a = @END
      This is indented 2 spaces in the source, but produces
      a result flush left with the initial 'T'
        This line is thus indented 2 spaces.
      | END

Without the leading pipe operator, the end tag may be placed anywhere on
the line. This will include all leading whitespace.

    0.........1.........2.........3.........4.........5.........6
    $a = @END
      This is indented 2 spaces in the source, and produces
      a result with left margin equal to the source file's left edge.
        This line is thus indented 4 spaces.
                                            END

When the indentation is right of the beginning position of some lines
the present left whitespace is removed, but not further adjustment is
made, thus altering the relative indentation.

    0.........1.........2.........3.........4.........5.........6
    $a = @END
      XXX
        YYY
       | END

Results in the string `XXX\n YYY\n`

### Tabs in the input and indentation

Since the puppet language uses a standard tab expansion of 2 spaces this
would also be used when trimming whitespace (first expand any tabs with
one or two spaces depending on even/odd position on line, and then
trim).

Trimming
--------

It is possible to trim the trailing whitespace from the last line (the
line just before the end tag) by starting the end tag with a minus '-'.
When a '-' is used this is the indentation position.

    0.........1.........2.........3.........4.........5.........6
    $a = @END
      This line will not be terminated by a new line
      -END
    $b = @END
      This line will not be terminated by a new line
      |- END

This is equivalent to:

    $a =  '  This line will not be terminated by a new line'
    $b =  'This line will not be terminated by a new line'

It is allowed to have whitespace between the - and the tag, e.g.

    0.........1.........2.........3.........4.........5.........6
    $a = @END    
      This line will not be terminated by a new line
      - END

Spaces allowed in the tag
-------------------------

Spaces are allowed in the tag name when the form is `"`, `'`, or `%`.
The end marker may use the tag without the quotes. Spaces are insignificant
between words.

    0.........1.........2.........3.........4.........5.........6
    $a = @"Verse 8 of The Raven"    
      Then this ebony bird beguiling my sad fancy into smiling,
      By the grave and stern decorum of the countenance it wore,
      `Though thy crest be shorn and shaven, thou,' I said, `art sure no craven.
      Ghastly grim and ancient raven wandering from the nightly shore -
      Tell me what thy lordly name is on the Night's Plutonian shore!'
      Quoth the raven, `Nevermore.'
      | Verse 8 of The Raven

The comparison of opening and end tag is performed by first removing any quotes, then
all whitespace, and then comparing the two "shrink wrapped" tags.

Multiple Heredoc on the same line
---------------------------------

It is possible to use more than one heredoc on the same line as in this
example:

    foo(@FIRST, @SECOND)    
      This is the text for the first heredoc
        FIRST
      This is the text for the second
        SECOND

If however, the line is broken like this:

    foo(@FIRST,    
    @SECOND)

Then the heredocs must appear in this order:

    foo(@FIRST,    
      This is the text for the first heredoc
      FIRST
    @SECOND)
      This is the text for the second
      SECOND

Additional semantics
--------------------

The `@tag` is equivalent to a right value (i.e. a string), only that its
content begins on the next line not already consumed heredoc text. The
shuffling around of the text is a purely lexical exercise.

Thus, it is possible to continue the heredoc expression, e.g. with a
method call.

    $a = @END.upcase
      I am not shouting. At least not yet...
      | END

Combining heredoc with inline template
======================================

Naturally, heredoc can be combined with inline templates just like any
other string, but these strings can not be syntax checked by external
tools (if a heredoc contains a template and it is later evaluated, any
errors will be detected at that time by the puppet runtime).

To help external tools, the heredoc syntax @%tag% is recommended as this
allows tools like Geppetto to provide syntax coloring, syntax validation
as well as reference checking also for the inline template strings.

Template wise, an `@END` is equivalent to `@%END%`.

Discussion
==========

Support syntax checking of additional grammars
----------------------------------------------

It is proposed that `@%tag%` means that the string has EPP syntax.
Wouldn't it be great to be able to specify additional syntaxes and
allowing them to be checked?

This could be done by "tagging the tag" with the name of the syntax.
e.g.

    @END:EPP    
    @END:JavaScript
    @END:Ruby
    @END:PropertyFile
    @END:Yaml
    @END:Json
    @"END":EPP

This way, the checking would take place server side before the content
is (much later) used with possibly very hard to detect problems as a
result.

Implementation could be that a function called `check_syntax-<LANG>` is
called with the result. Thus making the set of languages extensible and
customizable.
