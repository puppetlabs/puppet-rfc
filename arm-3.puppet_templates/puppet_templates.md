(ARM-3) Puppet Templates - EPP
==============================
A proposal for Embedded Puppet Templates (EPP)

Background
----------

It is currently possible to evaluate templates using ERB by calling
functions that either process an external template (an .erb file), or an
inline template passed as a string. In the ERB template the template
author must use the puppet ruby API to obtain variable values. The
author must be aware of quite a few rules and knowledgeable in Ruby to
be able to avoid problems.

By adding support for templates with puppet logic the problems
associated with using ERB/Ruby/Puppet API are eliminated.

Proposal
========

It seems natural to base the solution on how ERB works although not
required. The idea being that migration from ERB to EPP by using the
same tags.

EPP Tags
--------

When in text mode:

<table>
<tr>
  <td><tt>&lt;%</tt></td>
  <td>Switches to puppet mode</td>
</tr>
<tr>
  <td><tt>&lt;%=</tt></td>
  <td>Switches to puppet expression mode. (Left trimming is not possible)</td>
</tr>
<tr>
  <td><tt>&lt;%%</tt></td>
  <td>Literal <tt>&lt;%</tt></td>
</tr>
<tr>
  <td><tt>%%></tt></td>
  <td>Literal <tt>%&gt;</tt></td>
</tr>
<tr>
  <td><tt>&lt;%-</tt></td>
  <td>Trim Left<br/>
  When the opening tag is <tt>&lt;%-</tt> any whitespace preceding the tag, up to and
  including a new line is not included in the output.
  </td>
</tr>
<tr>
  <td><tt>&lt;%\#</tt></td>
  <td>Comment<br/>
  A comment not included in the output (up to the next <tt>%&gt;</tt>, or right
  trimming <tt>-%&gt;</tt>). Continues in text mode after having skipped the comment
  (observes right trimming semantics).
  </td>
</tr>
</table>

When in Puppet mode:

<table>
<tr>
  <td><tt>%&gt;</tt></td>
  <td>End puppet mode</td>
</tr>
<tr>
  <td><tt>-%&gt;</tt></td>
  <td>Trim Right and end puppet mode</td>
</tr>
</table>

When the closing tag is `-%>` any whitespace following the tag, up to and
including a new line is not included in the output.

When ending puppet mode the result of the puppet logic is rendered to
the output for a puppet expression tag `<%=`, but not for `<%`.

Invocation
==========

Currently a template is evaluated with the template function. This
function could probably be modified to support puppet templates as well.
In this exploratory document a new function is used while exploring a
suitable api.

One idea is that EPP templates are recognized by either ending with
`.epp`, or having an `.epp` pseudo ending before an ending indicating the
type of file (to make editing easier). e.g.

    404page.epp
    404page.epp.html

Both of these signal that this is an epp file, and could be used like
this:

    pptemplate ('404page.epp')

Passing (local) arguments to the template
-----------------------------------------

Since the template is evaluated in the calling context it has access to
all variables visible in this context. If the template is parameterized
and there is a need to use it more than once (with different parameter
values, it becomes difficult.

Since a template evaluation creates a lambda scope for the evaluation of
the template to prevent the template from leaking variable values this
can be used by also passing additional parameters into the context.

These are quivalent:

    $message = "This is not the $x you are looking for"
    
    pptemplate('404page.epp')
    
    pptemplate('404page.epp') |$template| { $template.render() }

> R.I. Pienaar:
> 
> I don't like the new syntax here - pptemplate("the\_template.epp",
> {"message" =\> "hello world"}) seems to fit better
> or better add the much requested named arguments:
> pptemplate("foo.epp", :message =\> "hello world")
> and it seems we should just use template() and let it detect what the
> template is based on file names?
>
>* * * * *
>
>> Henrik Lindberg:
>>
>> mostly agree (call by named parameter is an interesting separate topic I
>> think, has some intrinsic problems combined with varargs, and parameter
>> overloading; but separate topic I think).
>>
>> Regarding using the same function - all for it, but I did not want to
>> start out by being constrained by current function's signature)
>>
>> We can use lambda parameters to pass arguments instead. Either by
>> setting the variable in the context, or passing it to the template
>> function.


These are equivalent:

    pptemplate('404page.epp') |$template| {    
      $message = "This is not the $x you are looking for"
      $template.render()
    }
    
    pptemplate('404page.epp',
      "This is not the $x you are looking for") |$template, $message| {
      $template.render()
    }

> surplus.address:
>
> What does the lamdba yield without the assignment to
> \$template.render()?
>
> I'm just imagining a bunch of users who get as far as seeing it's a
> block to pass parameters, omit the call to .render() and get something
> unexpected.
>
>* * * * *
>
>> Henrik Lindberg:
>>
>> they would get nil back, but pptemplate should check the return and
>> issue an error "Did you forget to call render?"
>>

The `render` function requires an instance of a Template, an internal
object that is created by parsing a template file. It triggers the
rendering of the entire template. It can only be called once in a given
context if the template adds local variables (since they are immutable).

The name of the internal template variable is `$template` by default, but
an author may pick something else (and would then use that varible name
to reference the template if there is a need to produce output (see the
output method below).

Also see Discussion at the end containing an idea to specify local
parameters inside the template. The lambda way shown above would still
be valid since that is the way the template function works, but it
reduces the need to use the lambda and more naturally place the
parameter declarations inside the template.

Appending output in puppet code
-------------------------------

Consider the following EPP template

    Here is a list of servers:
    <ul>
    <% $array.foreach |$x| { %>
      <li><%= $x %>\</li>
    <% } %>
    <ul>
    </p>

And instead using an output method on the template.

    Here is a list of servers:    
    <ul>
    <% $array.foreach |$x| {
      $template.output('  <li>', $x, '</li>\n'>) } %>
    <% } %>
    </ul>

The template output method simply appends its input to what has already
been rendered in the template's output buffer.

Switching Puppet / Text mode constraints
----------------------------------------

Switching to text mode is only supported at positions in the puppet
language where a function call may appear. This limitation should have
no practical consequence, but it required since the lexer/parser
combination needs to be able to look ahead in certain circumstances, and
the semantics for evaluation of certain grammar positions is simply not
defined.

As an example:

    Here is a list of servers:    
    <ul>
    <% $array.foreach %> TEXT <% |$x|
      <li><%= $x %></li>
    <% } %>
    </ul>
    </p>

This will produce a syntax error.

Inline Templates
================

Just as with the inline template support for ERB this can be supported
for puppet. It is just a string being interpreted instead of loading the
template from a file. To also support passing additional parameters, the
`pp_inline_template` takes a single template string, or an array of one
or more template strings as input for the first parameters.

Discussion
==========

> R.I. Pienaar:
>
> would very much like to see this expand to use cases that this solves -
> nice and all to say its a puppet eval - but why? What can we not do
> today? what actual use cases can a user now solve once we have this?
>
> * * * * *
>
>> Henrik Lindberg:
>>
>> The general idea is to not depend on Ruby for template support. Other
>> than that I think it is the same use cases as for why Ruby templates are
>> used.
>>
>> Great to have real world examples, and I love to get more input on this.

Templates are like "puppet eval"
================================

The template support (esp. in the inline form), makes it possible to
construct puppet logic from strings and then evaluate it. This is better
than the ERB templates, as there is at least protection against things
that does not makes sense (see "Allowed Expressions").

A problem is that external files are easily recognized as being puppet
templates and can thus be syntax checked by tools like Geppetto. Inline
templates however are just strings, and therefore much harder to syntax
check. The Puppet Heredoc proposal includes a solution to this where it
is allowed to tag a heredoc with a language (for the purpose of syntax
checking the result).

Allowed expressions
===================

Since templates can contain puppet code and this can contain any type of
expression it is important to constrain the set of legal
expressions/statements.

Variable assignment is by virtue of the implementation already protected
(assigned variables go out of scope when the template has been
evaluated). Clearly, defines, classes, and nodes can not be declared,
but is it ok to create resources from within a template? (Although "No"
would be a reasonable answer - this may however be a common use case, as
this is possible using an ERB template, and we want users to migrate).
Is it ok to include or require classes? Declare dependencies? Realize
resources, collect exported resources?

> surplus.address:
>
> I'd say no, not directly.  That ERB users can do any of this is
> accidental feature due to not sandboxing the template evaluation well
> enough in the first place.
>
> There's already the parser functions available as an extension point, so
> if a template called on create\_resources then that should allowed, but
> generally restricting the template authors to the harm they can do in
> the DSL should be enough power.
>
>* * * * *
>>
>> Henrik Lindberg:
>>
>> the reason I am asking is that EPP means they \*are\* in the DSL in the
>> template as well. I do want to restrict the available expressions, but I
>> want to know if there are valid use cases not to be restrictive.

`<%= $x %>` is clunky
---------------------

Why not simply use Puppet string interpolation? The `<%= $x %>` syntax
is very heavy compared to puppet's simple `$x`, or `${x}`?

The idea is that the EPP tags are different enough to not clash with
most types of file content. If we pick `$x`, or `${x}`, the author may
instead have to use lots of escapes.

The proposal is based on using the ERB tags.

Declaring parameters inside the templates
-----------------------------------------

The proposal shows how addition of local parameters can be made at the
calling side. Should it be possible to declare parameters with default
values inside the template as an alternative? If so, the best solution
is probably to allow an EPP tag that does this (if starting in puppet
mode, I suspect this will enable lambdas to leak and closures to be a
required feature).

> R.I. Pienaar:
>
> would very much like to see this expand to use cases that this solves -
> nice and all to say its a puppet eval - but why? What can we not do
> today? what actual use cases can a user now solve once we have this?
>
> * * * * *
>
>> Henrik Lindberg:
>>
>> The general idea is to not depend on Ruby for template support. Other
>> than that I think it is the same use cases as for why Ruby templates are
>> used.
>>
>> Great to have real world examples, and I love to get more input on this.

Here are ideas for such a tag, it must appear first in the template:

    <%| $path_to_fame, $optional_riches = false %>

That looks a bit magic

    <% parameters: $path_to_fame, $optional_riches = false %>

That works because `parameters:` is not a normal puppet expression, it is
much clearer.

This syntax opens up for other textual oriented processing instruction
tags for future use.
