(ARM 0) Puppet Armatures 
========================

Introduction
------------

There is currently a gap in the process for developing puppet that is
sometimes critiqued as lack of openness,  frustration over where and how
to get ideas heard, and the perception that the redmine issue tracker
functions as a dumping ground for fly by feature requests that sit there
for years with no, or very little, action or value.

Other open source projects have adopted an enhancement process to deal
with such issues, and thus, this is an attempt to define ARM - Puppet
Armatures. This proposal is heavily inspired by the Java
community's JEP process.

One can view the ARM as the process leading up to one or several
issues/feature requests in the issue tracker.

Overview
========

Puppet Armatures is a process for collecting, reviewing,
sorting, and recording the result of proposals for enhancements to
Puppet and related ecosystem and processes.

An enhancement is an effort to design and implement a non trivial
change, or other types of work worth communicating broadly. An
enhancement; hereafter an "ARM" should be drafted for any work that meets
one of the following criteria:

1.  It requires more than 2 weeks of engineering
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

The occurrence of a ARM only means that it is a proposal of record from
a technical perspective. There is no guarantee that anyone will work on
it, much less that its end result will appear in any release of any
project.

It is up to the project leadership to ascertain whether sufficient
collaborators have signed up to complete the work of a ARM to consider
it to be funded. Non funded, stale or otherwise defunct ARMs are taken
off the list.

ARMs Talks, and ARM Rallies
===========================

There should be regular ARMs talks; where proposals for new ARMs are
presented, major revisions reviewed and where the ARM process itself is
reviewed. These meetings are open to the public. (The collaborators of a
particular ARM may naturally hold their own meetings).

ARM rallies should be arranged whenever there is a Puppet conf/camp -
this includes communicating the state of ARMs and possibly organizing
reviews/feedback on ARMs in the pipeline.

ARM implementation
==================

The ARM consists of a Git repository where each top level folder is a
ARM identified by an ARM number and the name of the ARM.
One folder exists for defunct/old ARMs, and one for accepted/implemented
and archived ARMs. Defunct ARMs are garbage collected after 12 months of
their deprecation. Archived ARMs are kept (but are not updated to keep
in sync with the implementation).

The github issue tracker for the ARM project is used to propose
addition/archiving of ARMs. The governors of the ARM resolves additions
by ascertaining the seriousness and likelihood of funding, assigns a ARM
number and creates a top level folder for the ARM. The seriousness
behind an idea does not necessarily mean that it is not a "whacky idea",
only that there is serious intent to work on the idea.

The ARM folder is created with initial metadata (see below), and a
template for the proposal. (See the JEP template as a reference to what
it may contain - link at the end of this document).

Contributions to a ARM are done via github pull requests. The ARM is
worked on until there is a first acceptable draft. It is then merged to
the master ARM repository.

It is suggested that a larger ARM forks the repository and uses the fork
as the collaboration project to drive that ARM; using the issue tracker
for the fork to handle its issues.

\
The issue tracker for the master ARM project is used only for the
lifecycle of ARMs, not for handling issues/work related to the content
of the ARMs.

The master ARM repository is owned by Puppet Labs.

Text Format
-----------

Proposals are written using Github Flavored Markdown.

Encoding is always UTF-8.

Metadata
--------

A proposal always has one root file manifest.json that
describes:

1.  arm - self reference, the ARM number
2.  project -  a URL to the project's homepage (typically a github
    repository complete with issue tracker); i.e. as
    simple/complex as required by the size/complexity of the ARM.
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
    to those collaborating on the ARM.
4.  requires - (optional) a list of ARMs that must be implemented before
    this ARM can be implemented
5.  issues - (optional) when the ARM is in the late stages, there will
    be one or several redmine issues tracking implementation work, these
    are also included in this file.

    {

       "arm": 123,

       "revision":"1.2.3",

       "project":"https://github.com/hlindberg/arm123",

       "requires":[

          42,

          36

       ],

       "issues":[

          "https://www.puppetlabs.com/...",

          "...."

       ]

    }

The intent of this metadata is to allow automatic publishing of ARM's in
an hyperlinked way and with communication of their status.

Implementation Issues
=====================

Possible meta data additions:

1.  Effort - XS, S, M, L, XL, XXL
2.  Duration XS, S, M, ...
3.  Startdate
4.  Component(s)/Area(s) - e.g. Puppet core, factor, mcollective,
    dev-process, arm, etc.
5.  Template/Pep version
6.  Creation date (is perhaps already defined by looking at git data)
7.  Author / Organization (is perhaps already defined by using git)
8.  Associated issues this ARM will address (fix)

Endorsements / Approvals
------------------------

If we want to keep track of endorsements/approvals we could use signed
tags. It may be difficult when the master project contains all ARMs -
not investigated. An alternative is also to have ARMs ultimate
approval/endorsement be controlled by a redmine issue.

References
==========

The Java JEP Process:
[http://openjdk.java.net/jeps/1](http://openjdk.java.net/jeps/1)

The Java JEP Template:
[http://openjdk.java.net/jeps/2](http://openjdk.java.net/jeps/2)

