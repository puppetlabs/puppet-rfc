ARM-8 Hiera 2
=============

Summary
-------

> TODO:  
> REQUIRED -- Provide a one-paragraph summary of the proposal, no more
> than a few sentences.  This summary will be rolled up into feature
> lists and other documents, so please take the time to make it short
> and sweet.

Goals
-----

> TODO: Tenative goals below - to be defined/edited (@eric0)  
> What are the goals of this proposal?

* Provide injection capabilities at par with or exceeding the capabilities of current Hiera directly in the Puppet Language that
  can be the foundation for all injection like behavior in Puppet.
* Be backwards compatible with Hiera to protect users investment and offer a migration path.
* Offer as much backwards compatibility as possible when introduced (exception: behavior that it is undesirable) where users
  can gradually opt-in to use new features.
* Provide a mechanism that makes it possible to plugin functionality to defined extension points to aid in the process
  of breaking up the monolithic puppet into separate pieces, and to define internal APIs.
* Provide injection like behavior that can operate with different configuration per environment
* Clearly define the chain of responsibilities for processing a request wrt. environment selection.
* Make selection of environment in Puppet Language possible.
* Be backwards compatible with ENC and node declarations.
* Provide an extension point for ENC (new API).
* Implement support for current ENC as an extension.
* Provide the foundation for x-node injection.
* Solution should allow users to be able to define a review/authorization process of configuration - manual or via CI.
* Provide a richer set of user organization business driven configuration selection criteria than the current node / ENC mechanism.
* Data structures interchanged via APIs should be made in a fashion that is transferable between different implementation
  languages with preservation of a high degree of semantics.

Non-Goals
---------
> TODO: W.I.P  
> Describe any goals you wish to identify specifically as being out of
> scope for this proposal.

### Data Transformations

Whenever data is processed there is usually also the need to perform data transformations; all data is simply not
in perfect form for all use cases all the time. Although tempting to bake data transformations into the
injector/binding framework this makes it less extensible. It is therefore not a goal for Hiera2 to also define
such data transformations - instead Hiera2 defines ways in which existing data transformation can be called
upon when needed. If data transformation power is lacking for a particular use case, this should be added as
specific functions.

### Review/Authorization Process

It is the goal of this ARM to offer a solution that is easily integrated into a review and authorization process defined
by a user organization. It is not the goal to define any such processes - only the basic mechanisms on which it is based.

### Secure Facts

It is not a goal for this ARM to define how secure facts may be handled/implemented.
(This ARM does not depend on such facts to be available, but discusses their importance and where they surface).

Success Metrics
---------------

> TODO:  
> If the success of this work can be gauged by specific numerical
> metrics and associated goals then describe them here.

Motivation
----------
> TODO: The goals need motivation - W.I.P  
> (There are issues reported for many of these...)  
> Why should this work be done?  What are its benefits?  Who's asking
> for it?  How does it compare to the competition, if any?

### Cleanup of Environment selection

* Dynamic environments:
  * not possible to enumerate them
  * ...
* Fuzzy rules in the chain to determine environment
* Security issues
* Inefficient and error prone / fuzzy handling of pluginsync (may require expensive iterations)
* Not possible to get indication/error if requested environment becomes the effective environment
* Not possible to base environment selection on facts (except by cheating, which is done in the wild)

Clear rules regarding environment selection are essential in order to have a solid process for the next
level of configuration since it is environment specific (per the environment itself and the modules on its
modulepath).

### Provide a mechanism that makes it possible to plugin functionality to defined extension points

Lack of definition of runtime extension points makes it difficult to plugin behavior (in Ruby) other that functions
and types/providers which have well reasonably well defined APIs (internal DSLs). One example of a simple extension point
(used as an example in this ARM) is the support for syntax checkers that is proposed in ARM-4 Heredoc where the only reasonable
option was to implement the syntax checker as a function; in this case not terribly bad since it may be of value to also
call such a function from user code. But other such plugins would not make sense to expose to users - several examples
are presented in this ARM.

This feature is important as it offers distinct APIs for extensions that can be delivered in modules with a minimum of effort.
Having clear APIs allows shipment of basic functionality in the open source version while making it possible to offer
commercial alternatives, as well as the possibility for partners to provide integration.


Description
===========
Hiera2 is an injection service catering to a variety of needs related to "dependency injection", ranging from simple
lookup of data to advanced cases of implicit bindings, mappings, and binding of runtime services. The principles for
the Hiera2 implementation are:

* to provide a common injection mechanism that can replace the current disjunct handling of injection-like behavior 
* to express all configuration in the Puppet DSL language to provide a consistent user experience
* to base the technology on a common Model-Driven Engineering approach for robustness and interoperability.

An Introduction to Dependency Injection
---------------------------------------
The term "Dependency Injection" (or just "Injection") means that constructs which form static/hard dependencies between
logical parts of a system are broken out to form a separate concern. A basic example is data injection: a module
needs a piece of data passed to it as a parameter, but it can not define exactly how this data is obtained -- if it did, that 
would form a hard dependency on the behavior how to obtain the data (which would be brittle), not simply on the data 
(which is desirable).  Likewise, another module that makes use of the first module may also be used by a top level construct
like a `node` declaration which needs to influence how an intermediate module passed parameters to the first module.
As you can imagine, if this were implemented with only parameter passing, the amount of parameters that would need to be exposed
at the top level would be staggeringly complex.

When we separate the concern and break the dependencies into the various aspects of the system we can use a "divide and
conquer" strategy where we are given powerful control of the system's composition in terms of its data and behavior ranging
from the most general (common/global) to the most specific (a specific point in the logic).

An injection system consists of bindings, which attach a *key* to data and/or behavior; a mechanism to compose such 
bindings with regard to scopes, precedence, and transactions; and a mechanism to
obtain the bound data/behavior when it is needed. The basic injection framework is then used to build higher order services.

At the core, the binding system binds a key that consists of a Type/Name combination that uniquely identifies a point of
injection. The system defines certain "special" names (typically for implicit injections done automatically if a
binding exists for that key), and a user may define other keys for implicit injection. As an example of implicit
injection, Puppet 3's "Data Bindings" provide such a service today by automatically looking up the parameter 
`ipaddress` for a class named `ntp::server` in a hiera data store as the key `ntp::server::ipaddress`.

Further, the binding system binds the key to a *producer* of the bound value; the simplest producer handles literal values such
as numbers, strings and other simple data types as well as structures (arrays and hashes) of data types. Other types of producers
may do early or late binding to more advanced data objects, create connectors to other systems, allow parameters to be passed
and much more.

To carry on the example above, the current injection system uses Hiera's configurable backends to determine how,
and in which order, a particular binding ought to be resolved; many users search first in the encrypted `hiera-gpg` backend
for secure data like database passwords, then fall through to an unencrypted YAML backend if no encrypted key is found. In
contrast Hiera2 separates the concepts "backend" (see [Bindings Provider](#bindings-provider)) and
"producer" (see [Producer](#producer)).

Examples of Current Injection-Like Behavior in Puppet
-----------------------------------------------------
Puppet already has some injection like mechanisms, some (like hiera) perform "true injection", and others deal with
similar problems with varying degree of success in separating caller and called logic and providing detailed control
of the mechanism:

* Hiera 1 -- injects data, classes and parameters for parameterized classes, and does explicit injection.
* `puppet.conf` and command line settings -- inject data configured per environment.
* Resource defaults -- inject values to resources which don't provide more closely-scoped overrides.
* ENC -- injects environment, classes with or without parameters, and top-scope variables.
* `site.pp` -- "injection" of classes and variables based on node name matching (a form of categorization).
* Spaceship operators that query virtual or exported resources and "inject" results.

Hiera2
------
Hiera2 is a new implementation based on the ideas in Hiera and general injection frameworks such as 
[Google Guice](http://code.google.com/p/google-guice/), which is the leading injection framework for Java.

Hiera2 consists of the following parts:

* [Categorization of Requests](#categorization-of-requests)
* [Bindings](#bindings)
  * [Bind Operations](#bind-operations)
  * [Producer](#producer)
* [Composition of Bindings](#composition-of-bindings)
  * [Bindings Providers](#bindings-provider)
* Explicit Injection
* Implicit Injection
* API for use in Ruby
  * Bindings Provider extension
  * Producer extension
  * Naming Scheme
* API for Puppet Extension Points
* Models (used by framework and for system interchange)

Categorization of Requests
--------------------------
The Puppet System needs to be able to create and configure an injector at the start of processing a request (and here we mean 
a request in the most general sense: asking the system to do something). This injector is then used throughout the servicing of
that request.

Since we want to be able to express bindings hierarchically (searching for something specific first, then falling back to
more generally applicable defaults), and we want users to be able to define their hierarchy based on their organization's needs,
we require a flexible, user-configurable framework. In Hiera2 this is called Categorization and it is expressed using the
Puppet DSL. Before jumping into the details,
examples of categories are "data_center", "rack", "physical_hw", "security_zone", as well as the built in categories
for "node", "environment", and "common".

Before explaining categorization (which may vary per environment) we must first define how the system establishes
which environment to use, and so we'll first look at the sequence of processing for a catalog request.

### Request Sequence

In Puppet 3.x the sequence of setting up and processing a catalog request to the point where evaluation in the 
root (site.pp) manifest takes place is somewhat complex and includes murky areas. As a side effect of implementing 
Hiera 2 it is of great value to also clean this up. The proposed sequence is as follows:

* Request reaches new node terminus.
* All facts are turned into top scope variables (or, optionally, put into a `$facts[]` hash).
* The `$environment` variable if set is renamed to `$agent_environment` (it is unsafe in its current form) - see
  [Environment from Agents](#environment-from-agents) for the specifics.
* If the request contains trusted data (from cert or encrypted facts), these are made available using a mechanism
  that is to be specified separately (i.e. via a different mechanism than just setting `$environment` in top scope).
* If an external (current API) ENC is configured it is called to obtain environment, top scope variables, class parameters
  and classes data, these are processed a special way - see [Integration with Existing ENC](#integration-with-existing-enc).
* Settings refer to a single manifest that enumerates the available environments, and for each environment the modulepath and
  environment root manifest.
* The entry point manifest can set `$environment`.
* If `$environment` is not set, it defaults to "production".
* If the entry point manifest is not present, the environment defaults to "production".
* If nothing else was specified, the execution delegates to the `site.pp` in the same location as the entry point manifest.

### environments.pp

The entry point manifest is named `environments.pp` and is placed in a directory appointed by `puppet.conf` (defaults
to `$confdir`)

> ##### Use hardcoded environments?
> Decision:  Should `environments.pp `be a hard-coded name, or be configurable in puppet-conf?
> Pro hard-coded: less confusion
> Pro configurable: easier to test different overriding configuration (dev/test)

The concrete syntax for an environment is specified by borrowing the resource syntax. It is made special by
being placed inside of a `site { }` expression (these _environment resources_ are not placed in the catalog).
All paths except `envdir` may be relative (this is explained in more detail below).

    site {
      # Environment defaults
      Environment {
        envdir      => ... # absolute path to environment's root, defaults to '$confdir'
        modulepath  => ... # colon separated paths, may mix absolute and relative paths, defaults to '<envdir>/modules'
        manifestdir => ... # path to directory containing manifests, may be relative, defaults to 'manifests/'
        manifest    => ... # path to environment root manifest, may be relative, defaults to '<manifestdir>/site.pp'
        templatedir => ... # path to templates directory, may be relative, defaults to 'templates/'
        bindingsdir => --- # path to root of bindings directory, may be relative, defaults to '<envdir>/bindings'
      }

      # A static environment
      environment { 'production':
        # ...
      }
      # Additional static environment
      environment { '...':
      }
      
      # Allowing dynamic environments, and specifying defaults if different than Environment defaults
      dynamic_environments { '/etc/puppet/environments':
        # envdir is automatically set and cannot be changed (error if attempted)
      }
    }
    
    $environment = 'production'

Here is an example where dynamic environments are turned on and then put to use. A dynamic environment comes into
existence by a user that checks out a branch; e.g. `/etc/puppet/environments/test36' becomes the environment called
`test36` with the configuration shown below). The selection of environment sets the `envdir` to either the
static `$confdir/master` if "production" was selected, or one of the dynamic environments if the selected environment
name matches a a name of a subdirectory of the `dynamic_environments` specified directory; e.g. `test36` 
The relative paths are resolved against the effective `envdir`.
(A relative path in `manifest` is however first resolved against `manifestdir` which
in turn is resolved against `envdir` if relative. This is done to enable just specifying the name of the manifest).
For the statically declared `production` environment the `envdir` must be specified (if it should be something other
than the default `$confdir`).

    site {
      Environment {
        modulepath => 'modules/dist:modules/site:/opt/puppet/share/puppet/modules',
      }
      dynamic_environments { "$confdir/environments":
      }
      environment { 'production':
        envdir     => "$confdir/master",
        modulepath => '$confdir/modules:modules/site:modules/dist',
      }
    }

> Note: The rationale for things inside the new top level keyword `site` borrowing syntax, but being
> special kinds of resource, and that the `site` top level keyword contains other new keywords that are not
> special outside of "site". This reduces
> potential clashes with existing logic. It also logically relates these kinds of statements with other
> statements about the "site" (as you will see in later sections). (Remember that we want one parser to
> be able to parse older versions of the puppet language.)
>


The following rules apply:

* It is no longer possible to introduce stanzas in `puppet.conf` for environments.
* Restriction on environment names is lifted
* Any number of environments may be used; statically declared or (and if enabled) dynamically placed under one directory
  specified by the _environment resource_ `dynamic_environments`.
* Environment resources are not placed in the catalog.
* The `--environment` flag is _by default_ ignored from agents. See [Environment from Agents](#environment-from-agents),
  [Integration with existing ENC](#integration-with-existing-enc) and [Request Sequence](#request-sequence) for details
  and discussion.
* Environments incorporate `Environment` resources defaults from `environments.pp` in addition to default puppet values
  for all parameters derived from `puppet.conf` (i.e. where defaults/defaults are set if factory settings are not
  wanted - but this is really not needed since user has full control in `environments.pp`).
* If `environments.pp` is absent, a default "production" environment is automatically defined.
* If `environments.pp` is present but does not define a "production" environment and an environment by the name of "production" is not
  available (and allowed) as a dynamic environment then one is automatically defined.
* If execution reaches the end of the `environments.pp` and `$environment` has not been defined, it is set to "production".
* A `dynamic_environments` environments resource may be defined with a title of an absolute directory path. This allows
  dynamic environments to be used and specifies that additional environments are available in its sub-directories named
  after the environment they represent.
* If `$environment` is set to a value other than one of the statically-defined environments in `environments.pp`,
  or (if allowed) a subdirectory of `dynamic_environments`, an error is raised ("unknown environment") and processing stops.
* In case `$environment` is successfully resolved against an existing directory the variable `$envdir` is determined;
  it must be explicitly set in a statically environment (or set from `Environment` defaults), and it is automatically
  set when `$environment` is successfully resolved to a dynamic environment.
* Then `$manifest` is resolved against the selected environment resource (relative paths resolved against `$envdir`)
* The `$modulepath` is resolved against the selected environment resource (relative paths resolved against `$envdir`)
* (Absolute paths remain absolute).
* The RHS parameter values may be any non top level puppet expression (just as for regular resources), but may not use injection
* Facts and secure data have been bound to variables when evaluation of `environments.pp` starts
* With the exception of `$environment`, any variables set in `environments.pp` are set in a local scope not visible to the
  rest of the system.
* The `environments.pp` may contain other puppet expressions - the use case is to support data manipulation, common
  definitions reused in several environment specifications, etc.
* Only functionality available in puppet core may be used (functions and plugins/extensions) since the environment has not been setup, and
  hence no module-path.
* `environments.pp` is reevaluated on each request (if changed) as the master would otherwise need to be restarted.

> ##### Use magical return?
> Discuss: Since the new evaluator will return the value of the last executed expression as the result, we could simply
> return a value that way, but it is slightly magical in this context.  
> Likely resolution : `$environment` is picked up, no magic.

> ##### Use literal default for environment parameters?
> Discuss: A parameter value of `default` could be used to initiate the parameter to a sane/typical default in an
> environment.

> ##### Discuss: Is `$envdir` of value later in the process?
> Or should it simply be used internally when resolving relative paths in `environments.pp`?

### Environment from Agents

It is of value to be able to set the environment on an agent using `--environment <env>`. There are however
issues with this approach:
* The setting is not secure
* The setting is not authoritative (an ENC may override it)
* If overridden by the master (ENC or `environments.pp`), there is no warning (or error).

For those that use classic node statements (no ENC) the environment set on an agent will not be overridden.

It is proposed that the environment set using `--environment` on an agent should be made available when evaluating `environments.pp`
under a different name e.g. `$agent_environment`. If `--environment` has not been specified, the value of `$agent_environment` is
`undef`. The logic in `environments.pp` can then resolve the various cases:

* ignore the `$agent_environment` completely
* compare it against ´$environment` and if different issue a warning or raise an error
* use it if there is a desire to override the decision in an ENC
* use it in combination with other data supplied from the agent to determine the environment to use.

In simple terms: the environment set on an agent does not automatically become the `$environment` that will be used, while
giving a user full freedom to do what they want.

### Integration with existing ENC

The use of the current ENC API requires special handling as it is believed that users
that have an ENC will want to use it in parallel with the new functionality until they have completely migrated. The current
ENC may be deprecated, or re-purposed.

The ENC can return a list of classes, parameters for classes, top scope variables, and the environment to use.
If these are produced, a local scope is created, and the ENC provided data that are of value during the evaluation of
`environments.pp` are bound to corresponding variables (e.g. any given top scope variables, `$classes`, and `$environment`).
Yet another local scope (with visibility into the local scope created for the ENC (if any)) is created where the
evaluation of the `environments.pp` body takes place. It is thus possible to refer to the
ENC set top scope variables and `$environment` or set them to a new value.
The resulting `$environment` is the visible value at the end of the evaluation
of `environments.pp` (before the two scopes are abandoned), all other local variable values are discarded.

The new implementation is best made as an extension point that defines a new API for ENC (enables a new ENC to be written
that has access to facts). An implementation (adapter) is made that adapts to the existing ENC.

> ##### Provide access to $classes and $parameters in environments.pp?
> Discuss: It is probably of questionable value to have access to the defined classes since environments.pp only
> influences the selection of an environment. Likewise, class parameters are of little value.

The binding of classes to nodes are done using the Hiera2 Layering mechanism;
see [Composition of Bindings](#composition-of-bindings) and [Enc Bindings Provider(#enc-bindings-provider).

> ##### Top scope variables - how?
> Discuss: Inject them from vars set in environements.pp, as named bindings, provide other mechanism (bind variables using bind operations)?
> The handling of top scope variables needs further thought/discussion. Top scope variables
> are bad in general because they may be introduced without being declared in Puppet Logic which makes static analysis
> of code referencing top scope variables impossible.
> We can continue in this style - say by making all variable assignments in `environments.pp` be top scope
> assignments (not so good), and simply mix in variables from the ENC, or change them into named bindings allowing them
> to be composed (see [Named Bindings](#named-bindings)), or have concrete syntax (probably the best).

### Categorization

When the system has finished computing `environments.pp` it delegates to the manifest specified by the selected
environment's `manifest` property. This allows further configuration of the [injector](#term-injector) that will be created for
use when servicing the request.

The categorization syntax looks like this:

    site {
      categories {
        node    => "$::fqdn",
        virtual => "$::is_virtual"
      }
    }

The list of categories specifies two things; the _order_ defines the precedence (in descending order, first has
highest precedence), and for each category, a Puppet DSL _category value expression_. Any non top
scope puppet expression may be used (conditional logic, loops, function calls, etc.) and should result in
a `Literal` value (`String`, `Boolean`, `Number`) or empty string/undef (if the category does not apply).

The categories `node`, `environment` and `common` are always bound with default values. The user may alter the
expression for `node`, but not the expressions for `common` and `environment`. Since everything is in the common
category and it is always has the lowest precedence, it should never be specified by the user.

Rules:
* If user specifies the category `common`, an error is raised
* If user specifies the category `environment` (for the purpose of defining precedence), it should always be
  set to `true`. If set to something else, an error is raised. (Internally it is always set to "$::environment", but
  this is a nuisance to have to repeat).
* If `node` is given lower precedence than `environment` an error is raised
* If `environment` is missing it is automatically added with the precedence just above `common`.
* The `environment` category always evaluates to `$environment` (this can not be changed).
* If `node` is missing it is added with the highest precedence and its value is set to `$::fqdn`.
* It is not allowed to perform injection since the injector has not yet been set up (this rule may be
  modified to allow injection of system level services that are setup by the puppet runtime - to be designed).
* It is illegal to evaluate more than one `site { categories {} }` expression and an error is raised if this
  is attempted.

Thus, `node`, `environment` and `common` are always guaranteed to be present, in that precedence order, and
always with sane defaults.

#### Example: Adding a Virtual Category Below Environment

    site {
      categories {
        node        => "$::fqdn",
        environment => true,
        virtual     => "$::is_virtual"
      }
    }

> ##### Conditional categories, or not?
> Discuss: We have the option to allow categories to be specified in conditional logic. This may be of value
> if the user wants to share the same `site.pp` across environments, and only need a small change to the
> setup.  
> Pro: Flexible for users  
> Con: Potentially more complex to read/understand, and makes static analysis harder, also adds a point of
> indirection that may lead users to choose the path of least resistance and thus end up with bad design.

### Summary of Categorization

The above design should be powerful enough to deal with:
* An existing ENC that determines the environment
* Blocking unsafe `$environment` from facts
* Allowing user logic to determine what `$environment` should be set to since the logic can consider:
  * An `$environment` set by an ENC
  * An environment passed as secure data via mechanism defined in a future ARM
  * Logic involving other facts/fqdn/certname, using general puppet logic such as case/if/pattern matching that results
    in the `$environment` to use.
* Provides a sane consistent set of categories and precedence semantics that can be used across modules and
  organizations (`node`, `environment` and `common`).
* Provides categorization that allows general bindings as well as specific bindings using categories suitable for
  user organization's business needs.

In essence, the "power of an ENC" moves into the Puppet language and is given a richer set of data as the basis for the
decision than what is currently available in the ENC API.

As you will see in [Bindings](#bindings), and [Composition of Bindings](#composition-of-bindings), the categories are
used when expressing bindings, and the composition section defines how overrides and shadowing of bindings can be made.

Bindings
--------

A binding associates a _Type/Name_ key with a producer of an object (i.e. something that produces
a value, or a provider of behavior).

This section defines how bindings are made using concrete Puppet language syntax.
The Puppet syntax hides almost all of the internal details; in most cases the actual internally used [Type](#type-system)
and Name are not visible as bindings are expressed in terms of Puppet syntax elements where type and name are inferred.

The internal representation in general is really only a concern when adding extensions, but advanced users need to be
aware of the [Naming Scheme](#naming-scheme), as certain Type/name keys have special meaning.

### How and Where are Bindings Defined

Bindings are either defined in `.pp` files or provided by a [Bindings Provider](#bindings-provider) that consults some
external source.

A (set of) bindings defined in a `.pp` file must have a filename that reflects the symbolic name of these bindings.
The keyword `bindings` is added to the language, and it denotes a first order named object (similar to class and defined
resource type) i.e. something that is symbolically referenced via its name (rather that its location; which may be
somewhere other than in the file system). Note that this is however not a named scope like that created for a class.

Here are some examples:

    # $confdir/bindings/default.pp
    bindings default {
      # bindings in categories
    }

    # $confdir/bindings/mybindings.pp
    bindings mybindings { 
      # bindings in categories
    }


The name should be the fully qualified name, a bindings named `default` in module `mymodule` is
named `mymodule::default` and must reside in a file called `default.pp` under the module's directory
called `bindings` or it will not be found by the default loader.

    # <modulepath>/modules/mymodule/bindings/default.pp
    bindings mymodule::default {
      # bindings in categories
    }

As you will see later in [Composition of Bindings](#composition-of-bindings), there are semantics
associated with the name `default`.

The rationale for using symbolic names is to separate the concerns of identity, representation/encoding/format, and storage.

### Category Specific Bindings

Bindings are category specific. If not nested inside an explicit category the bindings are in the common category.

The keyword `when` states the categorization and the category the _[current state](#term-current-state)_ should
be in for the bindings to be in effect.

    bindings default {
      bind ...   # a common binding
      when node 'kermit.example.com' {
        bind ... # a binding in the node category 'kermit.example.com'
      }
    }

The same categorization can be used multiple times, i.e. `when node 'kermit'` can occur multiple times to allow logical
grouping of bindings related to some common concern.

> ##### Explicit when common{}, or not?
> Discuss: We can allow `when common {}` to be used as well to bind in the common category (or require that
> global bindings are always done this way and that bindings are not allowed outside of a when declaration) - in both
> cases the internal representation would be one and the same).

#### Nesting of Categorized Bindings

It is allowed to nest `when` clauses. Although categorizations are placed in a precedence order, some
categorizations will describe disjunct sets (_virtual_ / _non virtual_, _in data center x or y_, etc.) rather
than some containment hierarchy (e.g. _rack_ / _data-center_).  Some concepts may have a natural association
with "containment", but this is not required. It does not matter if a category of higher precedence is nested
in one that is lower or vice versa since the state has to be in all of them (i.e. an _and_ operation) to have effect.
This makes it very flexible to chose what is done in "general" and what is "special".

    bindings x {
      when virtual true {
        when node 'kermit.example.com' {
          # when kermit is virtual bind something special than for all other virtual
        }
       # for anything else that is virtual
    }

Nesting is an _and_ operation. Internally, this results in one binding with the precedence of the highest
precedented category in the nesting - see discussion below as this simple rule may break 'the law of least surprise'.

Since (as you will see later) it is possible to use an `or` operator it is symmetrical to also allow `and` to have
the same effect as a simple nesting - using the same example as above:

    bindings x {
      when virtual true and node 'kermit.example.com' {
          # when kermit is virtual bind something special than for all other virtual
      }
    }

> ##### Precedence score - how?
> Discuss: One could argue that a nested binding should have higher precedence than a non-nested since
> it is "more specific" (compare CSS), but this leads to the need to also use tricks like CSS "important" to
> raise the precedence. Since we are not dealing with extremely general constructs (like a "letter in a document")
> the simplified rule will probably pose no practical problems and should be easier to understand, reason-about, and
> debug. OTOH, users have to be aware that bindings for 'kermit' and 'virtual kermit' must not be in conflict which seems
> quit unnatural.
>
> Alternative: A simple CSS mechanism where precedence is a score (sum of precedences) 
> has the odd effect that say 'environment' + 'virtual' gets a higher precedence than 'node'. The scoring
> must instead add a fraction to the highest precedence so that the score does not pass the ceiling. Thus combinations
> increase the precedence like numbered paragraphs: 'environment' (2) and 'virtual' (3) is given the score '3.2' which is
> lower than 'node' (4). The segments are ordered in descending precedence order.

It is also allowed to declare multiple categories using `or`:

    when virtual true or node 'kermit.example.com' {
      # bindings when virtual or when node is kermit
    }

Internally, this results in two bindings with different precedence. This is of value
to avoid repetition. (If overused there is probably something wrong with how the user defined the categorizations).

This may lead to hard to understand error messages as one of the precedented bindings may cause
a conflict - it should however be possible to explain the issue well enough to the user to make it understandable
- e.g. "The binding of x (on line n, file f) in category 'virtual true' is in conflict with the same binding in..."

Bind Operations
---------------

Bind operations are used to define the individual bindings (binding one or several Type/Name keys to a corresponding
producer of an object). The expressive power of bindings is achieved with a relatively small syntax that offers support for:

* **abstract bindings** -- that must be overridden with a concrete binding
* **override bindings** -- bindings that must override an existing (abstract or concrete) binding
* **multi bindings** -- binding multiple producers to a common multi-valued key
* **parameter bindings** -- binding one or several parameters to the various parameterized puppet types
* **query bindings** -- binding the result of a query
* **named bindings** -- binding type/name combination
* **anonymous binding** -- binding using only type
* **rebinding** -- re-binding an already bound key (alias or rename)
* **binding of parameter mapping** -- allows mappings of parameters between types
* **binding of variables** -- allows binding of top scope variables

You probably wonder what all of these mean, and if a user really need to learn all these different terms. But as
you will see, the concrete syntax makes specification of all these kinds of bindings quite natural as you only have
to consider; _Where do I want an injection to take place, and what should be injected_, and in some cases _how should it
be injected_.

### Bind Operations Syntax

To explain the expressive power of binding operations it is easiest to first show a pseudo grammar (i.e.
the real grammar will be more complex).

(You can safely skip to the detailed explanations of each concept and examples and return to this grammar
if you want to look at details).

    # Syntax
    
    bind : binding_kind options
    
    binding_kind
      : BIND      bindargs bind_key to_producer
      | BIND      bindargs bind_key to_producer IN STRING
      | BIND MAP  bindargs type TO type options
      | MULTIBIND bindargs bind_key AS STRING to_producer
      | BIND ALIAS bind_key AS bindargs bind_key
      | BIND RENAMED bind_key AS bindargs bind_key
      | REBIND    bind_key
      | INCLUDE   name_list
      | EXCLUDE   name_list
    
    bindargs : ABSTRACT | OVERRIDE | ABSTRACT OVERRIDE
    
    name_list: NAME | '[' names ']'
    names: NAME | names ',' NAME
    
    to_producer
       : # empty if inferrable or not applicable
       | TO producer
       | TO OPTIONAL producer
    
    bind_key
      : # empty if inferrable
      | PARAMETERS type
      | VAR
      | VAR type
      | type
      | type ',' STRING
      | STRING
    
    producer
      : literal
      | PRODUCER type
      | PRODUCER STRING
      | type
    
    options      : nil | '{' option_list '}'
    option_list  : NAME '=>' expression | option_list ',' option_list
    
    type
      : CLASSREF
      | CLASSREF '[' expression ']'
      | CLASS NAME

TODO: Query is missing

> Note: All capital names are keyword tokens except NAME and CLASSREF which corresponds to the Puppet language
> tokens (lower case, initial letter names, and capitlized '::' separated class references),
> lower case names are rules, quoted characters are punctuation tokens. The rule `expression` is the general
> expression rule in the egrammar, and `literal` refers to literal strings, numbers, booleans, arrays and hashes.

> ##### Use of term Literal?
> Discuss: The term 'literal' is a misnomer, but that is what it is currently called in the grammar; interpolated strings
> are not really literals as they require interpolation - it also clashes with the defined `Literal` defined
> in the [Type System](#type-system) where `Data` means a structure, and `Literal` is a leaf.
> The terms used for these should be unified to avoid confusion (even if some are invisible to users they make it
> difficult to talk about them among the developers).

> Also Note: The simplified grammar is only for illustration - it is not a correct grammar as it has many
> illegal invariants, and requirements that elements are required when they are not. Example; a multibinding
> requires the id option to be set. Most abstract bindings can not have a producer (nor options). This is explained
> in detail in the following sections.

### Binding and Multibinding

A binding is one of:

* `bind` -- (binds one or multiple keys)
* `bind parameters` -- binds one or multiple parameters
* `bind variables` -- binds one or multiple top scope variables
* `multibind` -- binds one key to which other (_fragment_) bindings contribute
* `bind map` -- binds a parameter map from one type to another
* `bind alias` -- creates a copy binding under a new name
* `bind renamed` -- create a copy binding under a new name and removes the original
* `rebind` -- bind existing binding with higher precedence
* `include` -- binds inclusion of classes
* `exclude` -- binds exclusion of classes

Each form of binding is explained separately in the following sections. The explanations of the general
concepts are illustrated with _simple named bindings_; binding a value to a name. These are not the most powerful and useful
kinds of bindings, but they are simple in form which makes it easier to explain the concepts. Also
when [Named Bindings](#named-bindings) are discussed in more detail, you may find that they are indeed quite useful.

### Abstract Bindings

A binding may optionally be declared to be `abstract`. (It may be [combined](#combining-override-and-abstract) with
the keyword `override`). A binding that is not _abstract_ is said to be _concrete_ (but there is no keyword for that).

Modules can not always provide default values for bindings that will make them work and it is required by a user to
configure a concrete binding. In order to do this, a binding needs to be able to express that it is abstract in such a
fashion that it is an error if an abstract binding is not overridden by a concrete binding.

    bind abstract "the meaning of life"

As you can see, an abstract binding does not bind any value (or producer to be more precise), it is simply a
declaration that a binding must exist.

In the example above, if a concrete binding is not provided that overrides this binding, an error is raised (when
the injector is created) telling the user that a binding must be provided
for the key `Data, "the meaning of life"`. The user may do so like this:

    bind "the meaning of life" to 42

An explicit injection calling `inject("the meaning of life")` will then produce the value `42`.

See [Named Binding](#named-binding) and [Type System](#type-system) for details about
type (in case you wonder why the error message contains the text `Data` and what it means).

#### Invariants of Abstract Bindings

Some combination are not meaninful, specifically abstract bindings of parameters - they are already declared to exist - it would not
be meaninful to also specify that a parameter must be bound. (The real grammar/validation rules will enforce this).

Abstract binding of variables is however meaningful, it declares the requirement that a top scope variable must be set.
See [Binding Variables](#binding-variables) for an example.

> ##### Is abstract too abstract?
> Discuss: If the Computer Science term 'abstract' is to abstract [sic] a more populistic name like `placeholder`
> could be used, but this would probably alienate developers.

### Override Bindings

A binding may optionally be declared with `override` which indicates that the given key must be an override of
an existing key with lower precedence, or an error is raised. (What happens when two bindings have the same precedence
is explained in the section [Composition of Bindings](#composition-of-bindings)).

When composing a system out of pieces, where the pieces are glued together via names it is easy to end up with bindings
to names that have no meaning -- e.g. binding "bananas" to 42 is probably meaningless.
In order to handle this, it is of value to declare that something must be an override of something that is declared/exists.
This helps catch spelling errors as well as errors introduced by API changes.

Using the same example as before, our override of the abstract meaning of life can thus be written:

    bind override "the meaning of life" to 42

Which protects us if the original abstract binding is changed to say "the meaning of love", which would make a
binding to "the meaning of life" as meaningless as injecting "my hovercraft is full of eels". In the abstract case, we
would naturally get an error that "the meaning of love" is abstract, but we are also told that the "the meaning of life"
does not override something that exists. The value is even greater when overriding something that is not abstract
as that would just be silently ignored.

> Note: This is similar to how the Java @Override annotation works.

> ##### Is override too overloaded?
> Discuss: If deemed that the term 'override' is too overloaded (one binding overrides another because of
> categorization and layering) another term that suggests 'in error if not bound elsewhere' should be invented.
> This type of protection may seem too sophisticated at first, but is really a great help in maintaining sanity in an
> evolving system.

### Combining Override and Abstract

It is possible to combine `override` and `abstract` which has the effect that an existing binding (typically concrete or
this type of binding would be pointless) must instead be supplied by someone else. This is useful when there are errors
or conflicts among used modules and the user wants to author a set of bindings that can be reused to solve these problems.
Naturally, the author can not know the concrete value to bind but can help downstream usage as they are helped by
an "abstract missing" error instead of a faulty binding that needs resolution.

Assume that in a composition of bindings, this binding would become effective if nothing else was stated:

    bind "the meaning of life" to 43

This is then easily modified to be abstract and passed on:

    bind abstract override "the meaning of life"

You can think of this as a statement of _"You are all wrong, someone needs to set the record straight"_.
Also see [Rebinding](#rebinding) which solves a similar problem.

### Named Binding

It is possible to bind to almost any name except that names may not conflict with the internal naming rules
as expressed in [Naming Scheme](#naming-scheme). Naturally, there
are no [implicit injection points](#term-implicit-injection-point) for such bindings, they
are always injected using [explicit injection](#term-explicit-injection). By convention, these names should
be based on the fully qualified name of the module/place they are defined to avoid name-space conflicts.

Since a human readable string can be used, named bindings can be self descriptive, as in this example:

    # Bind literal value to a string
    bind "main site URL for blogs" to 'http://blogs.example.org'

With this binding in effect, a call to `inject("main site URL for blogs")` results in the value `'http://blogs.example.org'`.

Named bindings are very useful for constants as they, in contrast to variables, can be composed/overridden.

Binding to a name without specifying the type, is the same as binding `Data/name` where `Data` may be a
subtype of `Data` when the type can be inferred from the bound value. In the example above, the binding would
be `String/main site URL for blogs`.

In general the names are unique per type, but special rules apply to all subtypes of `Data` where the name
must be unique across all bindings to any such subtype. This means it is not possible to bind say an `Integer` and a `String`
to the same name.

An injection using `inject('main site URL for blogs')` means lookup any `Data` compatible type with the given name.
A user concerned with type safety would instead call `inject(String, 'main site URL for blogs')`.
For more details about types see [Type System](#type-system).

### Typed, Named Binding

A typed, named binding is primarily used to ensure type safety. Rather than just binding to a name, the produced type is also
declared.

> Type binding is a bit of a misnomer since all bindings are typed, a better term would perhaps be
> explicitly typed binding.

Examples using explicit type:

    bind Integer, "mymodule::size" to 42
    bind Hash, "mymodule::stuff" to {'a' => 10, 'b' => [1,2,3]}

Explicit type is typically not needed for data as the type can be inferred from the bound object.

Typed bindings become more important when doing more advanced bindings. As an example, here is an abstract binding where
a module wants to be able to inject an array of class ntp instances.

    bind abstract Array[class ntp], "ntp resources"

At some point, someone adds a binding that results in an array of such resources; as a static list, as a query or
a function, a Ruby extension, etc.

And here is an example binding a syntax checker to a puppet extension point:

    bind Type[SyntaxChecker], "xml" to Type[MyXmlSyntaxChecker]

See [Type System](#type-system) for more information about how to write references to type.

### Typed, Anonymous Binding

A typed binding that is not named is said to be anonymous. The idea is that whenever an instance of the given
type is requested, the caller receives the object from the bound producer.

This is typically used to bind a concrete implementation of an interface, or to replace a null/simple implementation
with a more specialized. This type of binding is of value when type alone provides a unique key.

    bind Type[Logger] to Type[SpecialLogger]

This can also apply to Puppet types. This would enable instantiation of a specialized class instead of the
original. This can be useful where there is a desire to not modify the source.

    bind class ntp to class ntp2
    bind MyResourceType to MyBetterResourceType

The bound type must be compatible (i.e. be a subtype). This type of binding can not be applied to bindings that
address an instance (e.g.. it is impossible to change the type of an already existing resource; File['foo'] can
not be changed to BetterFile['foo']).

> Note: In case you wonder, if `inject(SomeType)` is called and there is no binding for `SomeType` the injector
> will then create an instance of the given type.

### Rebinding

Rebinding comes in three forms (examplified):

    rebind "the meaning of life"
    bind alias "the meaning of life" AS "the meaning of love
    bind renamed "the meaning of life" AS "the meaning of love"

Rebinding, as you may guess re-binds an already bound key's producer in a higher precedence (If it is not higher,
the operation is meaningless).

A `bind alias` creates a new binding with the name given after the `as` bound to the original producer. The original
binding remains in effect (if it surfaces).

A `bind renamed` is the same as `bind alias`, but the original key is erased.

The rebound key must naturally exist or an error is raised. This means that an `override` has no effect. 
An `abstract` makes the re-bound key abstract. None of the three rebind operations accepts a producer.

Rebinding is valuable when adapting old bindings to new and there is a desire to continue using what was originally bound
but under a new key. This also makes it possible to rebind in a category of higher precedence.

    # create an alias
    bind alias "the meaning of life" as "the meaning of love"
    
    # rename
    bind renamed "the meaning of life" as "the meaning of love"

### Multi-binding

A multi binding is a specialized binding that collects [binding fragments](#term-binding-fragment) into a structure.
The bound type must be a `Hash` or `Array` with a type argument that specifies the data type of values in the
structure. A multi-binding may also specify the identity of the multibind itself using an `as`-clause. If no `as` clause
is used, the binding id is the same as the name in the multibind's key.

Here are two examples showing the essential parts:

    multibind Array[String], "names" AS "mymodule::list-of-names"
    bind to 'Mary' in 'mymodule::list-of-names'
    bind to 'John' in 'mymodule::list-of-names'

This example collects fragment bindings that contribute to the multibinding id 'mymodule::list-of-names'. The
result is an `Array` of `Strings`. The fragments do not need to declare the type, and since the collected result
is an `Array` it is not meaningful to name these bindings (they may be, but the names are not used). Since type is inferred,
and name is not required it is possible to simply state `bind to 'Mary'` instead of `bind String to 'Mary'`.

    multibind Hash[Data], 'names-with-data' AS 'mymodule::names-with-data'
    bind 'Mary' to 'engineering' in 'mymoduyle::list-of-names
    bind 'John' to 'support' in 'mymodule::list-of-names'
    bind 'Fred' to ['support', 'sales'] in 'mymodule::list-of-names'

The second example collects fragment bindings that contribute to the multibinding id 'mymodule::names-with-data'.
The result is a `Hash` of `Data`. In this case the contributions must be named, but type can be omitted since it
can be inferred. As you can see, since the Hash is allowed to contain any `Data`, it is possible to bind things
like an `Array` to the key `'Fred'`.

#### Overriding Multibindings

The reason for the dual keys (the multibind's key, and its binding identity given after the `as`) is that it would
otherwise not be possible to differentiate between fragment associations to one specific multibinding
and an overriding multibinding that does not want to accept a completely different set of multibindings.

If a new multibinding is made with the exact same key and identity it would still be getting the same fragments.

The binding id must be unique across all effective bindings and by convention these should include the module
name as a suffix.

As an example, given these bindings:

    multibind Hash[TypeValidatorService]], 'syntax' AS 'syntax-validators'
    bind "xml" to Type[MyXmlValidator] IN 'syntax-validators'

And there is a desire to override the multibind with an empty Hash, this is stated as:

    multibind override Hash[TypeValidatorService]], 'syntax'

Since no identity is given, it is impossible for fragments to attach (the id is unknown). Had the same
identity (i.e. 'syntax-validators') been used the result would have been the same as in the original (i.e. the
"xml" binding will be contributed to it).

#### Array Multibind Example

We want to collect strings in an array where each string is a user name.

In this example, a multibind is created. It is declared that each contribution should be of type
`String`. Contributions should be made using the key 'included_users', and the result of
the collection is of `Array` type. Thus a call to `inject('users')` results in
an `Array` with the content ['anna', 'akuna', 'ries'].

    multibind Array[String], 'users' AS 'included_users' 
    bind to 'anna' in 'included_users'
    bind to ['akuna', 'ries'] in 'included_users'

A multibind is used (in addition to the obvious reason to enable collection) to allow the entire result
to be overridden with a binding of 'users' if so desired.

#### Multibind -- to Merge or not to Merge

There is an obvious issue when hashes are collected/combined. The order of fragment collection is undefined and thus
a regular merge will arbitrarily overwrite keys in the resulting hash if keys are not unique. The same happens
with deeper nested values if hashes indeed where to be merged.

A similar problem occurs for arrays. The default rule appends any non array value and concatenates array. But
what if the desired result is instead to always append, or flatten and append, or convert arrays into hashes
that are appended?

Instead of solving all such problems and turning Hiera2 into also being a Swiss Army Knife of data transformations
a general `combinator` option allows a lambda to be given that performs any non standard data transformation. This
lambda can then use other data transformation functions to produce the wanted result.

(It would be very good if the standard lib functions where cleaned up and improved in this area. There is no deep
merge function and available functions have varying power and flexibility). This should not be used as an excuse
to instead build such functionality into Hiera2.

> ##### Lambda or Type reference (to get an instance), or support both?
> Discuss: A lambda is convenient, it could also be allowed to use a Type reference to a subclass of Combinator
> to allow a runtime class to be plugged in.

#### Multibind Array Combinator Option

When performing a multibind of `Array` type, all contributions are by default concatenated. A non array
contribution is converted to array and then concatenated. If something else
is wanted, a lambda can be assigned to the binding option `combinator`. The lambda is passed the already combined
result and the new value to concat/add/combine.

Here is an example where the combinator concatenates flattened arrays (using the standard lib function
flatten). The example produces [1,2,3,4,5,6]:

    multibind Array[Data], 'flattened-data' AS 'example' {
      combinator => |$memo, $x| { $memo + flatten($x) }
    }
    bind to [1,2,[3]] in 'example'
    bind to [4,[[5,6]]] in 'example'

#### Multibind Hash Combinator Option

When performing a multibind of `Hash` type, all contributions must have unique keys or an error is raised.
If something else is wanted, a lambda can be assigned to the binding option `combinator`. The lambda is
passed the key, and the existing value for the key (or undef if no value) and the new value. The lambda should
produce the new value for the key.

Here is an example that concatenates arrays for the same key:

    multibind Hash[Data], 'merged-hash' AS 'example' {
      combinator => | $key, $current, $value| {
        unless !$current
          { $value }
        else
          { $current + $value }
      }
    bind 'fruits' to ['apple', 'orange'] in 'example'
    bind 'fruits' to ['pear', 'mango'] in 'example'
    bind 'berries' to ['strawberry', 'blueberry'] in 'example'

The result when calling `inject('merged-hash')` is:

    { fruits => ['apple', 'orange', 'pear', 'mango'],
      berries => ['strawberry', 'blueberry']
    }

(Unfortunately there is no suitable function in puppet standardlib that does this). A deep merge function would be
a welcome addition to the standard lib, and that is more flexible and handling an undef start value.

#### Multibinding Rules

* A `Hash` multibinding requires fragments to have a key with a name or an error is raised.
* All `Array` multibinds concatenates contributions by default (non array values are appended), if something
  else is wanted a _combinator_ should be defined.
* `Array` multibinds ignore the names of contributed fragments
* All `Hash` multibinds require that fragment names are unique or an error is raised. If something else is wanted a
  _combinator_ should be defined that either overrides or merges the result.
* Fragment collection order is undefined (if the order matters, the result should be sorted by the user)

### Binding Classes

Classes (or rather, class inclusion) is performed with (examplified):

    include name
    include [name1, name2, ...]

It is also possible to bind exclusion, and it is otherwise difficult to erase/replace a faulty binding.

    exclude name
    exclude [name1, name2, ...]

The list of classes is injected with `inject('/classes')`. Also see [Naming Scheme](#naming-scheme).
Internally, this is handled as multibindings to 'included_classes' and 'excluded_classes'.

An include wins over an exclude if the exclude has lower precedence. Eclude of a non-existing included class
is a no-op.

> ##### Use keywords include (and exclude) for class inclusion (and erasure)?
> Discuss: Class inclusion (and exclusion) was given concrete syntax rather than being multibind contributions.
> Both forms will naturally work, but the multibind contribution style is primarly for external services that
> act as a bindings provider. The rationale for the concrete syntax is that inclusion is one of the first things
> someone wants to do, and having to learn about multibindings upfront it not nice. Also, instead of using something
> like `bind classes`, it felt more natural to use the keyword `include` used in regular logic.
> As a consequence this makes it more difficult to erase/replace and the `exclude` was also needed.

### Binding Parameters

Binding parameters is done in this form:

    bind parameters <instantiateable-type> to <parameter-producer>

Where _&lt;instantiateable-type&gt;_ is a reference to:
* a classname (starts with a lower case character)
* a type
* a resource instance

and _&lt;parameter-producer&gt;_ is one of:
* a literal hash
* a type
* resource instance
* a producer

See [Type System](#type-system) for more detail.

It is possible to bind any object that supports keyed access to values.
If the bound object has incompatible parameter names it is possible to bind a mapping from the
expected name to the name in the bound object. The following sections explains this in more detail.

#### Binding Parameters using a Literal Hash

When binding parameters, it is possible to bind to a Hash where the hash contains parameter name
to value associations.

    bind parameters ntp to {
          autoupdate  => false,
          enable      => true,
          servers     => [
            "0.us.pool.ntp.org iburst",
            "1.us.pool.ntp.org iburst",
            "2.us.pool.ntp.org iburst",
            "3.us.pool.ntp.org iburst"
            ]
        }
    }


Internally, each parameter is bound separately and can thus be individiaully overridden (or overridden by a
binding of higher precedence hash which contains the same keys).

    bind parameters ntp to { autoupdate => false }
    bind parameters ntp to { enable => true }

The last two examples bind a single parameter respectively.

#### Binding Parameters using Instance Reference

It is possible to bind to any instance of any type (provided that a) the type exists, or there is a binding to
a producer b) it is possible to refer to an instance of this type using a name.

The injector will search for a parameter binding, and if it finds a binding to an instance of some type, it will
try to find this instance (or if there is a producer, create it). If the instance is not of the same type as the
LHS side a search is made of a `Hash[String, String]/Type[ParamMap[LHSType, RHSType]]` binding. If no such binding
is found the operation fails unless parameters happen to have equal names in the two types. If a mapping is found, it
is used to translate an LHS parameter to an RHS parameter and the mapped parameter is then used to lookup a value
in the object bound as producer of the parameter value.

    # bind the parameters of Web[one] resource to the resource Sql[one]
    # and map the parameters from Sql[one].
    #
    bind parameters Web[one] to Sql[one] {
      map => {dbuser => user, ... }
    }

#### Binding Parameter with Map per Type

It is possible to bind a parameter map for a pair of types by using a type on the RHS and specifying a map as in 
the following example:

    bind map Web to Sql {
      key_in_Web => key_in_Sql
    }

This means _if a parameter binding of type Sql is made to a Web, then map this way_.

With the binding above in effect, the mapping can now be skipped when the instance bind takes place:

    bind parameters Web[one] to Sql[one]

In this example, the map bound for the combination Web/Sql will be in effect, and parameters will be mapped without
having to be specified where such a binding is made.

#### Advanced Parameter Bindings

There are many cases where parameters for a given type should come from multiple source (e.g. some given in
user logic, other from a bound type, and lastly from bound parameters (a hash with name value associations).

In order for this to work, the bound producers can not produce overlapping parameters (or the result is
undefined as the lookup order is undefined).

Thus, the bound producers of parameter values must either produce values for disjunct parameters, or there must
be bound parameter maps that contain mappings for the parameters to make them disjunct.

As an example, for a bound hash it is easy to break it into individual parameter bindings
based on the given parameter names in the hash, but for a binding of an arbitrary object this becomes more complex.
Should all of its parameters be used? What about parameters that do not exists in the target, should they be skipped or
mapped, is it an error if there is no map entry, or does this indicate that it should be skipped - etc.

It is not allowed to bind a type more than once as a parameters producer; i.e. this is illegal:

    bind parameters Web[one] to Sql[one]
    bind parameters Web[one] to Sql[two] # error, an Sql is already bound.

Different types may however be bound:

    bind parameters Web[one] to Sql[one]
    bind parameters Web[one] to DbUserInfo[general]

The proposed solution is to use the following rules:

* When constructiong the effective bindings, maps are processed first
* For each effective binding of non hash based parameters, obtain the map, and bind the mapped parameters
* If no map is found; compute the join of the two types' parameter name sets - bind these names as mapped
* If the join of parameter names is empty an error is raised ("map required")

Also note that:
* A map option in a parameter binding overrides a default binding (at type level) - i.e. it acts as if it
  was a separate `bind map` with the same precedence as the parameter binding.

Here is an example showing how to ensure non overlapping parameter mappings when binding parameters from
multiple objects:

    bind map X to Y {
      a => a
    }
    bind map X to Z {
      b => b,
      c => c
    }
    bind parameters X to Y[some_y]
    bind parameters X to Z[some_z]

Here we resolve the issue that both Y and Z have parameters a, b, and c. By defining the maps it is clear what to
pick from the respective type.


### Binding Variables

The binding of top scope variables is similar to binding of parameters, but here no mapping is offered. Each
variable is internally bound separatly (and may thus, just like parameters be individually shadowed by a binding
with higher precedence).

Here is an example:

    bind variables to {
      myvar => 42,
      my_other_var => 44
    }

> Rationale: We could simply handle these bindings as variable assignments among the
> other binding operations - i.e. `$a = 10` is the equivalence of `bind variables to { a => 10 }`. Although straight forward
> it will probably cause confusion over parse/evaluation order, and would not be symmetrical with other types of bindings (How is
> the assignment then made abstract?
>

Variable binding keys must be unique across all types (i.e. variable names are unique).

#### Typing Variables

The simplest form of binding variables binds variables (as shown in the example in [Binding Variables](#binding-variables),
with type determined by inference. If a type is specified and a literal producer is used, the bound literals must comply with
the declared type.

    bind variables Integer to { 
      a => 10, # ok
      b => 'bananas' # error
    }

Although perhaps not valuable on its own it is of value when declaring abstract variable bindings.

#### Abstract Binding of Variables

It is worth noting that it is possible to specify what a module expects in terms of defined top scope variables
by declaring them as abstract. This makes it possible to give errors upfront (and tools can perform this analysiz statically)
without having to wait until the moment in logic where such a variable may be referenced.

Arguably, depending on the precence of top scope variables is not the recommended approach given that parameters can be
specified in more precise ways. It is however of value when dealing with older code / existing modules where this style is
used. An external source can now declare variables to be abstract in a helper module (say a helper module that adapts an
older module for use in a more modern puppet).

Examples
    # A module wants to ensure that the variable `$::mainurl` is set
    bind variables abstract to 'mainurl'

    # Declaring multiple variables at once as abstract
    bind abstract varibles to ['must_be_set', 'this_too']

This is (probably) the only exception to the rule that an abstract binding does not have a "producer"; in this case a producer
of the names to bind abstractly. It is farfetched that someone would need a construct where the set of abstract
variables is determined by a non literal producer (i.e. ruby logic) - hence that will not be allowed.

Typing a variable in combination with abstract is a valuable way of constraining the variable's type:

    bind abstract variable Array[String] to ['blog_administrators']

Abstract untyped variable bindings are made using the type `Any`.

### Binding Dependencies

Since the bindings system is used to compose the content of a catalog, it is important to also be able to compose
dependencies that define the order.

While it is possible to bind to the meta-parameters `before`, `require`, `notify` and `subscribe` it is possible to
define the order using parameter bindings.

I is naturally also possible to define concrete syntax. (This is not included in the proposed grammar, as the topic needs
to be discussed). Here are some examples what it could look like:

    bind dependency File['/etc/ssh/sshd_config'] ~> Service['sshd']

Syntax should support `->` and `~>` (but not right to left arrows). Syntax should also support an array of references.
The semantics are: if both left and right side are in the catalog, then add the dependency.

>##### How should dependencies/ordering be handled?
> Discuss: Is it enough to only bind order using meta parameters?
> Are the semantics the correct semantics? (i.e. ignored if not both sides are present)

### Summary of Binding Options

The concrete syntax for bind operations allows options to be declared in `{ }` at the end of the clause.
The options are related to different types of binding operations, and they are discussed in detail for the
respective binding type.

Here is asummary of the available options:

| Options            | Type    | Applicable Where | Description                                                           |
| ------------------ | ------- | ---------------- | --------------------------------------------------------------------- |
| `combinator`       | lambda  | multimap         | defines the behavior of combining contributions to a multimap         |
| `producer_args`    | hash    | producer         | a hash of parameter name/value to pass to the producer's constructor. |
| `map`              | hash    | parameters       | a hash with name to name mapping                                      |

> TODO: `select_result` from handling a query (lambda when there is > 1 result) (note optional defines what happens
> if there is no result from a query.

### Producer

A `Producer` is responsible for returning the resulting object when an injection takes place.
There are several producer subtypes in Hiera2, and additional producers can be added via
an extension. The simplest producer is the `ConstantProducer` that produces constant values
of `Data` type (see [Type System](#type-system)) i.e. `String`, `Integer`, `Boolean`, `Array`, `Hash` etc.

In the concrete Puppet syntax the producer is implicit in statements like:

    bind "the meaning of life" to 42

Which internally is the same as if the user had written:

    bind "the meaning of life" to producer Type[ConstantProducer] {
      producer_args => { 'value' => 42 },
      singleton     => true
    }

Other implementations of Producer are similarily used for other types of bindings.

A user can bind specialized producers written in Ruby using a Puppet extension. As an example
a user writes a UUID producer. It would be be bound (a completely typesafe way) with something like:

   bind Type[Producer[String]], "mymodule::uuid" to Type[MyUUIDProducer] in "producers"

With such a producer bound, UUID values can then be bound with early typesafety:

   # typesafe binding (type problem detected when creating injector)
   bind String, "request-id" to producer "mymodule::uuid"

Or without binding a registration, just binding the provider class directly:

   # also typesafe, but type problem detected when producer returns value
   bind String, "request-id" to producer Type[MyUUIDProducer]


The rationale to always use a producer is that there is no question whether a value or something
procedural that produces a value is bound - it is always a `Producer`. (The Binding is thus free from this concern).

#### Optional Producer

Normally it is expected that a producer always produce a value (other than undef/nil). By placing the keyword `optional`
after the `to` this requirement is removed, and a producer that produces nothing is silently ignored.

The use of optional is of value in cases when parameters should be bound from some cross node object, or named
resource in the catalog if this object exists. Placing such a binding at a higher precedence level makes it optionally
override others.

    bind parameters Web[one] to optional LimitedServices[web]

This binds parameters of `Web[one]` to parameters from an (also imaginary) `LimitedServices` called 'web'.
If there is no `LimitedServices[web]` included in the catalog the binding is silently ignored.

#### Query Result Producer

> TODO: This is in an odd place in the ARM; part of this is about a general purpose query mechanism,
> some about binding, and some about a query producer. It should either be a separate section, or 
> the parts should be moved to their respective category (text needs to be revised to ensure readability and
> explain/link future references).

It has been discussed if the existing query (spaceship) operators should be modified to be general purpose
queries, but there are several issues associated with doing this. The operators `<| |>` and `<|| ||>` imply _local
virtual_ and _external exported_ resources, a distinction that is not needed (or even wanted) in a
general purpose query. Also, changing the semntics of spaceship operators would break existing users, and will
cause confusion.

Instead, it is proposed that a new query operation is introduced:

    ??<query expression>

Which may be used like this to query a resource of some type:

    ResourceType[?? <query expression> ]

The result is always an array of resources. The <query expression> works the same way as the query
for the ufo operators, but is extended to the full set of expected operators
`and`, `not`, `or`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `~=` and `~!`.

Handling a richer query in a generic way in the Puppet Language opens up for many possibilities; such as passing
the query to a function (the function receives a parsed query model rather than having to do the parsing itself. Tooling
knows about the syntax and validate etc.

### Binding the Result of a Query

A general purpose query is very useful in Hiera2. Concrete syntax was therefore added that allows a query expression
to be used as an expression that is recognized when performing a bind. This means that a QueryProducer is internally
being used, and it gets the query expression to execute when the producer is asked to produce the result.
(This was favored over explicitly selecting a query producer and giving it the query as an argument).

To bind the result of a query, simply write the query in the bindings to clause.

    bind parameters Web[one] to Sql[?? name == one]

In addition to just binding the result, the use of the bound result is recorded in a way
that makes it possible to see the resulting cross node graph. This has not yet been explored, but should
be doable if the returned result has enough information included that allows recognition that they are
remote and require special treatment.

The query is (naturally) executed by a service that is bound via injection. One such service
queries the Puppet DB.

> Note: The result never includes resources from the catalog under compilation
> because of parse order dependencies.

> ##### Subcatalogs, workflows - implications on Hiera2, if any?
> Discuss: R.I. suggests it would be of great value to be able to break a catalog into "sub catalogs" and compose
> them, when doing so, the order between the parts is known, and it should be allowed to query already processed
> subcatalog. (This is concern for the future and is an implementation detail outside of the scope for Hiera2.

### Handling Multiple Query Results

One issue when binding a query is what to do in case zero or more than one object is returned by the query.
In all cases except the parameter binding case, it is up to the receiver/user to transform the produced array.

The simplest is naturally to let the QueryProducer have a standard behavior where a result of 0 or > 1 raises an error, and
a match of one unfolds the one object from the array and produces this object. This is however not always a good
strategy if a user wants to be able to pick _the best_, or do something more advanced).

In order to solve this issue in an open ended way, it is proposed that a lambda is assigned to the option
`select_result`. The lambda has one parameter and is expected to return an instance of the type at hand.

Composition of Bindings
-----------------------

It is important to remember that Modules can come with default bindings and abstract bindings that need to 
be overridden. The modules can come from a variety of sources and the available name space in which they
can bind things is not divided/reserved. As an example module A binds `color` to `blue` and module B binds it to `red`.
Both bindings are in the common category. In our configuration we want to be able to define that in general
color should be `green` as we want to reserve the higher precedence categories to define other colors in these
divisions. In order to compose these we need a mechanism to place bindings in a higher layer where all its
bindings shadow those in lower levels.

To keep things simple, it is proposed that bindings are arranged into layers where bindings in a
higher layer shadows bindings in a lower layer. (The alternative would be to have a directed graph - but this
finer grained control is probably not needed).

The stratas/layers are specified in the site statement where the layers are listed from highest to lowest
precedence. By default there are two such layers, one for all the modules, and one for the site. Here is
an example of what the default configuration looks like if spelled out:

    bindings => [
       layer { 'site':    include => "confdir:/default" },
       layer { 'modules': include => "module:/*::default"}
      ]

This specifies that there are two layers; 'site' and 'modules' - these names have no semantics, they are
simply used as labels so humans can talk about them, when there are errors, and when explaining which
statements resulted in a particular injected value.

Here is an example where bindings from an ENC is given higher precedence than modules, but lower than what is
in the confdir/default.

    bindings => [
       layer { 'site':    include => "confdir:/default" },
       layer { 'enc':     include => "enc:" },
       layer { 'modules': include => "module:/*::default"}
      ]

The references to named bindings use URIs where the scheme part is mapped to an implementation of
a [Bindings Provider](#bindings-provider) (via injection).

### Selecting Bindings with include and exclude

If we want to select a different set of bindings (e.g. one named load_balanced) for one
module (called MySpecial) instead of its default bindings the config could be:

    bindings => [
       layer { 'site': include => "confdir:/default" },
       layer { 'modules':
         include => [
           "module:/*::default",
           "module:/MySpecial::load_balanced",
         ],
         exclude => [
           "module:/myspecial::default"
         ]
       }
    ]

> ##### Are Include/Exclude as keywords overloaded - use something else?
> Discuss: include and exclude are overloaded; elsewhere 'include' means include class in catalog. Include is
> also a bind operation (to bind class inclusion). Maybe use other terms in layers?

### Bindings Provider

A Bindings Provider is responsible for producing an instance of a [Bindings Model](#bindings-model) - an
[ecore](#term-ecore) model. Three such providers are supplied with Hiera2 corresponding
to the schemes confdir, module and enc. An extension point is also offered for 3d party plugins.

#### Confdir Bindings Provider

The confdir scheme refers to the root of the configuration, the final part of the path is a symbolic
name that should correspond to a file name in the $confdir/bindings directory.

If there is a desire to create a structure below the confidir, say bindings for windows::default, these
should be placed in $confdir/bindings/windows/default.pp and referenced like this:

    confdir:/windows::default

An asterisk wildcard may be used to match a single missing segment, and two asterisk may be
used to indicate recursive expansion.

#### Module Bindings Provider

The module scheme is similar to confdir. It is capable of searching the modules on the modulepath.

#### Enc Bindings Provider

The enc scheme refers to the data produced by the ENC. The implementation creates a binding model
from the classes, parameters and top scope variables set aside at the start of the request processing (as described
in [Request Sequence](#request-sequence)).

If there is the need to refer to only part of what is produced by the ENC (say classes), these are individually
addressed as a path - e.g:

* enc:, enc:/, or enc:/*,  -- everything produced by the ENC
* enc:/classes -- only the classes bound to node categories
* enc:/parameters -- only the parameters bound in node categories
* enc:/variables -- only the variables (TODO: Needs discussion)

The rationale for allowing selection of parts would be to aid in migration (but may be of limited value).

> ##### Is it of value to pick parts from an ENC to Hiera2 transformation?
> Discuss: Are these detailed selection of value? Should it always be just enc: ?

#### Custom Bindings Provider

You have already seen three bindings providers; the confidir-, module-, and enc- schemes. This set of
schemes can be extended by binding additional bindings scheme providers. Examples could be providers that
maps existing hiera yaml/json data, connects to an inventory system, makes use of LDAP, etc.

**Example, adding ldap layer**

In this example, a layer is added where bindings are obtained from an external (custom) service.
We add this layer below the site layer because we want to be able to override the external bindings.

    bindings => [
       layer { 'site':     include => "confdir:/default" },
       layer { 'external': include => "my_ldap:" },
       layer { 'modules':  include => "modules:/*::default" }
    ]

Bindings providers are added like other Puppet extensions; they are written in Ruby, and are bound to
the configuration - as an example, the `my_ldap` could be bound like this:

    bind "my_ldap" to Type[MyLdapProvider] in 'bindings_providers'

The API for a `BindingsProvider` has methods that are called when a bindings model is requested, and these
are given access to the URIs and the include/exclude information given by the user. The provider
receives one call per layer where the scheme is used. (Naturally, the instantiation of such a provider is
via injection to allow it in turn to inject what it needs).

### Resolution of Inconsistencies

If there is an inconsistency between bindings in a strata/layer, it needs to be resolved. We can resolve
by simply stating a binding in a higher strata, but we may want to declare these separately.
There are several options depending on the type of issue, and if it is meaningful to create something
that is reusable.

We could place a resolution in the site layer as a separate inclusion or even in the site::default bindings, but
if we are dealing with an override that we may want to reuse across puppet installations, or that is a fix
when using a particular combination of modules, we may want to keep this fix in a reusable module (perhaps only
containing this fix), or we may want to keep it in our module that makes use of the conflicting
modules (i.e. it is stored close to the "source of the problem").

Lets say we place this as a source file in the module "X" that requires modules that cause
conflicts. We could add it to module X's default, but then we need to place the default
segment in a higher strata (since it would otherwise just add further conflicts), and we don't want
this because if something else is in conflict we want to know about it. Instead we add a segment that we
name "resolutions" (name does not matter). If we have several such resolutions, we can place them in the same
strata (and since they are in the same strata we would be told if we introduced new problems by adding an
inconsistent set of fixes).

It looks like this:

    bindings => [
       { 'site':        include => "confdir:/default" },
       { 'resolutions': include => "module:/X::resolutions",
       { 'modules':     include => "module:/*::default" }
    ]

Type System
-----------

Hiera2 assumes that _ARM-7 Puppet Types_ will be implemented in the future, but does not directly depend on it.
The type system that is proposed for Hiera2 will work with ARM-7, and is of value in general.

The design principle for the type system is to provide _reasonable type safety with minimal explicit type specification_.

Here is a list of references to types and instances using concrete syntax, and their meaning:

| Puppet      | Meaning                                                     |
| ----------- | ----------------------------------------------------------- |
| `Class[x]`  | Reference to the puppet host class 'x'                      |
| `class x`   | Same as `Class[x]`                                          |
| `Array`     | An Array containing `Data`                                  |
| `Hash`      | A Hash with `Literal` key, and `Data` value                 |
| `Boolean`   | A boolean value                                             |
| `Integer`   | An integer number                                           |
| `Float`     | A floating point number                                     |
| `Number`    | Integer of Float                                            |
| `String`    | A string (no interpolation)                                 |
| `Pattern`   | A regular expression pattern                                |
| `Data`      | Array, Hash, Boolean, Integer, Float, or String (recursive) |
| `Literal`   | Boolean, Integer, Float, or String                          |
| `Type[x]`   | Runtime type x (i.e. runtime classname x)                   |
| `Array[x]`  | Array with values of type x                                 |
| `Hash[x]`   | Hash with `Literal` key, and value of type x                |
| _FQN_       | A Fully Qualified Puppet type (see below)                   |

References to Instances:

| Puppet      | Meaning                                                     |
| ----------- | ----------------------------------------------------------- |
| _FQN_`[x]`  | Reference to instance of _FQN_ type with id x (see below)   |
| `Pattern[x]`| A regular expression pattern x (in string form)             |

The type names always start with an uppercase name. The names are all in the same name space, i.e. it is not
possible to define resoure types called `Array`, `Hash`, `Boolean`, `Integer`, `Float`, `Number`, `String`,
`Pattern`, `Data`, `Literal`, or `Type`. The name `Enum` is also reserved (it is defined in ARM-7). The type `Class` is
already reserved for classes in Puppet. (This implies that 3.x feature that the classname 'class' is allowed is revoked).

The _FQN_ (Fully Qualified Name) contains one or several `::` separated name segments where each segment starts
with an upper case letter. Such a name is a reference to a Puppet Type. The abstract type name `Resource` is a
reference to any puppet resource type. In the proposed puppet type system all resource types are directly or indirectly
derived from this abstract type. All other types introduced by the user using the extension proposed by _ARM-7 Puppet Types_
are also referenced this way.

> Note. Currently it is only possible to have resource types that are in the top name space. The rules in Hiera2 are
> flexible to handle name spaced resource types if/when added.

As you may have figured out, the global namespace is split into three; Puppet Host Classes, Puppet Types, and Runtime
type. This is required since the current system allows classes and types to have the same name as well as allowing
classes and/or types to have the same name as that of a runtime (Ruby) class.

The rationale for placing Integer, Float, etc.in the top namespace in the type reference scheme is that it is expected
that these are used more frequently and if they were placed in the Type namespace users would have to use `Type[Integer]`
instead of just `Integer`.

The rationale for the abstract data types; Number, Data, and Literal is that these are the most frequently used
(semi-typed) objects in current use. When used as sane type argument defaults for say Hash, a user that does not
care about explit typing still has reasonable type safety (keys are `Literal` and values are `Data`), this way
they are guaranteed that such objects can be safely represented as JSON while still avoiding to add explicit and detailed
type information.

Hashes always have `Literal` key (i.e. it is not possible to use a hash, array, or some runtime object as
a hash key) in the Puppet Language. Keys of different literal type are different even if their string representation
is the same - i.e. `"1"` and `1` are different keys.

The type argument feature allows a single type arg for Array and Hash. If deeper typed structures are needed they
should be declared as separate types as defined in _ARM-7 Puppet Types_. An example would be if a user wanted
to declare a Hash with values that in turn are Hashes of Integer value type.

Puppet Functions
----------------

The following puppet functions are envisioned:

    # args is type and/or name, and an optional hash of parameters
    # passed to an instantiator (if required by the provider).
    #
    inject(*args)

Injects an instance compatible with given type (or type bound to name if no type is given), and
passes an optional hash to the provider to produce an instance (if required by provider).

Not sure `inject_provider()` is needed in puppet, it returns an object that may be used to obtain an
instance by calling yet another function (get). This can probably be skipped in the puppet language, but
it is useful in Ruby when each "use" requires a new instance (e.g. a new connection, a transaction) etc.

The normal case is that injections should always produce the desired value or raise an exception. It
is however useful to support optional injections, this could be made via a different function, `optionally_inject()` w
ith the same signature as inject, or by passing a lambda to inject() that is invoked with either a value, or
undef/nil if nothing was bound.

Modus Operandi
--------------

(TODO: Simplify and move to introduction)
The model is far from everything that is needed - it describes the possible input to an injection mechanism,
not how it is obtained, or operated on.

### Injector
An Injector is created by giving it a "binding model" (as described above). The injector then constructs
the resulting effective bindings, reporting conflicts and other errors as soon as possible. The injector
creation must be given the data required to compute the current set of scopes (these are Pops::Expressions in
the binding model).

Technically there may exist more than one injector in one Puppet Runtime; it is easy to imagine one that
is applicable to the Puppet runtime itself; binding different kinds of services that cannot vary by environment
and that is puppet's internal wiring; i.e. it deals with separation of completely internal concerns. It may be
giving users too much power if this internal wiring is exposed in a UI. However for services/settings that may be
allowed to vary (the user could control them via settings), these should naturally be mixed into the binding model
available to users. It is also possible to build the final binding model by placing any internal Puppet segments in a
strata of highest precedence.

This also makes it possible to turn settings into injection. Puppet[somekey] is actually the same as requesting
injection by name.

For the sake of simplicity assume that each request triggers the full processing of all required steps to produce
an Injector (there are plenty of opportunities to cache intermediate steps later to speed this up).

A created Injector is bound to an environment.

Naming Scheme
-------------

Internally some bindings results in bindings where the name is based on puppet language elements/types.
As a consequence regular names used by users should not start with a '/'.

### Parameters

A parameter binding uses names that include a reference to the type/instance as well as the name of
the parameter.

    /param/Class[name]/pname
    /param/ResourceType/pname
    /param/ResourceType[instancename]/pname
    /param/Type[type]/pname


### Parameter Maps

Parameter maps are mapped as:

    /map/LHSType/RHSType

As an example, if the binding is a mapping from parameters in a type/instance Sql to a type Web, the resulting
key is:

    /map/Web/Sql

### Instance names

When instance names are used in a `/map` or `/param` named binding the name needs to be escaped
as there is a (small) risk that the name may lead to overlapping keys.
Internally, instance names are URL encoded (wrt. the characters that needs escaping `/`, `[`, ´]´, and `%`) e.g:

    File['foo/bar[x]']

Is encoded as:

    File[foo%2Fbar%5Bx%5D]

Note that this takes place internally, and is only a concern in Ruby extensions.

### Classes

The key `Array[String], '/classes'` is the name of the binding that provides a list of classes. It has a producer that
performs an set operation of `inject('included_classes') - inject('excluded_classes')`

### Variables

Top scope variables are bound using:

    /var/var_name


Ruby API
--------

Testing and Evaluation
======================

This ARM does not have any special testing needs. It consists of rich functionality and naturally requires
a substantial test suite.

Users evaluating the ARM should focus on a) feature completeness -- does it meet expectations/requirements
b) usability -- how easy/difficult it is to achieve the wanted result.

Alternatives and Recommendation
-------------------------------

There are currently many discussion points througout the document with open design issues.

There were never any real alternatives to the main feature; use of injection. This since injection
or injection like behavior is already in Puppet.

Unifying these and at the same time offering new possibilities for future services (like better cross node
dependencies (than the simple but useful query mechanism proposed as a starting point)), and a simpler mechanism
for handling extensions/plugins than having to write them as functions, lead to the conclusion that
a unified and powerful injection framework would suit the needs best.

The recommendation to use the Puppet language to describe data, and bindings, and to use better typing is
a step in the direction of providing users with a higher quality service; reasonable error messages instead of
mysterious or erroneous behavior.

A search for injection frameworks for Ruby did not turn up anything of substance.

Risks and Assumptions
---------------------

No known possible derailing events. At this point, since there is yet no exploratory implementation
some of the proposals may need to be adjusted or new ideas prove to be better. Such discoveries during
an exploratory implementation phase will naturally be reflected in this ARM before its final publication.

Dependencies
------------

This ARM does not directly depend on any other ARM, but will be easier to implement when
ARM-7 Types in Puppet has been implemented, and the evaluator compiler has transitioned to
technology similar to that used in eparser.

Also, this ARM proposed a general purpose Query mechanism that is currently available
in a module by Dalén. That functionality should probably be described in a separate ARM (based on
Daléns work).

This arm suggests improvements to the standard lib; e.g. a deep merge function for hashes. Such
an implementation may be based on one of the deep merge implementations available as a Gem. Hiera2
does not directly depend on such a function being added.

The implementation of this ARM will use RGen.

Impact
------

How will this work impact other parts of the platform, the product,
and the contributors working on them?  Omit any irrelevant items.

- Other Puppet components:
  * standardlib function in the family (is_xxx), can be replaced with a general is_a(type) since types
    can now be used as references e.g. `$x.is_a(String)`
- Compatibility:
  * Removes the possibility to have a class named 'class'
  * Reserves type names
- Security:
  * Should be an improvement over directly accepting yaml data
- Performance/scalability:
  * Expected to be on par with Hiera1 (a lot less file loading that is costly - but this is a speculation).
- User experience:
  * This depends on documentation and good communication on how to use the powerful features of Hiera2 rather than
    showing all the features in an academic way.
  * The hope is that common use cases where Hiera1 is used today should be easier to understand when done with
    Hiera 2. Naturally, since Hiera2 addresses issues that either just does not work in Puppet 3, or has mysterious
    concequences a user may wish long for the perceived "simpler times".
- I18n/L10n:
  * Neutral to improved, since logic moves from JSON where there is little that can be done, whereas if I18n, L10n is
    needed somewhere in the puppet logic this should be easier to handle.
- Accessibility:
  * Neutral
- Portability:
  * The proposal means moving logic/data into the puppet language as this is portable.
  * The proposal is based on modeling which makes it easier to integrate with implementation in other languages
  * The type system is designed to deal with "runtime classes" in a manner that should map to language such as Java
- Packaging/installation:
  * Care has been taken to not introduce functionality that requires additional packages.
- Documentation:
  * Hiera2 requies good documentation - it is a powerful framework and users will need to see examples since the
    basic mechanism are quite abstract. It is a new vocabulary for many.
- Spin-offs/Future work:
  * Opens for retirement of current `node` expression
  * Opens for retirement of resource defaults in puppet logic
  * Makes it possible to migrate certain settings to instead be composable injections
  * Opens for (current) ENC removal

Glossary
========

#### <a id="term-abstract-binding"></a> Abstract Binding
A binding that must be overridden with a concrete binding. Marked with the keyword `abstract`

#### <a id="term-binding-fragment"></a> Binding Fragment
A binding that contributes to a multibinding is said to be a _fragment_.

#### <a id="term-category-value-expression"></a> Category Value Expression
An expression that determines the [current state's](#term-current-state) value in a categorization. If
there exists a categorization called `color`, the category value expression may return strings
such as ´red`, or `blue`.

#### <a id="term-concrete-binding"></a> Concrete Binding
A binding that is not an [abstract binding](#term-abstract-binding).

#### <a id="term-current-state"></a> Current State
The categorized current state of the system - i.e. after a request to do some processing has been categorized.

#### <a id="term-ecore"></a> Ecore
A metamodel industry standard for metamodel and model interchange. Implementations exists for multiple language
such as RGen for Ruby, and EMF for Java. (is also available for C, Lua, and GWT (java to gwt javascript)).

#### <a id="term-explicit-injection></a> Explicit Injection
An injection that takes place in user logic by calling a function (or similar) to perform injection.

#### <a id="term-injector"></a> Injector
The mechanism that is responsible for configuring a set of [effective bindings](#term-effective-bindings) given
a state, categorization rules, and layered bindings, and then servicing request to lookup/produce objects to "inject".

#### <a id="term-implicit-injection></a> Implicit Injection
An injection that takes place on the initiative of the runtime.

#### <a id="term-implicit-injection-point"></a> Implicit Injection Point
A point in the puppet logic where automatic injection takes place; e.g. missing parameters for a parameterized class are
looked up using an internal key of `Data/<unqiue name of parameter>` using a unique parameter name as explained
in [Naming Scheme](#naming-scheme)

#### <a id="term-multibinding"></a> Multibinding
A multibinding binds multiple producers to the same key. The result is a `Hash` or an `Array` depending on the type
of multibinding used.

#### <a id="term-override-binding"></a> Override Binding
A binding that must override an existing [abstract](#term-abstract-binding) or [concrete](#term-concrete-binding) binding.
Used to provide name safety when overriding bindings of lower precedence.

#### <a id="term-parameter-binding"></a> Parameter Binding
A binding of a parameter of a parameterized class/resource/type.

#### <a id="term-query-binding"></a> Query Binding
A binding of a result that is obtain via querying.

TODO
====

There should be documentation in the model elements.

model, needs to group multiple parameter bindings using the same mapping

Write example where X has parameters a, b, c, these should come from two different injected objects.

TODO: To bind custom ruby code there is probably also the need to specify what to require in order to load
the code (or maybe this is part of some underlying registration of "plugins").

#### One to One Map

Since a map both defines mapping of names as well as which names to map it may be of value to allow binding an array
of names as a 1:1 map.

One could imagine the syntax:

    bind map T1 to T2 [a, b, c]

But this is not possible using the proposed grammar as an options hash is expected after T2, and this hash doubles as
the mapping hash in a bind map expression. This could be solved by making the options map requiring a map=> key also for
bind map and accept an array as a "unit map". 

Uncertain if it is of great value to make bind map worse to gain the unit ability. It can be done with an explicit unit
map { a => a, b => b }. Since the use case is both advanced and esotheric it seems not worth the cost as it is possible
to achieve the result with a bit more typing.


