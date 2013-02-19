Puppet Enhancement Process (PEP 0 - 0.1)

Introduction
============

There is currently a gap in the process for developing puppet that is
sometimes critiqued as lack of openness,  frustration over where and how
to get ideas heard, and the perception that the redmine issue tracker
functions as a dumping ground for fly by feature requests that sit there
for years with no, or very little, action or value.

Other open source projects have adopted an enhancement process to deal
with such issues, and thus, this is an attempt to define a PEP - Puppet
Enhancement Process. This proposal is heavily inspired by the Java
community's JEP process.

One can view the PEP as the process leading up to one or several
issues/feature requests in the issue tracker.

Overview
========

The Puppet Enhancement Process is a process for collecting, reviewing,
sorting, and recording the result of proposals for enhancements to
Puppet and related ecosystem and processes.

An enhancement is an effort to design and implement a non trivial
change, or other types of work worth communicating broadly. An
enhancement; hereafter a "PEP" should be drafted for any work that meets
one of the following criteria:

1.  It requires more than 2 weeks^[[a]](#cmnt1)^ of engineering
    construction effort, or
2.  It makes a significant change to something in the  Puppet eco
    system, or to the processes and infrastructure by which it is
    developed, or
3.  It is in high demand by developers or users

In addition, some changes are always considered significant:

1.  Changes to the Puppet Language
2.  Changes in evaluation semantics

The primary goal of the process is to produce an up to date list of
proposals serving as a long term roadmap for Puppet Projects. The intent
is that this roadmap extends well into the future (several years) to
allow for sufficient time for the most complex proposals to be
researched, defined and implemented.

The process is public, anyone may submit proposals as well as contribute
to any of the proposals.

The occurrence of a PEP only means that it is a proposal of record from
a technical perspective. There is no guarantee that anyone will work on
it, much less that its end result will appear in any release of any
project.

It is up to the project leadership to ascertain whether sufficient
collaborators have signed up to complete the work of a PEP to consider
it to be funded. Non funded, stale or otherwise defunct PEPs are taken
off the list.

PEP Talks, and PEP Rally
========================

There should be regular PEP talks; where proposals for new PEPs are
presented, major revisions reviewed and where the PEP process itself is
reviewed. These meetings are open to the public. (The collaborators of a
particular PEP may naturally hold their own meetings).

PEP Rallies should be arranged whenever there is a Puppet conf/camp -
this includes communicating the state of PEPs and possibly organizing
reviews/feedback on PEPs in the pipeline.

PEP implementation
==================

The PEP consists of a Git repository where each top level folder is a
PEP identified by a PEP number and the name of the PEP^[[b]](#cmnt2)^.
One folder exists for defunct/old PEPs, and one for accepted/implemented
and archived PEPs. Defunct PEPs are garbage collected after 12 months of
their deprecation. Archived PEPs are kept (but are not updated to keep
in sync with the implementation).

The github issue tracker for the PEP project is used to propose
addition/archiving of PEPs. The governors of the PEP resolves additions
by ascertaining the seriousness and likelihood of funding, assigns a PEP
number and creates a top level folder for the PEP. The seriousness
behind an idea does not necessarily mean that it is not a "whacky idea",
only that there is serious intent to work on the idea.

The PEP folder is created with initial metadata (see below), and a
template for the proposal. (See the JEP template as a reference to what
it may contain - link at the end of this document).

Contributions to a PEP are done via github pull requests. The PEP is
worked on until there is a first acceptable draft. It is then merged to
the master PEP repository.

It is suggested that a larger PEP forks the repository and uses the fork
as the collaboration project to drive that PEP; using the issue tracker
for the fork to handle its issues.

\
The issue tracker for the master PEP project is used only for the
lifecycle of PEPs, not for handling issues/work related to the content
of the PEPs.

The master PEP repository is owned by Puppet Labs.

Text Format
-----------

Proposals are written using Github Flavored Markdown.

Metadata
--------

A proposal always has one root file manifest.json^[[c]](#cmnt3)^ that
describes:

1.  pep - self reference, the pep number
2.  project -  a URL to the project's homepage (typically a github
    repository complete with issue tracker)^[[d]](#cmnt4)^; i.e. as
    simple/complex as required by the size/complexity of the PEP.
    Further information about the project/work should be found there
    (issue tracking, meetings, schedule, who's who, etc. as dictated by
    the complexity/size of the project).
3.  revision - a sequence number indicating the revision/version of the
    proposal. Revision uses semver (Major, Minor, Micro) where micro
    changes means that only typos or formatting have changed). The first
    version made available for consideration to be implemented should be
    1.0. Change in the Major number indicates a new completed version to
    be considered for implementation. Changes of the Minor number are
    work in progress revisions up to the next release, mostly of concern
    to those collaborating on the PEP.
4.  requires - (optional) a list of PEPs that must be implemented before
    this PEP can be implemented
5.  issues - (optional) when the PEP is in the late stages, there will
    be one or several redmine issues tracking implementation work, these
    are also included in this file.

{

   "pep": 123,

   "revision":"1.2.3",

   "project":"https://github.com/hlindberg/pep123",

   "requires":[

      42,

      36

   ],

   "issues":[

      "https://www.puppetlabs.com/...",

      "...."

   ]

}

The intent of this metadata is to allow automatic publishing of PEP's in
an hyperlinked way and with communication of their status.

Implementation Issues
=====================

Possible meta data additions:

1.  Effort - XS, S, M, L, XL, XXL
2.  Duration XS, S, M, ...
3.  Startdate
4.  Component(s)/Area(s) - e.g. Puppet core, factor, mcollective,
    dev-process, pep, etc.
5.  Template/Pep version
6.  Creation date (is perhaps already defined by looking at git data)
7.  Author / Organization (is perhaps already defined by using git)
8.  Associated issues this PEP will address (fix)

Endorsements / Approvals
------------------------

If we want to keep track of endorsements/approvals we could use signed
tags. It may be difficult when the master project contains all PEPs -
not investigated. An alternative is also to have PEPs ultimate
approval/endorsement be controlled by a redmine issue.

References
==========

The Java JEP Process:
[http://openjdk.java.net/jeps/1](http://openjdk.java.net/jeps/1)

The Java JEP Template:
[http://openjdk.java.net/jeps/2](http://openjdk.java.net/jeps/2)

[[a]](#cmnt_ref1)Henrik Lindberg:

Used by JEP

[[b]](#cmnt_ref2)J.D. Welch:

This is \*precisely\* what I had in mind. :)

[[c]](#cmnt_ref3)Andy Parker:

manifest.cudf perhaps? We were talking about that format earlier, would
this be a good place to use it?

* * * * *

Henrik Lindberg:

Not sure the CUDF is the best way to express this; looks more like a
machine format - if we need sat solving we would translate to it... but
don't think that is of value here.

[[d]](#cmnt_ref4)Andy Parker:

I suppose that more complicated PEPs that require changes across several
projects might have multiple repositories and that is when this would be
a link to a homepage instead of a repo...unless the multiple projects
were brought together with git subtrees or some such

* * * * *

Henrik Lindberg:

It boils down to a repo somewhere - an ultimately it will need to be at
github to make a pull request - but if a group wants to work on a
proposal somewhere else, an URL is needed to find where this takes
place. (Could also not allow this and require that it is always done at
github; in which case this is always a reference to the github repo).
