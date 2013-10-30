ARM-17: Module Namespaces
=========================

Summary
-------

Modules currently are namespaced simply on the name of the module resulting in
conflicting situations where users have more than one module with the same
name. This proposal would allow modules to have a unique addressable namespace
similar to libraries of other languages.

Goals
-----

Allow multiple modules with the same module name to exist on a single puppet
master and be referenced explicitly or implicitly based on import statements.

Non-Goals
---------

This left blank.

Success Metrics
---------------

A single manifest can declare classes, defines, and types from explicitly and
implicitly addressed non-unique module names when the modules include
conflicting and non-conflicting class/define/type names.

Motivation
----------

This work would allow projects which depend on separate versions of similar
modules to co-exist on a single puppet master when compiling a single catalog.
Examples would include a client wanting to use the latest version of the
PostgreSQL module without conflicting with PE's own PostgreSQL module. It would
also provide the ability to use the types & providers from the
`puppetlabs-mysql` module, but use classes and defines from `theirown-mysql`
module.

Description
-----------

Any given module on the Forge has a namespace that is the user account under
which it is published. Two nearly identical or entirely different modules may
be uploaded with the same name under two different namespaces, but the classes
and defines of only one module may be used on any puppet agent environment at
any one time, and types and providers are artificially limited to only a single
puppet master per type.

As this namespace is not currently used for any other purpose than the Forge,
we can leverage it to allow other manners of Puppet DSL declaration
manipulation previously unavailable. We can use examples from other languages
and namespace-addressable class/object/functor import statements to design a
fit for puppet modules. Notably Python, Haskell, and Clojure.

- [Haskell's `import`](http://www.haskell.org/haskellwiki/Import) allows for
  qualified, unqualified, and differently-qualified imports, as well as
  exclusive or inclusive imports.
- [Python's `import`](http://docs.python.org/3/tutorial/modules.html) allows
  for qualified and unqualified imports, as well as exclusive imports.
- [Clojure's `require` and
  `use`](http://blog.8thlight.com/colin-jones/2010/12/05/clojure-libs-and-namespaces-require-use-import-and-ns.html)
  allows for qualified, unqualified, and differently-qualified imports, as well
  as inclusive and exclusive imports.
- Puppet currently allows only unqualified imports with respect to module
  namespaces.
- Ruby currently is crazy. Let's ignore it.

Adaptations from these languages should evaluate qualified, unqualified,
differently-qualified, exclusive, and inclusive imports. It should look at the
effect of per-scope, per-manifest, or global imports, and backwards-compatible
defaults for the current system.

### Categories of imports

#### Qualified imports

- `import qualified puppetlabs-mysql` would import all classes, defines, and
  types of the module `mysql` in the `puppetlabs` namespace into the local
  scope as the qualified `puppetlabs-mysql` namespace. This will cause classes
  from the module to be references as `puppetlabs-mysql::server` and types to
  be `puppetlabs-database_user`.

#### Unqualified imports

- `import puppetlabs-mysql` would import all classes, defines, and types of the
  module `mysql` in the `puppetlabs` namespace into the local scope as the
  unqualified `mysql` and qualified `puppetlabs-mysql` namespace. This will
  cause classes from the module to be references as `mysql::server` and
  `puppetlabs-mysql::server` and types to be `database_user` and
  `puppetlabs-database_user`.

#### Differently-qualified imports

- `import puppetlabs-mysql as pl-mysql` would import all classes, defines, and
  types of the module `mysql` in the `puppetlabs` namespace into the local
  scope as the qualified `pl-mysql` namespace. This will cause classes from the
  module to be references as `mysql::server` and `puppetlabs-mysql::server` and
  types to be `database_user` and `puppetlabs-database_user`.

- `import qualified puppetlabs-mysql as pl-mysql` would import all classes,
  defines, and types of the module `mysql` in the `puppetlabs` namespace into
  the local scope as the qualified `puppetlabs-mysql` namespace. This will
  cause classes from the module to be references as `pl-mysql::server` and
  types to be `pl-database_user`.

#### Exclusive imports

This import mode is for including explicit classes/defines/types only and
excluding any others in a referenced module namespace.

- `import puppetlabs-mysql (mysql::server, database_user)` would import only
  the `mysql::server` class as `mysql::server` and `puppetlabs-mysql::server`
  and `database_user` type as `database_user` and `puppetlabs-database_user`.

- `import puppetlabs-mysql as pl-mysql (mysql::server, database_user)` would
  import only the `mysql::server` class as `mysql::server` and
  `pl-mysql::server` and `database_user` type as `database_user` and
  `pl-database_user`.

- `import qualified puppetlabs-mysql (mysql::server, database_user)` would
  import only the `mysql::server` class as `puppetlabs-mysql::server` and
  `database_user` type as `puppetlabs-database_user`.

- `import qualified puppetlabs-mysql as pl-mysql (mysql::server,
  database_user)` would import only the `mysql::server` class as
  `pl-mysql::server` and `database_user` type as `pl-database_user`.

#### Inclusive imports

This import mode is for explicitly excluding some classes/defines/types and
implicitly including any others in a referenced module namespace.

- `import puppetlabs-mysql hiding (mysql::server, database_user)` would not
  import the `mysql::server` class or `database_user` type and functions the
  same as the unqualified imports statement above for all other
  classes/defines/types in the module namespace.

- `import puppetlabs-mysql as pl-mysql hiding (mysql::server, database_user)`
  would not import the `mysql::server` class or `database_user` type and
  functions the same as the differently-qualified imports statement above for
  all other classes/defines/types in the module namespace.

- `import qualified puppetlabs-mysql hiding (mysql::server, database_user)`
  would not import the `mysql::server` class or `database_user` type and
  functions the same as the qualified imports statement above for all other
  classes/defines/types in the module namespace.

- `import qualified puppetlabs-mysql as pl-mysql hiding (mysql::server,
  database_user)` would not import the `mysql::server` class or `database_user`
  type and functions the same as the differently-qualified imports statement
  above for all other classes/defines/types in the module namespace.

#### Current system and backwards-compatibility

No import statement currently exists. The behaviour of class/define/type
references is only unqualified and is handled by the autoloader. This behaviour
may continue in the absence of a given imported module, but in the event that a
module *is* imported unqualified, the imported module will be used instead of
the autoloader.

### File paths

Modules will continue to be in the `modulepath` directories deliniated by a
colon, but any modules that are usable with {{import}} must be named
{{namespace-modulename}} much in the fashon that the puppet module tool used to
do. Module versions are not to be taken into account in the names of the module
  directories.

### Other import considerations

- If an import statement would import classes/defines/types and references a
  namespaces that does not exist, it will import the given items into the new
  namespace.

- If an import statement would import classes/defines/types that are already
  imported, a duplicate import failure would be given.

- If an import statement would import classes/defines/types that do not
  conflict with any currently-imported items but references a namespace that
  already exists, it will merge the imported items into the existing namespace.

- If an import statement would not import any classes/defines/types and the
  module with a given namespace and module name does not exist, it will throw
  an error.

- If an import statement would not import any classes/defines/types but the
  module with a given namespace and module name exists, it will not throw an
  error.

- Import statements operate on the local scope, thus the top scope may be
  preferable for performing imports as opposed to sub-scopes

Testing and Evaluation
----------------------

I think the ability for multiple modules in separate namespaces with the same
modulename that use several different concat modules to construct files with
contents assembled by compatible concat modules would be a significant
acceptance test.

Alternatives and Recommendation
-------------------------------

Alternative implementations would evaluate different scope import styles and
have not yet been heavily evaluated.

Open questions:
- Should imports operate on the top scope or the local scope?
- Should module designers leave it to the implementors to declare included
  modules, or should they declare them?
- Would module designers need to add top-scope imports?
- What would conditional imports look like, and how would they promote or
  inhibit good module hygiene?

Risks and Assumptions ---------------------

This could either significantly increase or significantly decrease the
confusion around extension or implementation of modules.

Impact ------

- Other Puppet components: It will impact any components that would be impacted
  by the future parser, or any components that take advantage of the
  autoloader.
- Compatibility: If the abilities are not used, it should be backwards
  compatible.
- Performance/scalability: Unknown, but potentially.
