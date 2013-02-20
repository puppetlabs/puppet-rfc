Evaluation / Decision Support
=============================
There are a number of important questions to resolve about this proposal:

* Should the solution be part of Puppet, like this is, or a separate tool that integrates with Puppet?

* Should the solution be as magical as this one is, such that Sql instances (in our examples) are created for you, or should the user be forced to specifically create them?

* Should capabilities actually be required for this kind of dependency injection to work, or should any resource be usable?

* Should the capabilities have to be described (either for production or consumption) in the definition, or should they somehow be described as part of specifying the relationship?

The current proposal assumes an answer to all of them:

* The solution must be a part of Puppet.  We could not conceive of a solution that didn't essentially exactly duplicate the objects and semantics already present in Puppet resources and graphs, which did not make sense.  Note that the orchestration itself -- that is, the process of triggering a set of hosts to run in the right order -- does actually need to be done by a separate tool, but all of the semantics and data are in Puppet.

* This solution is too magical.  It's unclear at this point how to make it less so.

* It would be nice to allow any resource to be used like this, but it's unclear how to do so.

* If the capability production and consumption are not hard-coded in the definition, then the same information must be provided every time a relationship is created.  In addition, you could not automatically determine information like whether a web server needs a database.  Thus, this solution states that it must be hard-coded.
