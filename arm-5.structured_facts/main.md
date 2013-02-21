ARM-5: Structured Facts in Facter
==========================


Summary
-------

One of the oldest problems in Facter is enhancing the representation of fact 
values from simple strings to arbitrarily nested data structures. The most 
easily-accessible use case is enumerating network interfaces on a host:
these are currently separate facts with names of the form `interface_eth0`,
`interface_eth1`, etc., whose values are the "primary" IP address of that 
interface. With structured facts, this could be available in a much more 
natural and useful representation where the key is `interfaces`, its value is a 
hash keyed by interface name, whose leaf-node values are arrays of IP 
addresses. This enables powerful enumeration, searching, and usability in the 
Puppet DSL which is currently either clunky or impossible.

Redmine: [Feature #4561](https://projects.puppetlabs.com/issues/4561)

Goals
-----

This proposal aims to describe a possible implementation, mock up its usage, 
and think through the ramifications of the change to the larger Puppet 
ecosystem.

Non-Goals
---------

None specified at this time.

Success Metrics
---------------

Community concensus on the correct solution and an implementation path in the 
code.

Motivation
----------

* User pain: Representing non-string data in facter is currently painful. A 
  common request is for the right-hand-side value of a fact to be a simple list 
  of items; currently this requires putting some delimiter in the fact value 
  and then parsing it back out into an array in Puppet either through an inline 
  ERB template or a parser function like `split()` from stdlib.

* Competitive disadvantage: Facter's main competititor, ohai, has represented 
  data as a complex JSON data structure from the beginning.

Description
-----------

The proposal consists of several pieces (top-level bullet place-holders for 
now, further details to come)

* Facter itself might need new DSL syntax to create a structured fact
* The wire-format and console-output representation of facts might have to 
  change to present structured facts
* The affordances in the Puppet DSL for accessing structured facts vs legacy 
  facts might be different.
* Different/additional query syntax against PuppetDB or other fact storage
  terminii might be needed to indicate the 

Alternatives
------------

* Top level hash vs sigil distinction in DSL

Testing
-------

TBD

Risks and Assumptions
---------------------

TBD

Dependencies
------------

TDB

Impact
------

TBD
