ARM-8 Hiera 2
=============

Summary
-------

REQUIRED -- Provide a one-paragraph summary of the proposal, no more
than a few sentences.  This summary will be rolled up into feature
lists and other documents, so please take the time to make it short
and sweet.

Goals
-----

What are the goals of this proposal?  Omit this section if you have
nothing to say beyond what's already in the summary.

Non-Goals
---------

Describe any goals you wish to identify specifically as being out of
scope for this proposal.

Success Metrics
---------------

If the success of this work can be gauged by specific numerical
metrics and associated goals then describe them here.

Motivation
----------

Why should this work be done?  What are its benefits?  Who's asking
for it?  How does it compare to the competition, if any?

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
and much more. To carry on the example above, the current injection system uses Hiera's configurable backends to determine how,
and in which order, a particular binding ought to be resolved; many users search first in the encrypted `hiera-gpg` backend
for secure data like database passwords, then fall through to an unencrypted YAML backend if no encrypted key is found.

Examples of Current Injection-Like Behavior in Puppet
-----------------------------------------------------
Puppet already has some injection like mechanisms, some (like hiera) perform "true injection", and others deal with
similar problems with varying degree of success in separating caller and called logic and providing detailed control
of the mechanism:

* Hiera 1 -- injects data, classes and parameters for parameterized classes, and does explicit injection
* puppet.conf and command line settings -- inject data configured per environment
* Resource defaults -- inject values to resources which don't provide more closely-scoped overrides
* ENC -- injects environment, classes with or without parameters, and top-scope variables
* `site.pp` -- "injection" of classes and variables based on node name matching (a form of categorization)
* Spaceship operators that query virtual or exported resources and "inject" results

Hiera2
------
Hiera2 is a new implementation based on the ideas in Hiera and general injection frameworks such as 
[Google Guice](http://code.google.com/p/google-guice/), which is the leading injection framework for Java.

Hiera2 consists of the following parts:

* Categorization of Requests
* Composition of Bindings
* Bindings
  * named data
  * parameters and parameter mapping
  * producers
* Explicit Injection
* Implicit Injection
* API for use in Ruby
* API for binding provider extensions
* API for Puppet Extension Points
* Models (used by framework and for system interchange)

Categorization of Requests
--------------------------
The Puppet System needs to be able to create and configure an injector at the start of processing a request (and here we mean 
a request in the most general sense: asking the system to do something). This injector is then used throughout the servicing of
that request.

Since we want to be able to express bindings hierarchically (searching for something specific first, then falling back to
more generally applicable defaults), and we want users to be able to define their hierarchy based on their organization's needs, 
we require a flexible, user-configurable framework. In Hiera2 this is called Categorization and it ie expressed using the 
Puppet DSL. Before jumping into the details,
examples of categories are "data_center", "rack", "physical_hw", "security_zone", as well as the built in categories
for "node", "environment", and "common".

Before explaining categorization (which may vary per environment) we must first define how the system establishes
which environment to use, and so we'll first look at the sequence of processing for a catalog request.

### Request Sequence

In Puppet 3.x the sequence of setting up and processing a catalog request to the point where evaluation in the 
root (site.pp) manifest takes place is somewhat complex and includes murky areas. As a side effect of implementing 
Hiera 2 it is of great value to also clean this up. The proposed sequence is as follows:

* request reaches new node terminus
* all facts are turned into top scope variables (or, optionally, put into a `$facts[]` hash)
* the $environment variable if set is cleared (it is unsafe in its current form)
* if the request contains trusted data (from cert or encrypted facts), these are made available using a mechanism
  that is to be specified separately (i.e. via a different mechanism than just setting $environment in top scope).
* if an external (current API) ENC is configured it is called to obtain environment and classes data, these are
  processed a special way - see [Integration with Existing ENC](#integration-with-existing-enc)
* settings refer to a single manifest that enumerates the available environments, and for each environment the modulepath and
  environment root manifest.
* the entry point manifest can set $environment
* if $environment is not set, it defaults to "production"
* if the entry point manifest is not present, the environment defaults to "production"
* if nothing else was specified, the execution delegates to the site.pp in the same location as the entry point manifest.

### environments.pp

The entry point manifest is named `environments.pp` and is placed in the same directory as `puppet.conf`.

> Decision: Should environments.pp be a hard-coded name, or be configurable in puppet-conf?
> Pro hard-coded: less confusion
> Pro configurable: easier to test different overriding configuration (dev/test)

The concrete syntax for an environment is specified by borrowing the resource syntax. It is made special by
being placed inside of a `site { }` expression.

    site {
      # Production environment
      environment { 'production':
        modulepath  => 'colon separated path',
        manifest    => 'path to environment root manifest',
        manifestdir => 'directory',
        templatedir => 'directory',
      }
      # Additional environments
      environment { '...':
        # ...
      }
      # Environment defaults
      Environment {
        modulepath => "colon separated paths which may use ${environment}",
        manifest   => "path to environment root manifest which may use ${environment}",
        # ...
      }
    }
    $environment = 'production'

> Note: The rationale is that things inside the new top level keyword `site` are different, and that
> this top level keyword contains other new keywords that are not special outside of "site". This reduces
> potential clashes with existing logic. It also logically relates these kinds of statements with other
> statements about the "site" (as you will see in later sections).

The following rules apply:

* It is no longer possible to introduce stanzas in puppet.conf for environments. (There are several issues related to
  this as any stanza that is not one of the known stanzas become an environment stanza, and only some settings can be
  made per environment).
* Restriction on environment names is lifted (it is ok, but still not recommended to use the previously restricted
  names `main`, `master`, `agent` or `user`).
* If `environments.pp` is missing, one default "production" environment is added by default, with default values for all
  parameters.
* If `environments.pp` exists, and it does not contain a "production"
  environment, it is added by incorporating `Environment` resources defaults in addition to
  default puppet values for all parameters.
* A defined "production" environment is authoritative.
* Any number of environments may be added
* When execution reaches the end of the `environments.pp` and `$environment` has not been defined, it is set to "production".
* Dynamic environments may be enabled or disabled via a `dynamic_environments =
  <bool>` puppet configuration parameter. It defaults to `true`.
* If `$environment` is set to a value other than one of the defined
  environments and dynamic environments are enabled, then an environment of the
  given `$environment` value is added by incorporating `Environment` resource
  defaults in addition to default puppet values for all parameters.
* If `$environment` is set to a value other than one of the defined
  environments and dynamic environments are disabled, an error is raised and
  processing stops.
* The RHS parameter values may be any non top level puppet expression, but may not use injection
* facts and secure data have been bound to variables when evaluation of `environment.pp` starts
* any variables set in `$environment.pp` are set in a local scope not visible to the rest of the system.(with the exception of
  $environment which is picked up from this scope).
* the `environment.pp` may contain other puppet expressions (non top scope) - the use case is to support data
  manipulation, common definitions reused in several environment specifications, etc.
* only functionality available in puppet core may be used (functions and plugins/extensions) since the environment, and
  hence no module-path, has been setup.

> Discuss: Since the new evaluator will return the value of the last executed expression as the result, we could simply
> return a value that way, but it is slightly magical in this context.

> Discuss: A parameter value of `default` could be used to initiate the parameter to a sane/typical default in an
> environment.

### Integration with existing ENC

The use of the current ENC API requires special handling as it is believed that users
that have an ENC will want to use it in parallel with the new functionality until they have completely migrated. The current
ENC may be deprecated, or re-purposed.

The ENC can return a list of classes, and the environment to use. If these are produced, a local scope is created where these
values are bound to `$classes`, and `$environment`. Yet another local scope (with visibility into the local scope created for
the ENC (if any)) is created where the evaluation of the `environment.pp` body takes place. It is thus possible to refer to the
ENC set $environment or set it to a new value. The resulting `$environment` is the visible value at the end of the evaluation
of `environments.pp` (before the two scopes are abandoned).

> Discuss: It is probably of questionable value to have access to the defined classes since environments.pp only
> influences the selection of an environment.

The binding of classes to nodes are done using the Hiera2 Layering mechanism; see TBD "layers" and "enc binding provider"

### Categorization

When system has finished computing `environments.pp´ it delegates to the manifest specified by the selected
environment's `manifest` property. This allows further configuration of the injector to be created for the request.

The categorization syntax looks like this:

    site {
      categories {
        # TBD - Copy from google doc
      }
    }

TO BE CONTINUED...

### Summary of Categorization

The above design should be powerful enough to deal with:
* An existing ENC that determines the environment
* Blocking unsafe `$environment` from facts
* Allowing user logic to determine what $environment should be set to since the logic can consider:
  * $environment set by an ENC
  * environment passed as secure data via mechanism defined in a future ARM
  * Logic involving other facts/fqdn/certname, using general puppet logic such as case/if/pattern matching that results
    in the $environment to use.
* TO BE CONTINUED...

In essence, the "power of an ENC" moves into the Puppet language and is given a richer set of data as the basis for the
decision.

IT ALSO TO BE CONTINUED...

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

Describe all dependencies that this ARM has on other ARMs, components,
products, or anything else.  Dependences upon other ARMs should also
be listed in the "depends:" field in the metadata.json.

Describe any ARMs that depend upon this ARM before they can be implemented.

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
