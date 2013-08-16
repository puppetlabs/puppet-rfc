ARM-9 Data in Modules
=====================

This ARM is focused specifically on the problem of providing
Hiera data in Puppet modules.

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

The functionality consists of:

- **Hiera-2** - a reimplementation of Hiera (referred to as Hiera-1)
- **Composition of data** across multiple sources (most notably modules)
- **Implicit lookup** - (parameters for parameterized classes). Can combine Hiera1 (at
  site level) with Hiera2 from modules (or use Hiera-2 throughout). Hiera-1 is
  not supported inside modules.
- **Explicit lookup** via function

Terminology
===========

If you like examples more than theory, you may want to skip to the next section, or even go directly to the Examples
and come back here when a term is unclear.

**Binding** - conceptually this term means "value bound to a key". If you are familiar with Hiera
already; when writing apache::port: 80 in a hiera data file, this is a binding that binds the value 80 to the key/name apache::port.

**Bindings** - simply a collection of Binding

**Category** / **Categories** - if you are familiar with Hiera already, this is the term chosen to denote an entry
in the hierarchy — you can use it interchangeably with the term hierarchy, but the hierarchy term is difficult to
use when talking about a specific entry in the hierarchy – the best English word seems to be 'strata' - which is a bit
ceremonial, and the term "category" was instead chosen. The term makes it easier to talk about
them e.g.
*'the node category has higher priority than the environment category*',
*'In which category did we find the value for the key 'apache::port'?'*,
*'Do we have a category for data-center?*'.
The term was also chosen because there are multiple things that makes up the total "hierarchy" of things; there is
data at the site level as well as the module level, and it is possible to do overrides. The term "hierarchy" was kept
when dealing with Hiera based data (for sake of familiarity), but it is referred to as "categories" instead of "hierarchy"
when dealing with the configuration of the overall system.

**Layers** - this is simply the top-most specification of the hierarchy; as you will see later, this defines that site may override
modules, and if you like, that some modules may override others.

Hiera-2
=======

Hiera-2 is different from Hiera-1 in the following ways:

- The `hiera.yaml` has more information.
- Interpolation uses Puppet language interpolation (syntax and expressive power).
- All decisions about composition of data are made "in the data" (by the author of the
  data), not when looking things up.
- Only the equivalence of Hiera-1’s *priority lookup* is supported in the first   
  implementation.

There are also internal changes that affects those that want to write backends.

The hiera.yaml
--------------

Hiera 2’s config file looks like this:

    ---
    version: 2
    hierarchy:
      [ ['osfamily', '${osfamily}', '${osfamily}' ],
        ['common', 'true', 'common' ]
      ]
    backends:
      - yaml
      - json

*(NOTE: there are several ways to write this in YAML, the notation above is shorter than an explicit listing of each
entry using '-' ; what to use is up to the author, as long as the file is valid YAML and contains an array of arrays of three strings).*

The hierarchy is no longer a simple list of paths to search (as in Hiera1), instead an array of arrays is used where each
array describes three things:

- the **name** of the “category”
- the **value** / expression of that category (e.g. ‘kermit.example.com’ in the
  category ‘node’)
- the **path** to the Hiera-2 data file

*What are these used for? Why the change?*

The **category name** (i.e. ‘node’, ‘environment’, ‘common’) is used to ensure that all contributed data bindings
bind them with the same priority. Since all contributions are composed into one coherent set of bindings it would be
very bad if modules did not have the same relative order.

The **category value** is there to enable checking that all modules have the same notion of what it means
to be in a category. Some providers of bindings may filter the result (of what they contribute) based on
these values, and it would be bad if bindings were contributed that would never take effect.

The **path** is there (just like in hiera1) to point the implementation to the data file that contains the values
for this particular category. The path is relative to the location of the directory containing the hiera.yaml file.

Three categories are predefined; `node`, `environment` and `common` - these have to have this
relative precedence (node having the highest priority (of the three), and common the lowest).
Note that “everything” is in the common category.

(The implementation may change to allow ‘common’ to be specified without the category value since it
is meaningless; although a path to a file still is. By convention, the value should be set to `‘true’`).

The **backends** array has the same semantics as in Hiera-1. The first implementation has support
for `json` and `yaml`, and it is possible to plugin/extend the system with Hiera-2 backends (as well as extending
the system in other ways (a backend is not always the best choice).

Hiera Data
----------

The data files (in json or yaml) work the same way as in Hiera-1 except that interpolations are written using Puppet
syntax (the future parser is used for this). This means that `${}` should be used instead of the `%{}` syntax
for interpolation in Hiera-1.

This also means that any expression can be used, not only variable references. Most notably it is possible to lookup a value!

Here are some examples:

    some_key: 'Hello ${some_var}'
    some_other_key: 'Hello ${some_array[2]}'
    yay: 'Hello ${lookup(akey)}'

Composition
===========

The composition of the bindings is driven by a configuration. If nothing is specified, a default configuration will be
used (as shown below). If there is a `$confdir/binder_config.yaml`, it will be used instead of the default, and
finally, if the setting `:binder_conf` is set to point to a yaml file, that file is used instead of any
existing `$confdir/binder_config.yaml`.

The `binder_config.yaml` defines how all contributions to the  bindings are composed. The default looks like this:

    ---
    version: 1
    layers: [
      { 'name' => 'site',
        'include' => ['confdir-hiera:/', 'confdir:/default?optional']  
      },
      { 'name' => 'modules',
        'include' => ['module-hiera:/\*/', 'module:/\*::default']
      },
    ]
    
    categories: [
      ['node',        "${fqdn}"],
      ['osfamily',    "${osfamily}"],
      ['environment', "${environment}"],
      ['common',      "true"]
    ]

There are two main entries; **layers** (which defines the layering of contributions), 
and **categories** (which defines the names of valid categories, and their priority). 
Contributions in a layer with higher priority completely override those in lower layers (i.e.; by default - anything
defined at site level overrides everything contributed from modules. Note, there is nothing preventing
bindings from the `$confdir` to placed in the same layer as contributions from modules, and only have site-wide
overrides in the top layer).

The **categories** is a list in priority order (highest priority first) giving the name of the category, and the
expression that evaluates to the category value. That is; all categories used in all modules must have a corresponding
entry in this category section - but there are no paths specified.

Note: Although this may look as a chore at first, it is expected that almost all public modules only contain default
values in the common category. It is not an error to include a category that is not used in a particular
configuration; so once your organization has defined the various categories to use, they can stay unchanged.

Note about the category value expression: it should use variable names without `'::'` (i.e. for facts) as the
evaluation takes place in top scope, and it is slightly more efficient to leave out the `'::'` (it works to enter
them with '`::'` as well e.g. `'::fqdn'`. For lookup in the `::settings` namespace there is no difference in speed).
At the point of evaluation of categories, only facts and `::settings` are initialized, and the top scope has not yet
finished its evaluation (i.e. the `site.pp` manifest has not yet been evaluated).

The **layers** describe which contributions to include in the resulting bindings. Contributions in higher layers have
higher priority; their contributions win over those in lower layers even if the priority of an individual binding is
lower that the corresponding binding in a lower layer - i.e. this is an override mechanism that is used to “repair” things
that are broken, or ensure that certain bindings can not be specified by contributions from the lower layers.
More simply put; the simplest use-case is to  allow you to override the contributions from the modules at the site level.

The number of layers to use is up to you; maybe you want your organization’s modules to have higher priority than
public ones, maybe you have some that are more sensitive than others etc.

A layer is specified with:

- **name** - a name that is used only in information output (error messages etc.) - it
  has no effect on the composition of data.
- **include** - a list of bindings provider URIs that denote the bindings to include
- **exclude** - a list of bindings provider URIs that should not be included

The version has has to be specified, but the remaining sections are optional and will get default values.
If a section is given it must be given in full.

Schemes
-------

The bindings provider URI is specific to the used scheme. In the first implementation there are four such schemes:

- `confdir-hiera`
- `module-hiera`
- `confdir`
- `module`

### Hiera-2 Schemes

The `confdir-hiera` scheme is used with a path that is relative to the *confdir*. The path should appoint a
directory where a `hiera.yaml` file is located. If the `hiera.yaml` for the site is located in the same directory as
the `binder_config.yaml`, the entry would simply be `confdir-hiera:/`

The `module-hiera` scheme is used with a path where the first entry on the path is the name of the module,
or a wildcard `*` denoting all-modules. When using the wildcard, those contributions that have a `hiera.yaml` in
the appointed directory will be included (those that do not are simply ignored). The path following the module name
is relative to the module root.

Thus, the URI `module-hiera:/*` uses the `hiera.yaml` in the root of every module. The URI `module-hiera:/*/windows`
loads from every modules' windows directory, etc.

It is expected that `include` is used with broad patterns, and that a handful of exclusions are made (broken/bad module
data, or data that simply should not be used in a particular configuration).

When the URI contains a wildcard, and there is no `hiera.yaml` present that entry is just ignored. When an explicit
URI is used it is an error if the config file (or indeed the module) is missing.

Module / Confdir Scheme
-----------------------

In the first release of data-in-modules, the `module` and `confdir` schemes are only used to refer to bindings
made in Ruby. In later releases these schemes will be used to also refer to bindings defined in the Puppet Language
(as explored in ARM-8 Puppet Bindings). You can safely skip the rest of this section on a first read through if you
want to concentrate on the features provided when only using the much simpler Hiera-2 structures.

The `module` and `confdir` schemes use symbolic references - e.g. `module:/modulename::bindingsname`. In the first
implementation it is possible to define such bindings in Ruby by placing a file matching the qualified name under
`lib/puppet/bindings` - in the example `<module>/lib/puppet/bindings/modulename/bindingsname.rb` and then defining the
binding using the Ruby class `Puppet::Bindings` like this:

    Puppet::Bindings.newbindings("modulename::bindingsname") do
      bind.name("modulename::foo::bar").to(42)
    end

This way of binding is much more powerful than the Hiera-syntax as the full range of features available in the Puppet
Bindings system can be used.

The intention is to make the same capabilities available with concrete Puppet Language syntax as described in ARM-8.

The `module` and `confdir` schemes differ in that the `confdir` scheme does not support module wildcards, and internally
they handle *optionality* differently (they need to be able to determine if there is a file to load or not). For both
module- and confdir- schemes, the loading is done the same way - `confdir:/foo::bar` and `module:/foo::bar` is the same
named bindings, the only difference is the check for optionality (This is not ideal and should be addressed).

Optional Inclusion
------------------

To make inclusion more flexible, it is possible to define that an (explicit) URI is optional - this means that it is
ignored if the URI can not be resolved. The optionality is expressed using a URI query of `?optional`. As an example,
if the module ‘`devdata`’ is present its contributions should be used, otherwise ignored, is expressed as
`module-hiera:/devdata?optional`.

Implicit Lookup
===============

Implicit lookup of class parameters is done by first consulting the Puppet Bindings and then Hiera-1 (if present).
Bindings are done the same way as in Hiera-1 - i.e. binding `<fully qualified class name>` `::` `<parameter-name>` to a value.

Explicit Lookup
===============

Explicit lookup is done with (surprise...) the function `lookup`. It takes one or two parameters, the first is always the
name to lookup, and the second optional parameter is a type description (a string) that when used will assert that the
returned data is type compliant.

    lookup('apache::port')
    lookup('apache::port', 'Integer')
    lookup('mything::list_of_users', 'Array[String]')

If the produced value does not comply with the given type an error is raised.

The Type System
===============

The type system contains both concrete types; `Integer`, `Float`, `Boolean`, `String`, `Pattern` (regular expression pattern),
`Array`, `Hash`, and `Ruby` (represents a type in the Ruby type system - i.e. a Class), as well as abstract
types `Literal`, `Data`, `Collection`, and `Object`. The `Array` and `Hash` types
are parameterized, `Array[V]`, and `Hash[K,V]`, where if `K` is omitted, it defaults to `Literal`, and if `V` is omitted,
it defaults to `Data`.

The `Ruby` type (i.e. representing a Ruby class not represented by any of the other types) does not have much
value in puppet manifests but is valuable when describing bindings of puppet extensions.
The Ruby type is parameterized with a string denoting the class name - i.e. `Ruby['Puppet::Bindings']` is a valid type.

The abstract types are:

- `Literal` - `Integer`, `Float`, `Boolean`, `String`, `Pattern`
- `Data` - any `Literal`, `Array[Data]`, or `Hash[Literal, Data]`
- `Collection` - any `Array` or `Hash`
- `Object` - any type

The bindings system uses the type system to infer the type of bound data, as key when performing a lookup, and to
assert type compliance of produced results.

Examples
========

This section contains a series of examples starting with the simplest possible configuration that makes use of data in modules.

NOTE: In order to activate the "data-in-modules" and "Hiera-2" it is required to:

- Have the gem rgen >= 0.6.5 installed
- Use one of these settings (in puppet's config or made from the command line).
  Neither is on by default in Puppet 3.3.x
  - `--binder`
  - `--parser future`

Note: The opt in with `--binder` basically exists to avoid adding a required runtime dependency to Rgen for
those that upgrade their puppet installation (i.e. it would in a way be a breaking change). The future parser
however already depends on Rgen so it also turns on data-in-modules. The future parser will also always use the
binding systems to handle extensions (e.g. the puppet templates and puppet heredoc makes use of extensions for syntax
checking; the bindings/injector is not just for application data).

Opting in on Hiera-2 'Data in Modules'
--------------------------------------

First, `--binder` or `--parser future` must be in effect.

The, in order for a module to be able to contribute bindings (i.e. data) using Hiera-2 and the default configuration,
the module needs to have a `hiera.yaml` in its root directory. (This is not entirely true; to opt in using bindings
expressed in Ruby there is no need to have a `hiera.yaml` file). Also as you will see later, you can modify the overall
composition to place the module's `hiera.yaml` elsewhere, or have alternatives for the user to choose from. The convention
is that the defaults for a module should be in the root of the module, and optional/alternative configurations placed elsewhere.

This `hiera.yaml` is needed because it:

- indicates that there is data that can be used in the module
- tells the system how the data is organized

Hiera-2 has sane defaults, so all that is required is a file in the root of the module

**hiera.yaml:**

    ---
    version: 2

Yes, that is all that is needed to be able to express common bindings (and bindings for osfamily).
The default configuration declares that all data is in a directory called `data`, and that *common* is
in `data/common/`, and *osfamily* is in `data/osfamily/`. You also get both `yaml` and `json` backends by default
where `yaml` has higher priority.

At this point, we have not defined any data in the module, but this can now be added.

Adding a common (default) data
------------------------------

To define that 'has_funny_hat' should have the value 'the pope' in general we add the following file:

**data/common.yaml:**

    ---
    has_funny_hat: 'the pope'

You can now obtain this data anywhere in your puppet logic, here is quick way to check:

    puppet apply -e 'notice lookup(has_funny_hat)'

which results in this output

    Notice: Scope(Class[main]): the pope

Adding an override for osfamily "darwin"
----------------------------------------

Now, when running on OS X (osfamily "Darwin"), we want the value to be different.
Since the default already places the category "osfamily" at higher priority than "common" and
has defined where it is supposed to be found, we simply add the following file/content.

**data/osfamily/darwin.yaml:**

    ---
    has_funny_hat: 'steve martin'

and then run:

    puppet apply -e 'notice lookup(has_funny_hat)'

the output is:

    Notice: Scope(Class[main]): steve martin

That is, if you are running this on an OS X machine (Darwin osfamily), otherwise you would get 'the pope'.

Adding categories
-----------------

It is expected that almost all modules only contain default data, or default data for an os family.
This because in general modules (especially those published by others) cannot know anything about the names
of your environments, nor the names/identities of your nodes, data centers etc.

If you however have your own modules you may want to add data for a category with values specific for your organization.

When adding a category, you need to specify the entire section (i.e. hierarchy since this is a hiera configuration).

**hiera.yaml:**

    ---
    version: 2
    hierarchy:
      [['osfamily', '\$osfamily', 'data/osfamily/\$osfamily'],
       ['environment', '\$environment', 'data/env/\$environment'],
       ['common', 'true', 'data/common']
      ]

Since "environment" (and also "node") are already defined in the default `binder_config.yaml`, this is the only
change we have to make in order to define data for a specific environment. We can now add a data file for an environment:

**data/env/production.yaml:**

    ---
    has_funny_hat: 'comedians'

Now, we get 'comedians' as default when running in production (except for 'darwin' where we get 'steve martin').

Note that the `hiera.yaml` only needs to list the categories/hierarchies that it is using, and those that are
used have to be present in the `binder_config.yaml`, and have the same relative order (i.e. you are allowed to skip
categories that are not in use, but not change their relative order).

Performing Type Assertion
-------------------------

When doing a lookup it is possible to also state the expected return type, which will be asserted by the lookup
function (i.e. raise an error if the result does not comply with the specified type).

To assert that the result is a String:

    lookup(has_funny_hat, 'String')

To assert that the result is an array of integers:

    lookup(port_numbers, 'Array[Integer]')

To assert that the result is a Hash with String keys and values being arrays of integers:

    lookup(named_port_numbers, 'Hash[String, Array[Integer]]')

and so on...

The type specification is one of:

- the basic types; `Integer`, `String`, `Float`, `Boolean`, or `Pattern`'
  (regular expression)
- an `Array` with an optional element type given in `[]`, that when not given
  defaults to `[Data]`.
- a `Hash` with optional key and value types given in `[]`, where key type defaults to 
  `Literal` and value to `Data`, if only one type is given, the key defaults to  
  `Literal`.
- the abstract type `Literal` which is one of the basic types
- the abstract type `Data` which is `Literal`, or type compatible with
  `Array[Data]`, or `Hash[Literal, Data]`.
- the abstract type `Collection` which is `Array` or `Hash` of any element type.
- the abstract type `Object` which represents an object of any type

Binding default values for parameterized classes
------------------------------------------------

Just like in Hiera-1, it is possible to bind default values for parameterized classes. This now also works for data in
modules. The fully qualified name of the parameter should be used when binding a value - e.g:

    'apache::port': 80

Which binds the parameter port to 80 for the class apache.

Note: There is no limitation on where parameters are defined; one module may provide default parameters for classes in other
modules. (Which leads us to the next example).

Lookup with default
-------------------

If you are looking something up and want to use a default specified in your puppet code if the lookup did not result in
anything you can do this by...

In puppet 3 by something similar to:

    $x = lookup('something')
    $looked_up = $x ? { undef => 'nothing', default => $x }

When `--parser future` is used, a lambda can instead be used:

<pre>
# lookup and provide default
$looked_up = lookup('something') |$result| {
  if $result {$result} else {'nothing'}
}

# lookup and fail if not defined
$looked_up = lookup('something') |$result| {
 unless $result { error('data for something is missing')}
 $result
}
</pre>

Overriding bindings from modules
--------------------------------

If we find that two modules define a value for the same name; say 'apache::port' - which one should be used if they
are defined with the same priority?

We can solve this in two ways, have a module that binds a value at a higher priority, or more practical, bind a value at the
site level; since these bindings by default override all bindings contributed from modules.

If Hiera-1 is not in use, simply add a `hiera.yaml` in the environment/confdir root:

**hiera.yaml:**

    ---
    version: 2

and add the file:

**data/common.yaml:**

    ---
    'apache::port': 8080

By doing so, we have now overridden the (conflicting) bindings from the modules layer.

Making overrides in a separate layer
------------------------------------

We may want to keep the bindings done for the sole purpose of overriding modules in a separate layer.
The reasons could be that we do not want to mix them up with the regular site level bindings as it can become
difficult to entangle them later (not knowing why certain bindings have been made).

We do this, by creating a `bindings_config.yaml` in the confdir as we can no longer use the default configuration,
and specify it like this (a comment shows what is added to the default):

**bindings_config.yaml:**

    ---
    version: 1
    layers:
      [{name: site, include: 'confdir-hiera:/'},
       {name: overrides, include: 'confdir-hiera:/overrides'}, # <--added
       {name: modules,
        include: ['module-hiera:/\*', 'module:/\*::default']}
      ]

We do not have to enter the categories, since we are not going to change those from the default.

We can now define the data for overrides under 'overrides'. This is done by adding:

**overrides/hiera.yaml:**

    ---
    version: 2

**overrides/data/common.yaml:**

    ---
    'apache::port': 8080

Excluding a module
------------------

We find that there is something wrong with a particular module. It may have syntax errors in the data or it has
definitions of data that we are not happy with in general - in short, we just do not want the data it contributes by default.

To do this, we need a `bindings_config.yaml` (the default does not exclude anything). As an example, assume
we want to exclude the default Hiera-2 data from module 'bad'

**bindings_config.yaml:**

    ---    
    version: 1
    layers:
      [{name: site, include: 'confdir-hiera:/'},
       {name: overrides, include: 'confdir-hiera:/overrides'}
       {name: modules,
        include: ['module-hiera:/\*', 'module:/\*::default'],
        exclude: 'module-hiera:/bad'}
      ]

Shipping a module with alternative bindings
-------------------------------------------

If you are publishing a complex module - say one that is configurable to be used in combination with other
modules or stand alone and you want to help users consume it by including different sets of data.

This is easily done by having one default (say for standalone) configuration one for each alternative.
You could organize this like this:

<pre>
&lt;module&gt;
|-- hiera.yaml
|-- data
|-- |-- default
|   |   |-- common.yaml
|-- |-- for_x
|   |   |-- hiera.yaml
|   |   |-- data
|   |   |   |-- common.yaml
|-- |-- for_y
|   |   |-- hiera.yaml
|   |   |-- data
|   |   |   |-- common.yaml
</pre>

Here, the `<module>/hiera.yaml` uses paths that points to '`data/default`',and the `hiera.yaml`
in '`for_x`' and '`for_y`' uses path starting with '`data`' (i.e. relative to the respective `hiera.yaml` file).

The documentation for the module needs to explain that it comes with different configuration options.
The user of the module then selects which one to use (or several at the same time if that makes sense).

If there is a default contribution that is mutually exclusive the user needs to add it to the excluded list
before including one of the alternatives. The user may end up with these two entries in their modules layer
in `bindings_config.yaml`:

    include: [... , 'module-hiera:/the\_module/for\_x' ],
    exclude: [... , 'module-hiera:/the\_module]

If you have complex data you may want to consider defining this in ruby instead as you can then define everything
per option - i.e. `default.rb`, `for_x.rb`, and `for_y.rb` and have them all in the same place.

Going Advanced
==============

Ruby Examples
=============

The Puppet Binding system is capable of binding data in more advanced ways than what can be expressed with simple
Hiera definitions. (Adding these capabilities to hiera data would make the regular (straight forward) cases far more
complicated to enter. Instead, the capability to define bindings in Ruby was added. Later it is expected that the
same capabilities are made available directly in the Puppet Language based on the ideas in ARM-8).

You have already seen the URIs module: and confdir: in the examples above, and you may have wondered what they refer
to. These schemes use symbolic names rather than paths, and they are capable of loading ruby logic using conventional
ruby name to path conversion.

The only difference between the two schemes is that the module scheme supports wildcard notation for the module name.

You do not have to modify the default configuration to opt-in since the defaults are to include all ruby based default
bindings from modules and the confdir. All that is needed is for a ruby file defining bindings to be in the expected
location with the expected name.

The ruby bindings are loaded using the auto loader, and may thus be loaded from the confdir, modules on the module
path, gems, or directories on the configurable load PATH.

Who has a funny hat in Ruby?
----------------------------

In our module we can define a data binding using ruby.

**&lt;module&gt;/lib/puppet/bindings/mymodule/default.rb:**
<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind.name('has_funny_hat').to('the_pope')
end
</pre>

This can also be written as:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind {
    name 'has_funny_hat'
    to 'the_pope'
  }
end
</pre>

The location in the file system and the name of the resulting binding must match. By convention, all bindings
from modules should have a fully qualified name that starts with the module name. For bindings in the
confdir/an-environment care must be taken with naming; using either unqualified names (like default.rb), or a name
that does not clash with any module names.

Categorized bindings in Ruby
----------------------------

In ruby, we do not have to store data separately for the different categories, instead we can organize the data
per concern (i.e. have the bindings that have a common purpose in one file, and bindings for a different purpose in another file).

The examples above where different people got to be the person having a funny hat can all be written in one place.

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind {
    name 'has_funny_hat'
    to 'the_pope'
  }
  
  when_in_category('osfamily', 'darwin') {
    bind {
      name 'has_funny_hat'
      to 'steve martin'
    }
  }
  
  when_in_category('environment', 'production') {
    bind {
      name 'has_funny_hat'
      to 'comedians'
    }
  }
end
</pre>

Using Multibinding
------------------

You can use *Multibinding* to collect data into an array or a hash. To do this the multibind is first declared, and
the bindings that should be included are contributed by stating that they are made in the scope of the multibind.
It may sound more complicated than what it is... here is an example:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  multibind('collected-users') do
    name 'users'
    hash_of_data #determines the resulting type
  end

  bind.in_multibind('collected_users') do
    name 'fred'
    to 'Fred Flintstone'
  end

  bind.in_multibind('collected-users') do
    name 'mary'
    to 'Mary Poppins'
  end
end
</pre>

As you may have guessed, the individual contributions can have different priority, and the binding with the highest
priority of a given contributed key wins.

The multibindings take additional parameters that are passed as options (default in parentheses) - these are the
options for a hash based multibinding:

- `:conflict_resolution` (`:priority`) is one of `:error`, `:merge`, `:append`,
  `:priority`, `:ignore`
  - `ignore`, the first found highest priority contribution is used, the rest
    are ignored
  - `error`, any duplicate key is an error
  - `append`, element type must be compatible with Array, makes elements be arrays and
    appends all found
  - `merge` element type must be compatible with hash, merges hashes with
    retention of highest priority hash content
  - `priority`, the first found highest priority contribution is used, duplicates with
    same priority raises and error, the rest are ignored.

- `:flatten` (`false`), If appended elements should be flattened. If argument is a
  number it is used as the max level of flattening, a value of `true` is the same as `:flatten => -1`
- `:uniq` (`false`), If appended result should be made unique (i.e. remove duplicates)
- `:transformer` (`nil`), A Ruby or Puppet lambda that is called to transform the produced value
  before it is returned to the caller. The Ruby lambda gets scope and value, the puppet lambda gets
  the value. (All producers support this option)

Here is an example that collects all entries bound in 'collected-users' in a resulting merged hash.

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  multibind('collected-users') do
    name 'users'
    hash_of_data
    producer_options { :conflict_resolution => merge }
  end
</pre>

This example collects and merges users like the previous example, but removes all users in the 'banned-users' list of users.

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  multibind('collected-users') do
    name 'users'
    hash_of_data
    producer_options {
      :conflict_resolution => merge
      :transformer => lambda do |scope, value|
           banned = scope.compiler.injector.lookup('banned-user')
        value.delete_if {|k,v| banned.include? k }
      value
      end
    }
  end
</pre>

And yes, you can contribute multibinds to mutibinds if you need several levels of collection.

For completeness, here are the options for an array based multibind:

- `:uniq` (`false`) if collected result should be post-processed to contain only unique entries
- `:flatten` (`false`) if collected result should be post-processed so all contained arrays are 
  flattened. May be set to an Integer value to indicate the level of recursion (-1 is endless, 0 is 
  none).
- `:priority_on_named` (`true`) if highest precedented named element should win or if all should be 
  included
- `:priority_on_unnamed` (`false`) if highest precedented unnamed element should win or if all should 
  be included

- `:transformer` (`nil`), A ruby or puppet lambda that is called to transform the produced value 
  before it is returned to the caller. The ruby lambda gets scope and value, the puppet lambda gets 
  the value. (All producers support this option)

Looking up another value
------------------------

Say you want `myclass::service::port` to have the same value as what is bound to `apache::port`. This is how you achieve this:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind do
    name 'myclass::service::port'
    to_lookup_of(type_factory.integer, 'apache::port')
  end
end
</pre>

A user can naturally bind '`myclass::service::port`' to something else with higher priority.

Looking up first-found value
----------------------------

Say you want to bind the port parameter to either `apache::port`, or `nginx::port` (you know one will be used, but not both).
Here is how you do this:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind do
    name 'myclass::service::port'
    integer
    to_first_found('apache::port', 'nginx::port')
  end
end
</pre>

The list to `first_found` may be an array of arrays specifying type, name combination. We did not need this in the
example above, since the type of the binding is integer, and an assertion is made that the result complies with the type.
We could have written the longer:

    to_first_found([T.integer, 'apache::port'], [T.integer, 'nginx::port'])

The result would be the same if there is no error, but in the later case an error is reported for the looked up
(`apache`, or `nginx`) instead of for the '`myclass::service::port`' lookup.

You may also provide a default value using a transformer option (i.e. if neither of `apache::port` not `nginx::port`
were defined, the example below produces the value 80).

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind do
    name 'myclass::service::port'
    integer
    to_first_found('apache::port', 'nginx::port')
    producer_options(:transformer => lambda do |scope, value|
      value or 80
    end
  end
end
</pre>

Going Even More Advanced
========================

The following sections has information if you want to dig deeper into the bindings system. This information is
intended for Ruby developers.

Where to find information about the methods available in newbindings.
---------------------------------------------------------------------

The block given to the `Puppet::Bindings.newbindings` method is evaluated in an anonymous class inheriting from
`Puppet::Pops::Binder::BindingsFactory::BindingsContainerBuiilder`. This enables calls to `bind` and `when_in_category` and a few others.

When calling `bind`, which also accepts a block, the evaluation takes place in a `BindingsBuilder`.
This class has the methods `name`, `type`, `to`, and a dozen other for more special forms of bindings.

The result produced by the block, must respond to the method `model`, which should return an instance
of `Puppet::Pops::Binder::Bindings::NamedBindings`. This enables you to call out to whatever logic you want
to produce the result. As an example, it could call to some external service and transform the result into a
bindings model (most easily done using the `BindingsFactory`).

#### Extending Data in Modules

There are several ways to extend Data in Modules with custom implementations in Ruby. This section is intended as
an overview of the capabilities.

-   Custom Hiera-2 backend
-   Custom bindings scheme handler
-   Custom producers

Custom Hiera-2 backend
----------------------

Custom Hiera-2 backends are implemented in Ruby and have a very simple API; given a directory and a filename
(without extension), its responsibility is to produce a Hash.

Implementing a Hiera-2 backend is easy, but does not give access to the more powerful features in the bindings system.
You can use this if you need to read simple name to value definitions in some file format.

Here is a toy backend that binds an echo of the input parameters. The example below should be placed in a module
(in the example, in the module 'awesome'). The method `read_data` should return a Hash with the wanted map of names to values.

**&lt;module-root&gt;/lib/puppetx/awesome/echo\_backend.rb:**
<pre>
require 'puppetx/puppet/hiera2_backend'
module Puppetx::Awesome
  class EchoBackend < Puppetx::Puppet::Hiera2Backend
    def read_data(directory, file_name)
      {"echo::#{file_name}" => "echo... #{File.basename(directory)}/#{file_name}"}
    end
  end
end
</pre>

This custom backend is registered in the `binder_config.yaml` like this:

    ---
    # categories, and layers
    # ...
    extensions:
      hiera_backends:
        echo: 'Puppetx::Awesome::EchoBackend'
    
Which can then be included in a `hiera.yaml` under the name '`echo`' like this:

    ---
    hierarchy:
      - ['node', '$fqdn', '$fqdn']
      - ['common', 'true', 'common']
    backends:
      - yaml
      - json
      - echo

Now, when executed for the node '`localhost`' and a lookup is made of '`echo::common`' or '`echo::localhost`',
the result is "`echo... awesome/common`", "`echo... awesome/localhost`".

(Yes, this is a toy example, the purpose is to show the API).

Custom Bindings Scheme Handler
------------------------------

A custom bindings scheme handler is responsible for interpreting an URI such as `module:/foo::bar` and turning it
into bindings. The scheme handler's responsibility is to process URIs for inclusion and exclusion when
wildcards/query/optionality is supported, and to produce a bindings model. (The example does not show wildcard expansion).

Here is a toy echo scheme handler example:

<pre>
require 'puppetx/puppet/bindings\_scheme\_handler'
module Puppetx::Awesome

# A binding scheme that echoes its path
# 'echo:/quick/brown/fox' becomes:
#   '::quick::brown::fox' => 'echo: quick brown fox'.
class EchoSchemeHandler &lt; Puppetx::Puppet::BindingsSchemeHandler
  def contributed_bindings(uri, scope, composer)
    factory = ::Puppet::Pops::Binder::BindingsFactory
    bindings = factory.named_bindings("echo")
    bindings.bind.name(uri.path.gsub(/\\//, '::'))
      .to("echo: #{uri.path.gsub(/\\//, ' ').strip!}")
    result = factory.contributed_bindings("echo", bindings.model, nil)
    end
  end
end
</pre>

The new '`echo`' scheme is registered in `binder_config.yaml` under the key `extensions`, `scheme_handlers`.
This example also shows the use of the echo scheme by including it in a layer.

    ---
    version: 1
    # categories
    layers: [ ...
              [name: testing, include: 'echo:/quick/brown/fox']
            ]
    extensions:
      scheme_handlers:
        echo: 'Puppetx::Awesome::EchoSchemeHandler'

With the content above, a lookup of `'::quick::brown::fox'` produces `'echo: quick brown fox'`.

(Yes, this is a toy example, the purpose is to show the API).

For a full description, look at the yard documentation for the classes.

Custom Producer
---------------

Both the Hiera-2 backend, and scheme handler extensions, deal with the production of bindings.
These bindings are created up-front before catalog compilation and they are therefore somewhat static in nature.
While it is possible to bind a puppet expression (that is evaluated either before compilation starts, or on each lookup),
'there may be the need to create a reusable value producer to keep the amount of repeated logic down, or to provide
features not already available from one of the existing producers.

Note that it is only possible to bind a name to a custom producer when defining the bindings in Ruby. The Hiera-2
backends only provide binding to constant values, and to string interpolation.

Using a custom producer is the most powerful way to integrate into the bindings system, but since it is feature
rich also a bit more complex. Examples of custom producers could be a producer that maintains a connection to another
system and that has a custom method of producing a value, other producers lookup this producer to use its API to perform
remote lookup. A custom producer could produce a series of unique values, or produce the same unique value after
having produced it once, etc. However, don't worry if this seems complex - these features are mainly available for
integrators. For users of custom producers, it is simply the matter of giving the name of the producer, and
declaring the options; i.e. not more difficult that calling a function.

While the producer sub-system has many features a simple producer implementation is quite small. All that is
needed is a class derived from `Puppet::Pops::Binder::Producers::Producer` that implements the
method `internal_produce(scope)`, and that this method returns the *"looked up value"*. The producer will
also need to implement the method `initialize` which is given information about the context where this producer
is being used, arguments defined in the bindings (the producer options), and the binding itself. A simple producer
that returns a constant value may look like this:

<pre>
class MyConstantProducer &lt; Puppet::Pops::Binder::Producers::Producer
   def initialize(injector, binding, scope, options)
      super
      @value = options[:value]
    end

def internal_produce(scope)
  @value
end
</pre>

When used from Ruby:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  bind {
    name 'has_funny_hat'
    to_producer(MyConstantProducer)
    producer_options(:value => 'the_pope')
  }
end
</pre>

If you are interested in the full capabilities, you will need to study the source of the Producer class and its
subclasses and read the yard documentation for the classes.

FAQ
===

Hiera
=====

I already use Hiera at the site level - How do I opt in on Hiera-2?
-------------------------------------------------------------------

If your Hiera-1 configuration is in `hiera.yaml`, the `confdir-hiera:` scheme will detect that it is not
Hiera-2-compliant and silently ignore it (and instead letting the existing Hiera-1 deal with the data).
If you want to use both Hiera-1 and -2 at the same time at the site level, simply alter the configuration
for Hiera-1 by using one of the alternatives:

By moving the Hiera 1 configuration:

-   rename `$confdir/hiera.yaml` to `$confdir/hiera1.yaml`
-   change the setting `:hiera_config` to `$confdir/hiera1.yaml`
-   add a `$confdir/hiera.yaml` for hiera-2, i.e.  a file that has `version: 2`

By changing the `bindings_config.yaml`

-   modify the site layer to reference `confdir-hiera:/hiera2` instead of the
    default `confdir-hiera:/`
-   put a `hiera.yaml` in `$confdir/hiera2`

Can I change the file name hiera.yaml to something else for hiera-2?
--------------------------------------------------------------------

No. This has to work the same way across all modules. The name of the file signals if it is contributing data or not.

An alternative would be to alter the meta-data in the module. This may be a future feature.

Can I continue using Hiera-1?
-----------------------------

Yes. Hiera-1 continues to work as before. The hiera functions will not lookup in bindings, and the lookup
function will not lookup using Hiera-1.

Parameterized classes will get their default from bindings first, and from Hiera-1 second (if present).

Can I call functions in interpolation?
--------------------------------------

Yes, this is possible, you can even call lookup, or one of the Hiera-1 lookup functions if you like.
(It protects you from doing endless recursion).

Can I use the same data files in both Hiera-1 and Hiera-2?
----------------------------------------------------------

Yes, the data files are compatible. However if you rely on Hiera-1 lookups that use something other
than priority lookup (e.g. `merge`, `merge deeper`, etc.) you will not get the same result from the
lookup function. If you need this, you should either stay with Hiera-1, or use the more advanced ruby bindings.

The reason for this is that decision about how data gets composed is defined in the bindings, not when looking up.

Can I contribute (host) classes using data in modules?
------------------------------------------------------

No. Not yet at least.

Can I set top level scope parameters using data in modules?
-----------------------------------------------------------

No. Not yet at least.

How do I handle different values for a combination of categories?
-----------------------------------------------------------------

Say I want osfamily 'darwin' to have different values for environments 'test' and 'production', how do I specify that?

There are two ways; by adding a combined category, or by defining the binding in Ruby.

Adding a combined category requires you to:

- add the hierarchy to hiera.yaml,  e.g.
    `['os-env', '${osfamily}-${env}', 'data/os/${osfamily}/${env}']`
- add the category 'os-env' to the `bindings_config.yaml` to indicate how this category
  is prioritized against all others (across all modules).
- write the data in `data/os/darwin/production.yaml`

Or in Ruby:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do
  when_in_categories({
    'environment' => 'production',
    'osfamily' => 'darwin'}) do
      bind {
        name 'has_funny_hat'
        to 'the pope'
      }
  end
end
</pre>

When doing this in Ruby, the result is automatically bound at a priority higher than the highest of the given categories,
but lower than the next higher category.

Can I bind a regular expression?
--------------------------------

Yes, in Ruby, but not in Hiera-2.

    bind.name('mypattern').to(/[a-z]+/)

Note: at the moment, the regular expression value is not that useful as match in the puppet language does not
accept an expression on the RHS, but it can be used with custom resource types that knows how to deal with a regexp,
or be passed to a custom function.

Advanced
========

Can I play with this now?
-------------------------

Yes, this is available on the Puppet master branch.

Can I extend the binding schemes?
---------------------------------

Yes, bindings schemes can be defined in the `bindings_config.yaml`

Can I add Hiera-2 backends?
---------------------------
Yes, Hiera-2 backends can be defined in the `bindings_config.yaml`

As an alternative you can instead contribute bindings directly using `Puppet::Bindings::newbindings`.

You will get more flexibility if instead writing a scheme handler as you can then control what the user can
specify as query parameters, fragments, etc. in the URI. A backend is only recommended if you want to provide
something that is path based and loads from the file system.

Can I write a puppet function that adds/alters the bindings?
------------------------------------------------------------

No, all the bindings are computed first. Once computed they are immutable. Calling functions happens much later.

So, can I do any dynamic binding?
---------------------------------

Yes, in Ruby, and accessing the scope (peek at the top scope variables and then define bindings suitable for the
particular catalog compilation). Also see "Can scope be accessed in newbindings?" below.

Can I access data in the current scope?
---------------------------------------

No. Access to variables interpolated into looked up data are always in top scope only. Anything else depends on the parse order.

Can scope be accessed in newbindings?
-------------------------------------

Yes, it is passed as a parameter for the block - e.g:

<pre>
Puppet::Bindings.newbindings('mymodule::default') do |scope|
  # logic
end
</pre>

The scope is always topscope. It may be used to only produce bindings that are relevant for the current compilation.

For small sets of data it is not meaningful to filter; simply produce all the bindings. If you however have very large
data sets, or do something advanced like calling an external service to get data, you may want to limit the request
to the particular node, osfamily etc. and not send information about every possible node over the wire in the response.


Testing and Evaluation
======================

Nothing special is required to test and evaluate.

Alternatives and Recommendation
-------------------------------

Continued use of Hiera-1 search based algorithm was considered, but rejected
because of poor composability traits.


Risks and Assumptions
---------------------

No particular risks identified.

Dependencies
------------

The implementation of this ARM is a subset of the proposed implementation described in ARM-8.
The implementation requires RGen.

Impact
------

How will this work impact other parts of the platform, the product,
and the contributors working on them?  Omit any irrelevant items.

- Other Puppet components: External tools that visualize or edit "data"
- Compatibility: The implementation is backwards compatibile wrt. Hiera-1
- Documentation: There is quite a lot to document
- Spin-offs/Future work: Continuing with concrete syntax for bindings (i.e. ARM.8).

