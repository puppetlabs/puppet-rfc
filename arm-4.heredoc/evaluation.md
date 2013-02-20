Evaluation / Decision Support
=============================
* The heredoc proposal offers options:
  * [Support spaces in tags](heredoc.md#spaces-allowed-in-tags) or not?  
    Allowing spaces in the tags offers self documentation, but could potentially
    be more difficult to read/understand.
  * Should the default indentation be based on start of tag instead of pipe, and pipe if
    it is present? [Indentation](heredoc.md#indentation) - the proposal uses the ERB way (nothing
    happens unless you say so).
  * Is the suggested handling of tabs enough? [Tabs in input](heredoc.md#tabs-in-the-input-and-indentation)
  * Should there be a way to right trim all lines?
  * Should there be a way to join all lines (without space/nl, or with specificed chars)?
  * Do you like using the @ operation for heredoc, or would you like to see something else?
  * Do you like the syntax checking feature?

