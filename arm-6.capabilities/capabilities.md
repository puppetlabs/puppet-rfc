(ARM-6) Puppet Capabilities
===========================

Background
----------
There are multiple related problems in Puppet today:
* Module Reusability: It is difficult to build reusable modules that are not
  hardwired to their dependent or depending modules.  For instance, if one is
  building a mysql module and a related apache module, it is difficult to build
  them such that someone else can use the mysql module without also using the
  apache module.
* Dependency Injection: It is difficult to configure related services on
  multiple hosts at once.  For instance, if you want to build a database and
  web service on different hosts, you generally have to duplicate configuration
  to make it work.  Exported resources kind of make this work, but it is far
  harder than it should be.  In particular, there is no visibility of such
  dependencies, in general use of exported resources results in hard-wiring of
  those dependencies, and they are just not very easy to use.
* Orchestration: When running Puppet on related hosts, it is relatively
  difficult today to figure out the order in which to run Puppet.  That is, if
  you would like to orchestrate the deployment of related changes, you must
  essentially manually do it, and you almost certainly must use a separate tool
  to manage the ordering.

This proposal provides a single solution for almost all of them.  It does not
completely solve the Orchestration problem, but it provides all of the
information necessary to do so, and the proposal covers how one would do so.

As a specific example, given these definitions:
```
define db($port = 80, $user = root, $password, $host = "127.0.0.1", $database = $name) {
  notify { "db $name, password $password": }
}

define web($dbport, $dbuser, $dbpassword, $dbhost, $database) {
  notify { "web $name, password $dbpassword user $dbuser": }
}
```
and two hosts, ``web1`` and ``db1``, you would normally need to do this:
```
node db1 {
  db { one: password => "passw0rd" }
}
node web1 {
  web { one:
    dbport => 80,
    dbuser => root,
    dbpassword => "passw0rd",
    dbhost => db1,
    database => one
  }
}
```
Note that we're not just duplicating configuration here, we're having to
actually specify default values as hard-coded values in the web1 configuration.

**A note on terminology**: The term ``capability`` is less than ideal, and it
is considered temporary.  Ideally we will choose more appropriate terms before
we ship.

Proposal for Puppet Capabilities
--------------------------------
This proposal adds multiple parts to the Puppet ecosystem: A syntactical
addition, additional semantics to PuppetDB, and a new concept to the overall
architecture.  We also add the concept a catalog for an entire environment,
rather than tied to a specific host.

The key to the proposal is that, given two related resources (e.g., a Web
instance and a required Database instance), we would like to specify the
required resource and a dependency to it, and to have a) the requiring resource
be automatically configured to use the required resource and b) both resources
deployed appropriately on the appropriate host.  The mechanism for this
connection is a resource-like object type, which I am calling a ``capability``,
that abstracts the configuration of (in this case) the web service to use the
database service.

Taking our above example of db and web hosts, note that the key configuration
information can be abstracted into the key bits needed to configure a database.
That is, the information that the web server needs from the database server is
exactly and specifically the information needed to speak to the database.
Thus, we propose an abstract 'sql' type:
```
Puppet::Type.type(:sql) {
  newparameter(:name)
  newparameter(:port)
  newparameter(:user)
  newparameter(:password)
  newparameter(:host)
}
```

What we want is for the database specification to create an instance of this
resource type, and then some mechanism for the web server to discover this
instance and use it to configure itself, even if the two services are running
on two separate hosts.

Capability Creation
-------------------
The first challenge is how to get the database service to create an instance of
the ``sql`` capability.  We do that by adding an optional syntax to defined
resource types:
```
define db($port = 80, $user = root, $password, $host = "127.0.0.1", $database = $name) produces sql() {
  notify { "db $name, password $password": }
}
```
Note that we have slightly modified our ``define`` syntax; we now specify that
we ``produce`` "sql".  Note, also, that the parameters of our database resource
happen to exactly match the parameters of our sql resource.  This isn't that
surprising, but it makes one wonder why we even need this new concept.  In
answer to that, I ask, what do you do if you use postgresql, mysql, and oracle?
They each need the same parameters, but they would almost definitely be
implemented in different ways with different names.

By default, creating an instance of ``db`` does **not** result in the creation
of an ``sql`` instance.  Thus:
```
db { one: password => "passw0rd" }
```
behaves exactly as before.

However, if you modify this slightly to:
```
db { one: password => "passw0rd", produce => Sql[one] }
```
then an instance of the Sql resource type will be created.  Ok, so we've got
this abstract resource; how do we use it to configure our web server?

Capability as Configuration
---------------------------
We're going to start with the simpler case where both web and database servers
are running on the same host.  We'll solve the harder case of them running on
separate hosts next.

In this case, we need to "teach" the web service how to configure itself using
an sql instance.  To do this, we introduce a new syntax for mapping the sql
parameters to the web parameters:
```
define web($dbport, $dbuser, $dbpassword, $dbhost, $database) consumes sql(
  $port      = dbport,
  $user      = dbuser,
  $password  = dbpassword,
  $host      = dbhost,
  $database  = database
) {
  notify { "web $name, password $dbpassword user $dbuser": }
}
```
Note that we can't just use exactly the same parameter names.  Both web and
database servers specify ports, for instance.  You could build modules that did
map directly, but that would be another case of hard-coding the connection
between your modules; you could use modules with this mapping, but few other
people would be able to.

So, this syntax tells us how to take the sql server's port and convert it to
our web server's dbport attribute.  How do we use this in practice?
```
web { one: consume => Sql[one] }
```
This specifies that Puppet should find the 'one' instance of the 'sql'
capability and use its parameters to configure itself.

Note that the mapping presented here can also be used to map during the
'produce' syntax; if your defined resource uses a different parameter than the
capability, then you map just like above.

Look how compact our configuration is here.  Almost no duplication (just that
of the name of the sql instance), and certainly no repetition of default values
or hard-coding of dependencies.

You could also skip the sql instance entirely and configure the services by
directly specifying a relationship:
```
db { one: password => "passw0rd" }
web { one: require => Db[one] }
```
In this case, we don't create an sql instance, but the compiler knows how to
use the db instance to configure the web instance.

Configuring Separate Hosts
--------------------------
Now, how do we configure services spread across multiple hosts?

The key to this solution is understanding that it is unlikely within a given
infrastructure to need to have multiple capability instances with the same
name.  That is, you are unlikely to have many databases with exactly the same
name, and unlikely to have multiple web servers with exactly the same name.  In
fact, one could argue that such an occurrence would be a bad thing.

So, I propose that capability instances be unique, by type and title, across a
given environment.  That is, using our above exactly, Sql[one] would uniquely
identify that instance across whichever environment it is in.  This way, you
don't have to worry about complicated querying, exporting, or anything else.
If you create a capability, then it's global to that environment.

With this, we just put all of the capabilities into PuppetDB, like we do the
rest of the catalog, and make sure 1) we can distinguish capabilities from
other resources and 2) we test on catalog insertion that we're not duplicating
capabilities.

Crucially, this begins to treat an environment analogously to single host - it
has a set of resources which are related to each other and must be unique
within that environment.  We can also view the graph for that environment, just
like we could view the graph for a given host.

In this situation, the code to configure the separate hosts looks like:
```
node db1 {
  db { one: password => "passw0rd", produce => Sql[one] }
}

node web1 {
  web { one:
    consume => Sql[one]
  }
}
```
The configuration is just as compact as when they're on a single host.

Cross-host Orchestration
------------------------
Given that you can now graph the relationship between resources across any
number, you can now easily figure out the appropriate order in which to run
your hosts.  Given the above dependency, you know you must first configure your
database server, and then your web server.

Just like a given Puppet client uses the graph to apply its resource set in
the appropriate order, a new service (probably based on mcollective) would use
the new environment-wide catalog to run Puppet in turn on each host.  Many
hosts would not be related and thus could be run in parallel, but those that
had a specified dependency could be ordered appropriately.

Service List
------------
The capabilities generated and consumed by the various hosts can essentially
function as a list of all of the "services" a given environment requires and
provides.  This means it is relatively straightforward to move from this list
to configuring monitoring, for example.

Alternatives and Modifications
------------------------------
A desire to be able to extract values from any resource, not just special
capabilities has been expressed.  To me, it's unclear how to do this without
providing a far more complicated syntax for mapping attribute names, which is
the critical bit.  The closest I can see is to allow mapping in all cases,
but to only allow defined resources to declare the mapping.  For instance,
I could see these resources automatically mapping:
```
define db($dbport) { ... }
define web($dbport) { ... }
db { foo: dbport => 50 }
web { foo: require => Db[foo] }
```
In this case, both definitions use the same parameter, so you could
automatically map from one to the other.

You'd then only need to use this syntax to either provide resource mapping or
to specifically declare the availability of a capability.

I think this might actually be more confusing for most people, but it seems
to match what others have asked for.

Concerns
--------
The primary concern I've heard about this proposal is that capabilities are
inherently global.  It has been expressed that it would be better if users had
to explicitly mark instances as available to other hosts, rather than
defaulting that way, and in fact, not being able to restrict them to a single
host.  One of the things I specifically like about this proposal is that it
begins to treat environments like a kind of über-host, with a catalog, and the
same uniqueness requirements on a catalog.  The whole concept of exporting has
always been challenging, and I think this would allow us to remove it.
