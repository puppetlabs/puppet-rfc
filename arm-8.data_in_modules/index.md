Data in Modules Summary
=======================

Problem Statement
-----------------

Currently, module authors use a 'params class pattern' to provide defaults for
the parameters their classes accept. A module for managing a database service
`mydb`, for example, will provide a `class mydb::params`, which contains
parameter assignments like `$mydb::params::tcp_port`,
`$mydb::params::install_dir`, etc. These assignments can use the puppet DSL
for conditional logic so that the `install_directory` follows different OS'
filesystem conventions appropriately. The parameter values are then used in
the module's other classes, either in the prototype for the class or directly
in the manifest:

    class mydb::packages (
      $tcp_port    = $mydb::params::tcp_port,
      $install_dir = $mydb::params::install_dir,
      ...
      ) {
        # class resources go here
      }
 
This approach has some drawbacks (summarized from user comments on [#16856][16856]:

* it lives apart from hiera, which we've landed on and promoted as being the
  solution to the data/code separation problem

* it’s an attempt to supply default data to your classes and falls over
  whenever someone wants to change data or provide their own for their site.
  They’re forced to either run with a modified module or commit changes
  upstream, which may contain private data. (Ryan Coleman)

We'd like to build a simple, consistent interface for data in modules that
ties into our larger data/code separation story.

User Stories
------------

   As a module author,
   I want to separate data from code in my Puppet modules,
   so users can add them to their infrastructure without forking/patching locally.

   As a module user,
   I want modules I use to have a consistent interface to their parameters,
   So I don't have to re-learn how to use every new module I download.

   As a site administrator,
   I want the modules in use at my site to integrate with my infrastructure data,
   so I can easily adapt them to changing business conditions.

Spike Solution
--------------

RI wrote an implementation which provides an instantiation of hiera's yaml
backend inside a module. On the filesystem, a Forge module which used this
implementation would be laid out like this:

    modname/
      Modulefile
      README.md
      data/
        hiera.yaml  # configures a hierarchy of:
                    # osfamily/%{osfamily}.yaml
                    # default.yaml
        default.yaml
        osfamily/
          redhat.yaml
          debian.yaml
      manifests/
        init.pp
      [etc]

The hiera module backend uses the hiera.yaml for variable lookups implicitly
on each `hiera('modname::varname','defaultvalue')` function call or
automatically via data bindings.

The use of the `data/` directory and the normal hiera conventions for yaml
file layout are good points here, but working with the proposed branch brought
up a couple of problems that the final solution must address:

* The separate hiera.yaml configuration file lacks integration with the
  existing module metadata. The config should instead live inside the
  'modulefile'. This would provide a better user experience for both
  publishers and module users.

* If we put data in modules, we need to remove pluggability from data
  bindings, otherwise the module data is no longer guaranteed to be available.
  Therefore the data bindings have to always go through hiera+hiera-in-modules
  and people can plug in their own hiera backends if they require customized
  data storage.

* Lookup policies (merge/append vs single-value override) are implemented in
  the hiera parser functions through the use of differently-named functions.
  Since data bindings are implicit/automatic and do not provide a way indicate
  the lookup policy for values, some other mechanism needs to be provided.
  This is a general data-bindings problem not a specific issue with D-i-M, but
  a good solution will address both cases.


[16856]: http://projects.puppetlabs.com/issues/16856