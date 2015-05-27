Puppet RFC
===

Puppet RFC Repository
---------------------

This repository contains Puppet RFCs, which are "Requests For Comments" in the tradition of the IETF 
RFC process. They are proposals to enhance / add features to Puppet or something in the Puppet 
ecosystem. This is a community-focused process intended to improve openness and transparency in both 
Puppet Labs-funded and contributor efforts. For details, please read [PRFC-0](prfc-0.prfc/index.md), 
which describes the rationale and process.

Historical note
---------------

From their introduction in 2013 until a renaming/reboot in mid-2015, the process and artifacts were 
called "Armatures", abbreviated to "ARMs". While the git repository and all current documentation 
are updated, it's possible you will see references to the old name in immutable artifacts like 
mailing list threads, bug tracker tickets, etc.

Index of PRFCs
-------------
<table>
<tr>
  <th>PRFC</th>
  <th>Title</th>
  <th>Description</th>
  <th>Revision</th>
  <th>Date</th>
  <th>State</th>
</tr>
<tr>
  <td>0</td>
  <td><a href="prfc-0.prfc/prfc.md">Meta-RFC</a></td>
  <td>Describes the Puppet RFC process itself</td>
  <td>1.0.0</td>
  <td>March 26, 2013</td>
  <td>Completed</td>
</tr><tr>
  <td>1</td>
  <td><a href="prfc-1.templates/templates">Templates</a></td>
  <td>Templates for creating a new Puppet RFC</td>
  <td>1.0.0</td>
  <td>March 26, 2013</td>
  <td>Completed</td>
</tr><tr>
  <td>2</td>
  <td><a href="prfc-2.iteration/index.md">Iteration</a></td>
  <td>Add iteration to the Puppet Language</td>
  <td>2.0.0</td>
  <td>Oct 24, 2013</td>
  <td>Completed</td>
</tr><tr>
  <td>3</td>
  <td><a href="prfc-3.puppet_templates/index.md">Puppet Templates</a></td>
  <td>Add support to write templates in the Puppet Language</td>
  <td>1.0.0</td>
  <td>May 1, 2013</td>
  <td>Submitted<br/><a href="https://groups.google.com/d/topic/puppet-dev/HZXt_G0nZLE">Discuss</a></td>
</tr><tr>
  <td>4</td>
  <td><a href="prfc-4.heredoc/index.md">Heredoc</a></td>
  <td>Add support for Heredoc feature to Puppet Language</td>
  <td>2.0.0</td>
  <td>April 23, 2013</td>
  <td>Submitted<br><a href="https://groups.google.com/d/topic/puppet-dev/mrYmTa_2L6M">Discuss</a></td>
</tr><tr>
  <td>5</td>
  <td><a href="prfc-5.structured_facts/index.md">Structured Facts</a></td>
  <td>Implementation of structured facts in Facter</td>
  <td>0.0.1</td>
  <td>February 25, 2013</td>
  <td>Draft</td>
</tr><tr>
  <td>6</td>
  <td><a href="prfc-6.capabilities/index.md">Puppet Capabilities</a></td>
  <td>Support cross-host application management and monitoring</td>
  <td>1.0.0</td>
  <td>February 20, 2013</td>
  <td>Draft</td>
</tr><tr>
  <td>7</td>
  <td><a href="prfc-7.puppet_types/index.md">Puppet Types</a></td>
  <td>Express Puppet Types in the Puppet DSL</td>
  <td>0.0.0</td>
  <td>February 26, 2013</td>
  <td>Draft</td>
</tr><tr>
  <td>8</td>
  <td><a href="prfc-8.puppet_bindings/index.md">Puppet Bindings</a></td>
  <td>Unified dependency injection framework for Puppet</td>
  <td>0.0.1</td>
  <td>June 1, 2013</td>
  <td>Draft<br><a href="https://groups.google.com/d/topic/puppet-dev/ITIqQrEY9ZY/discussion">Discuss</a></td>
</tr><tr>
  <td>9</td>
  <td><a href="prfc-9.data_in_modules/index.md">Data in Modules</a></td>
  <td>Focused specification for providing Hiera data in Puppet Modules</td>
  <td>0.0.2</td>
  <td>Aug 16, 2013</td>
  <td>Draft</td>
</tr><tr>
  <td>10</td>
  <td><a href="prfc-10.additional_node_scope/index.md">Additional Node Scope</a></td>
  <td>Add a level of scope for nodes</td>
  <td>0.0.0</td>
  <td>March 1, 2013</td>
  <td>New</td>
</tr>
</tr><tr>
  <td>11</td>
  <td><a href="prfc-11.execution_model/index.md">Execution Model</a></td>
  <td>Describe a next-gen execution model for Puppet</td>
  <td>0.0.2</td>
  <td>March 22, 2013</td>
  <td>New</td>
</tr><tr>
  <td>12</td>
  <td><a href="prfc-12.star_resource_data_type/index.md">Star Resource Data Format</a></td>
  <td>Adds new resource data format to describe a resource without automatically declaring it</td>
  <td>0.0.1</td>
  <td>April 1, 2013</td>
  <td>New</td>
</tr><tr>
  <td>13</td>
  <td><a href="prfc-13.ssl_behaviour/index.md">SSL Behaviour</a></td>
  <td>Describe Puppet's use of SSL, configuration options, and best practices.</td>
  <td>0.0.1</td>
  <td>April 17, 2013</td>
  <td>New</td>
</tr><tr>
  <td>14</td>
  <td><a href="prfc-14.reboot/index.md">Managing System Reboots</a></td>
  <td>Describe Puppet's ability for managing system reboots.</td>
  <td>0.0.1</td>
  <td>April 17, 2013</td>
  <td>New - <a href="https://groups.google.com/forum/?fromgroups=#!topic/puppet-dev/5QFelBbbAMw">Discuss</a></td>
</tr><tr>
    <td>15</td>
    <td><a href="prfc-15.master_status/index.md">Master to Produce Meaningful Status Messages</a></td>
    <td>Allow administrators and/or the master process itself to update the status response when 
    /production/status/no_key is hit so that proper load balancer support can be achieved.</td>
    <td>0.0.1</td>
    <td>August 21, 2013</td>
    <td>New - <a
    href="https://groups.google.com/forum/?fromgroups=#!topic/puppet-dev/5QFelBbbAMw">Discuss</a></td>
</tr><tr>
    <td>16</td>
    <td><a href="prfc-16.acls/index.md">Managing ACLs (Access Control Lists)</a></td>
    <td>Describe Puppet's ability for managing access control lists.  </td>
    <td>0.0.1</td>
    <td>October 25, 2013</td>
    <td>New - <a href="https://groups.google.com/forum/#!topic/puppet-dev/9xq-oFWSqXo">Discuss</a></td>
</tr><tr>
    <td>17</td>
    <td><a href="prfc-17.lambda_in_case/index.md">Lambda Case Option</a></td>
    <td>Add ability to use a lambda as case option for advanced matching. (See <a href="https://tickets.puppetlabs.com/browse/PUP-4193">PUP-4193</a>)</td>
    <td>0.0.0</td>
    <td>-</td>
    <td>New. <a href="https://groups.google.com/forum/#!topic/puppet-dev/">Discuss TBD</a></td>
</tr><tr>
    <td>18</td>
    <td><a href="prfc-18.ignored_parameter/index.md">Ignored Parameter</a></td>
    <td>The work is to compare how variables like this is used in other languages and how this can be applied/made useful in the Puppet Language. (See <a href="https://tickets.puppetlabs.com/browse/PUP-4193">PUP-4193</a>)</td>
    <td>0.0.0</td>
    <td>-</td>
    <td>New. <a href="https://groups.google.com/forum/#!topic/puppet-dev/">Discuss TBD</a></td>
</tr><tr>
    <td>19</td>
    <td><a href="prfc-19.direct_eval/index.md">Direct Evaluation of Resource Instantiation</a></td>
    <td>Current behavior makes it very difficult for users to reason about evaluation order; this work aims to fix that.</td>
    <td>0.0.1</td>
    <td>-</td>
    <td>New. <a href="https://groups.google.com/forum/#!topic/puppet-dev/">Discuss TBD</a></td>
</tr><tr>
    <td>20</td>
    <td><a href="prfc-20.iteration_control/index.md">Iteration Control</a></td>
    <td>The idea is to specify the semantics for two functions: break(val = undef), and next(val=undef) that enables control of iteration.</td>
    <td>0.0.1</td>
    <td>-</td>
    <td>New. <a href="https://groups.google.com/forum/#!topic/puppet-dev/">Discuss TBD</a></td>
</tr><tr>
    <td>21</td>
    <td><a href="prfc-21.dsc/index.md">PowerShell Desired State Configuration (DSC)</a></td>
    <td>Puppet should support integration with PowerShell DSC as a supported module. [FM-2410](https://tickets.puppetlabs.com/browse/FM-2410) </td>
    <td>0.0.1</td>
    <td>-</td>
    <td>New. <a href="https://groups.google.com/forum/#!topic/puppet-dev/">Discuss TBD</a></td>
</tr>
</table>