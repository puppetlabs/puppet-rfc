Implementation
==============
Luke has built a prototype implementation, at:

https://github.com/lak/puppet/tree/prototype/master/capabilities

It implements this syntax:
```
define db($port = 80, $user = root, $password, $host = "127.0.0.1", $database = $name) produces sql() {
  notify { "db $name, password $password": }
}

define web($dbport, $dbuser, $dbpassword, $dbhost, $database) consumes sql(
  $port      = dbport,
  $user      = dbuser,
  $password  = dbpassword,
  $host      = dbhost,
  $database  = database
) {
  notify { "web $name, password $dbpassword user $dbuser": }
}

node direct {
  db { one: password => "passw0rd" }
  web { one: require => Db[one] }
}

node indirect {
  db { one: password => "passw0rd", produce => Sql[one] }
  web { one: consume => Sql[one] }
}

node db1 {
  db { one: password => "passw0rd", produce => Sql[one] }
}

node web1 {
  web { one:
    consume => Sql[one]
  }
}
```
This is a simple prototype that shows how you can build related database and web definitions, where the web definition requires the database definition and must be configured by it.  We show three ways of using it:

* A direct relationship, that looks essentially identical to how Puppet works today, except that the database definition's configuration details do not need to be duplicated in the web definition's specification.

* An indirect relationship, where an Sql capability is created by the database definition, which is then subsequently used by the web definition.  These are still on the same host.

* A cross-host relationship, that otherwise looks like the indirect relationship.  The database server creates the Sql instance, which is then stored in PuppetDB, and the web server looks it up by name and uses it to configure the web definition.

These can be run using the above-linked code by running a puppet master backed by puppetDB, and then running the agent multiple times with different ```--certname``` calls.  The code is in bin/xhost.pp in the prototype.

The prototype also has a script at bin/xhost.rb that will generate a graph (in dot format) of all of the related applications.
