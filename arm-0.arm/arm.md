(ARM 0) Puppet Armatures 
========================

Introduction
------------

There is currently a gap in the process for developing Puppet that is
sometimes critiqued as lack of openness,  frustration over where and how
to get ideas heard, and the perception that the redmine issue tracker
functions as a dumping ground for fly by feature requests that sit there
for years with no, or very little, action or value.

Other open source projects have adopted an enhancement process to deal
with such issues, and thus, this is an attempt to define ARM - Puppet
Armatures. This proposal is heavily inspired by the Java
community's [JEP][JEP1] process.

In some cases, ARMs are artifacts of the process leading up to one or several
issues/feature requests in the issue tracker. In others, they may be detailed,
principled specifications for major innovations.

Overview
--------

Puppet Armatures is a process for collecting, reviewing,
sorting, and recording the result of proposals for enhancements to
Puppet and related ecosystem and processes.

An enhancement is an effort to design and implement a non trivial
change, or other types of work worth communicating broadly. An
enhancement; hereafter an "ARM" should be drafted for any work that meets
one of the following criteria:

1.  It requires more than 2 weeks of engineering
    construction effort, or
2.  It makes a significant change to something in the Puppet ecosystem, or to 
    the processes and infrastructure by which it is
    developed, or
3.  It is in high demand by developers or users

In addition, some changes are always considered significant:

1.  Changes to the Puppet Language
2.  Changes in evaluation semantics

The primary goal of the process is to produce an up-to-date list of
proposals serving as a long term roadmap for Puppet Projects. The intent
is that this roadmap extends well into the future (several years) to
allow for sufficient time for the most complex proposals to be
researched, defined and implemented.

The process is public; anyone may submit proposals as well as contribute
to any of the proposals.

The occurrence of a ARM only means that it is a proposal of record from
a technical perspective. There is no guarantee that anyone will work on
it, much less that its end result will appear in any release of any
project.

It is up to the project leadership to ascertain whether sufficient
collaborators have signed up to complete the work of a ARM to consider
it to be funded. Non funded, stale or otherwise defunct ARMs are taken
off the list.


ARM Process
-----------

A successful ARM (and we want them _all_ to be successful!) passes through the
following states:

[(Here's a diagram of the process)]('arm-01-01.png')

* **Draft** —  In circulation by the idea's champion for initial review and
  consensus- building; Generally a fork of the ARM repository that has not
  been merged into the master branch.

* **Posted** — Pushed into the ARM repository by the champion for wider
  review. When an ARM is posted (via a pull request), comments must be
  requested from the [Puppet Developers](https://groups.google.com/group
  /puppet-dev) and [Puppet Users](http://groups.google.com/group/puppet-users)
  mailing lists. Feedback from that forum, and any other means the champion
  wishes, is incorporated by the champion and other authors until the ARM is
  ready for a more formal review.

* **Submitted** — Declared by the champion as ready for evaluation by the
  Armatures governors. A formal review results in an up-or-down decision. If
  the proposal is accepted, it moves to Candidate status.

* **Candidate** — Accepted for inclusion in iteration planning (costing and
  scheduling). A [ticket](http://projects.puppetlabs.com/puppet) is created
  for the sole purpose of tracking development.

* **Funded** — Implementation is costed and staffed. In some cases, Puppet
  Labs will be responsible for executing on an ARM; in others, community
  contributors (likely the ARM authors) are responsible for execution,
  following the normal procedures for [contributing to Puppet]
  (https://github.com/puppetlabs/puppet/blob/master/CONTRIBUTING.md).

* **Completed** — Built, tested, packaged, released. Once completed, ARMs are
  archived and marked as such.

* **Rejected:** — We hope it doesn't happen, but there may come a point we
  need to reject proposals outright. The proposal will still be in the
  repository, and we'll let you know exactly why we reject a proposal.
  Rejected proposals may be resubmitted after necessary rework.

* **Withdrawn:** — If the ARM champion does not wish to continue further work,
  it can be marked as withdrawn. The proposal will contiune to exist, and may
  work may be taken up by a new champion.

### ARMs Talks, and ARM Rallies

There should be regular ARMs talks; where proposals for new ARMs are
presented, major revisions reviewed and where the ARM process itself is
reviewed. These meetings are open to the public. (The collaborators of a
particular ARM may naturally hold their own meetings).

ARM rallies should be arranged whenever there is a Puppet conf/camp -
this includes communicating the state of ARMs and possibly organizing
reviews/feedback on ARMs in the pipeline.

Creating an ARM
---------------

The ARM consists of a Git repository where each top level folder is a
ARM identified by an ARM number and the name of the ARM.
One folder exists for defunct/old ARMs, and one for accepted/implemented
and archived ARMs. Defunct ARMs are garbage collected after 12 months of
their deprecation. Archived ARMs are kept (but are not updated to keep
in sync with the implementation).

To request a new ARM identifier or suggest archiving an old one, submit a 
Github Issue against the main repository. The governors of the ARM resolve additions
by ascertaining the seriousness and likelihood of funding, assign a ARM
number and creates a top level folder for the ARM. The seriousness
behind an idea does not necessarily mean that it is not a "wacky idea",
only that there is serious intent to work on the idea.

The ARM governors create a folder with initial metadata (see below) and a
template for the proposal. (See the JEP template as a reference to what
it may contain - link at the end of this document).

Contributions to a ARM are done via github pull requests. The ARM is
worked on until there is a first acceptable draft. It is then merged to
the master ARM repository.

We suggest that people working on a larger ARM fork the repository and use the 
fork (and its issue tracker) as the collaboration point to drive that ARM to
the point where it's ready for submission to the upstream repository.

The issue tracker for the master ARM project is used only for the
lifecycle of ARMs, not for handling issues/work related to the content
of the ARMs.

The master ARM repository is owned/goverened by Puppet Labs but commit access
should be opened to community members who have demonstrated interest in working 
on ARMs.

Text Format
-----------

Proposals are written using Github Flavored Markdown.

Encoding is always UTF-8.

The templates in ARM-1 should be a starting point, but a minimal beginning 
point might be simply writing to the section header prompts in [the JEP2 
document][JEP2].

Auxiliary Files
---------------

The major part of an ARM's content should be plain text, but they may include
auxiliary files such as diagrams. Such files should be named
`arm-[number]-[serial number].[extension]`.

Metadata
--------

A proposal always has one root file `manifest.json` that describes:

1.  **arm**- self reference, the ARM number
2.  **champion** - Github handle of the primary author
2.  **authors** (optional) - Github handles of contributors to the ARM
2.  **effort** - quantity of work required to fully implement. Resonable
    values: XS, S, M, L, XL, XXL
2.  **project** -  a URL to the project's homepage (typically a github
    repository complete with issue tracker); i.e. as simple/complex as
    required by the size/complexity of the ARM. Further information about the
    project/work should be found there (issue tracking, meetings, schedule,
    who's who, etc. as dictated by the complexity/size of the project).
3.  **revision** - a sequence number indicating the revision/version of the
    proposal. Revision uses semver (Major, Minor, Micro, where micro
    changes means that only typos or formatting have changed). The first
    version made available for consideration to be implemented should be
    1.0. Change in the Major number indicates a new completed version to
    be considered for implementation. Changes of the Minor number are
    work in progress revisions up to the next release, mostly of concern
    to those collaborating on the ARM.
4.  **requires** - (optional) a list of ARMs that must be implemented before
    this ARM can be implemented
5.  **issues** - (optional) when the ARM is in the late stages, there will
    be one or several redmine issues tracking implementation work, these
    are also included in this file.
6.  **implementation** - (optional) a list of URLs to puppet project branches
    where exploratory/reference implementions can be found
7.  **area** – (optional, but highly recommended) the target area of the
    Puppet Platform, e.g. puppet, facter, hiera.
8.  **target_version** – (optional) target release series. Useful for
    scheduling work.

Example metadata.json

~~~json
{
  "arm":123,
  "title":"Introduce a new syntax"
  "champion":"hlindberg",
  "authors":[
    "hlinkberg",
    "ahpook",
    "jdwelch"
  ],
  "effort":"XL",
  "project":"https://github.com/hlindberg/arm123",
  "revision":"1.2.3",
  "requires":[
    42,
    36
  ],
  "issues":[
    "https://www.puppetlabs.com/...",
    "...."
  ],
  "implementation":[
    "https://github.com/...",
    "...."
  ],
  "area":"puppet",
  "target_version":"4.x"
}
~~~

The intent of this metadata is to allow automatic publishing of ARMs in an
hyperlinked way and with communication of their status.

Keep in mind we may add fields as necessary, for example including status or a
particular reviewer's name.

Implementation Issues
---------------------

Possible meta data additions:

3.  Startdate
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


[JEP1]: http://openjdk.java.net/jeps/1 "The Java JEP Process"

[JEP2]: http://openjdk.java.net/jeps/2 "The Java JEP Template"

