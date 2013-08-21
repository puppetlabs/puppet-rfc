ARM-15: Master to Produce Meaningful Status Messages
====================================================


Summary
-------

A master responds to a status request (i.e. 
https://<master>:8140/production/status/no_key) with effectively static
text. This proposal is to add command line switches to control the 
response that a master responds with and provide the master with the
ability for it to respond with changes in its internal state. Proper
status response is necessary to handle taking the master out of 
service in the load balancer pool in an automated fashion. 


Goals
-----

   * Give the master to remove itself from a load balancer pool if it detects an error condition.


Success Metrics
---------------

The master's status response can be manually controlled. 

Motivation
----------

As Puppet infrastructure is grown, it becomes necessary at times to take
a master out of service for maintenance or testing without making changes
to the load balancer configuration. This proposal provides for a graceful
shutdown of client connections and eventually enables the master to become
more self aware and take itself out of service when it detects a problem
condition. 


Description
-----------

REQUIRED -- Describe the enhancement in detail: Both what it is and,
to the extent understood, how you intend to implement it.  Summarize,
at a high level, all of the interfaces you expect to modify or extend,
including APIs, command-line switches, network protocols,
and file formats.  Explain how failures in applications using this
enhancement will be diagnosed, both during development and in
production.  Describe any open design issues.

This section will evolve over time as the work progresses, ultimately
becoming the authoritative high-level description of the end result.
Include hyperlinks to additional documents as required.

Testing and Evaluation
----------------------

What kinds of test development and execution will be required in order
to validate this enhancement, beyond the usual mandatory unit tests?
Be sure to list any special platform or hardware requirements.

What criteria should people use to evaluate the alternatives? If
there are suggestions you have as the author for helping readers
decide between alternatives (or whether to the ARM should be 
implemented at all), build a decision tree here.

Alternatives and Recommendation
-------------------------------

Did you consider any alternative approaches or technologies?  If so
then please describe them here and explain why they were not chosen.

Describe which, if any, of the alternatives you recommend and why
you prefer it. This could walk through the Author's use of the
"Evaluation" decision tree explaining the rationale.


Risks and Assumptions
---------------------

Describe any risks or assumptions that must be considered along with
this proposal.  Could any plausible events derail this work, or even
render it unnecessary?  If you have mitigation plans for the known
risks then please describe them.

Dependencies
------------

   * None
   
   
Impact
------

How will this work impact other parts of the platform, the product,
and the contributors working on them?  Omit any irrelevant items.

- Other Puppet components: ...
- Compatibility: ...
- Security: ...
- Performance/scalability: ...
- User experience: ...
- I18n/L10n: ...
- Accessibility: ...
- Portability: ...
- Packaging/installation: ...
- Documentation: ...
- Spin-offs/Future work: ...
- Other: ...
