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

The intent is to add the switches --markup and --markdown [message] to 
the puppet master command to allow an administrator to change the 
internal status of the master process. 

Within lib/puppet/status.rb there is simple support allows the 
internal status to be set when the Puppet::Status object is created. 
The switches will manipulate this internal status directly so that 
calls to https://<master>:8140/production/status/no_key will change
the static value of "is_alive" to "is_down". This will allow a load
balancer to mark down the master based on the response from the 
status REST call. 

The optional message will be added to the PSON output of the status 
REST call if specified. At this point it is recommended that the 
key label for the message would be 'reason' to allow automation 
software to use the message in notifications or logging of events
related to the master being marked down. 

This proposal will also provide future options to allow the master 
to mark itself down in the event it detects an error condition in 
which it is not able to recover from. Conditions such as not 
receiving proper responses from the ENC could be detected and take
the master out of service to prevent it from providing invalid or
wrong catalogs to an agent. 


Testing and Evaluation
----------------------

TBD

Alternatives and Recommendation
-------------------------------

None at this time.


Risks and Assumptions
---------------------

No determined risks.


Dependencies
------------

   * None
   
   
Impact
------

No anticipated impact to other components.
