Using Hiera2 - User Stories
===========================

Akunna and Vatsan are going to try out the new features in Hiera2 to explore how they can deal with
configurations. They currently have hundreds of nodes hand crafted in a regular site.pp where they
allocate classes and set parameters. They do this in one site.pp that they use for both production
and development work. The site.pp is in constant flux. Vatsan works with the developers and helps them
update the configuration whenever there is a need.

They both hope that the new features will make it easier for them to separate the concerns; rather than
keeping all in one pile they really want the developers to be able to deal with their own machines.
They decide to pair up and refine the configuration as they learn about the features.

Akunna takes the lead while Vatsan is watching. They use a setup that is the equal to the production environment
to be able to test that everything works as expected.

Unboxing
--------
Akunna is happy because the existing configuration will run as before. That fits with their idea to make
changes in a stepwise fashion. They read up on what the defaults mean:

The default configuration will give users a single environment "production", with default reference to
`manifest` (i.e. `site.pp`), a default `modulepath`, and a default bindings configuration which will
include the default bindings for the environment and all default bindings from all modules on
the modulepath.

That is; if all bindings from modules are consistent the users does not have to do anything.

That sounds fine. They don't have any bindings in their modules yet, and they are not using hiera.
They do however need a `dev` environment. So that is the first task they tackle.

Akunna Adds a "dev" Environment
-----------------------------
Akunna learns that this is done in a new file called `environments.pp` and she writes the following (specifying all options as
she wants to see them and think about their use). She marks the one that are restating the defaults:

    site {
      # Environment defaults
      Environment {
        envdir      => '$confdir/master'
        modulepath  => 'modules/'     # default
        manifestdir => 'manifests/',  # default 
        manifest    => 'site.pp',     # default
        templatedir => 'templates/',  # default
        bindingsdir => 'bindings'     # default
      }

    }
    
    $environment = 'production'

Akunna understands from the documentation that an entry for "production" is added by default so she leaves that out.

Akunna runs this with the existing classic `site.pp` and sees that it works.


Akunna Wants Authoritative Setting of Environment
-----------------------------------------------
Akunna wants authoritative setting of environment (like in the ENC) and not use an environment that is specified on the
agent. Akunna could naturally use the ENC - if so, nothing needs to be done - the ENC will then provide the authoritative
mapping from a node request to environment. But Akunna wants to try out the new features so they leave out their existing ENC,
well aware that she could have used that to save themselves some work. She is mainly interested in being able to later set up a
review process and she likes the fact that everything is written in .pp manifests and kept in a git repo.

Akunna is happy to learn that the full set of facts is available when deciding which environment the request should use. This was
something that made things difficult in their existing ENC, and they had to cheat and read facts stored as a side effect
of processing incoming requests.

To start off, Akunna keeps it simple. All machines except those that are named after the seven dwarfs in German
should be in production, the rest are "dev" machines.
Since "production" is default this is done with a simple `if` expression in `environments.pp` - she also adds
the definition of the production and dev environments.

      # Environment defaults
      Environment {
        envdir      => '$confdir/master'
        modulepath  => 'modules/'     # default
        manifestdir => 'manifests/',  # default 
        manifest    => 'site.pp',     # default
        templatedir => 'templates/',  # default
        bindingsdir => 'bindings'     # default
      }


      # A static environment
      environment { 'production':
        # All default
      }
      # Additional static environment
      environment { 'dev':
        envdir => "$confdir/dev"
      }

    if $hostname in [huckelpack, naseweis, packe, pick, puck, purselbaum, rumbelbold] {
      $environment = 'dev'
    }
    else {
      $environment = 'production'
    }

Akunna likes that she can make this as simple or complex as required (using `if`, `case`, calling functions etc.).


Akunna Wants to assign Common Classes and Parameters
--------------------------------------------------

Two things are assumed at this point:
* Akunna is happy with the default categories (`node`, `environment`, and `common`) available for binding.
* Akunna does not need to do any advanced layering of bindings; site level bindings simply overrides those specified in modules.

Akunna decides to come back to that when she understands what she wants to do.

> Note: Assumes improvement to spec as above (basedir)

In the given configuration, the bindings for the dev environment are in `$confidr/dev/bindings`. Akunna edits the
file called `default.pp`. She wants to use the same set of classes and the same set of parameters (i.e. top
scope variables) on all nodes.

    bindings default {
      # classes common to all
      include [common, puppet, ntp, aptsetup]
      
      # top scope variables common to all
      bind variables to {
        servers     => ['0.pool.ntp.org', 'ntp.example.com'],
        mail_server => 'mail.example.com',
        iburst      => true
      }
      
      # bind parameters for classes
      #
      bind parameters ntp to { ntpserver => '0.pool.ntp.org' }
      bind parameters aptsetup to {
        additional_apt_repos => [
          'deb localrepo.example.com/ubuntu lucid production',
          'deb localrepo.example.com/ubunty lucid vendor'
          ]
      }
    }

> Note: bind variables not in ARM text yet, may change...

Vatsan points out that in the "dev" environment they want to include the development apt repo, and that they do
not want autoupdate for ntp. Akunna adds the following at the end in the `bindings default { }`:

    when environment dev {
      bind parameters aptsetup to {
        additional_apt_repos => [
          'deb localrepo.example.com/ubuntu lucid dev',
          'deb localrepo.example.com/ubuntu lucid production',
          'deb localrepo.example.com/ubunty lucid vendor'
          ]
      }
      bind parameters ntp to { autoupdate => false }
    }

Akunna Crafts Each Node by Hand
-----------------------------
With the defaults specified, Akunna now wants to configure the nodes used for development. She adds this to
`default.pp`; on node pick, Vatan points out, they want to try a new implementation of ntp, but use the parameters
as they are defined for ntp.

    when node huckelpack {
      include meat
    }
    when node naseweis {
      include potatoes
    }
    when node packe {
      include [meat, potatoes]
    }
    when node pick {
      include mymodule::hack
      
      # get rid of the ntp included by default
      exclude ntp
      
      # include the new (to be tested)
      include myntp::ntp
      
      # reuse the parameter settings for ntp
      bind parameters ntp2 to ntp 
    }
    
    # These three are the same
    when node puck or purselbaum or rumbelbold {
      include ourapp::baseapp
    }

They have 140 more nodes to configure to also do this for all the production machines, and Akunna realizes that it would
be better to use a new category for role. She realizes that she would otherwise be repeating very similar `include` and
`bind parameters` statements.

Akunna adds a role categorization
---------------------------------
Instead of making configurations node by node, Akunna has decided that there are really four distinct roles:
backend, frontend, app, and combined (running both frontend and backend).

Akunna edits the `site.pp` to add a "role" categorization. She places the following snippet in `site.pp`
where the role is determined by a pattern applied to the hostname (for production), and the name spelled out
for the development machines. She already learned that in Puppet 4 it is possible to use a `case` as an expression
producing a value, and she decides to use that to see what it looks like. Vatsan points out that they use
pattern based naming in production and that they could do both the dev servers and the production at the same time.

Akunna writes:

    site {
      categories {
        role => case $hostname {
        
          hucklepuck, /.*-be[0-9]+.*/ : { backend }
          
          naseweis,   /.*-fe[0-9]+.*/ : { frontend }
          
          packe                       : { combined }
          
          puck, purselbaum,
          rumbelbold,
          /*.-app[0-9]+.*/            : { appserver }
      }
    }
    
Akunna and Vatsan realizes that this is a mix of production and dev servers, but they conclude that
typically dev uses the seven dwarfs the same way all the time. They understand that the site.pp can later be split up
into two; one for production and one for dev, but for now they are happy that that it is clear which dwarf is playing
the same role as a host acting in a role based on their host naming scheme.

Vatsan wants to also include the other categories (`node` and `environment`), but Akunna points out that
they are there by default with sane precedence; the just added `role` category is above `environment` and below `node`.
Vatsan goes off to read for himself and finds information that they can later use secure facts to assign roles
safely - he points out to Akunna that they can later use this to give roles directly to the dev machines. They decide to explore
that at some other time.

Akunna now modifies the configuration and replace the use of hostnames with the new `role` category:

    when role backend {
      include meat
    }
    when role frontend {
      include potatoes
    }
    when role combined {
      include [meat, potatoes]
    }
    when role appserver {
      include ourapp::baseapp
    }
    
    when node pick {
      include mymodule::hack
      
      # get rid of the ntp included by default
      exclude ntp
      
      # include the new (to be tested)
      include myntp::ntp
      
      # reuse the parameter settings for ntp
      bind parameters ntp2 to ntp 
    }

Akunna thinks the configuration for host `pick` sticks out - this is really an experiment devs are running
and this configuration should not be merged into production. Akunna decides to come back to this (in the next example).

Akunna Breaks out Part of the Configuration
-------------------------------------------
The node `pick` is used for experiments, and it should not really be configured in `default.pp` which is
otherwise clean and defined in terms of roles. Akunna decides to move the configuration of `pick` to a bindings file
called experiment.pp. She realizes that if she wraps the entire configuration in a `when environment dev { }` she
will have an easy job later when reviewing changes as this ensures that the bindings are only in effect in the dev
environemtnt.

Akunna writes:

    bindings experiment {
      when environment dev {
        when node pick {
          include mymodule::hack
          
          # get rid of the ntp included by default
          exclude ntp
          
          # include the new (to be tested)
          include myntp::ntp
          
          # reuse the parameter settings for ntp
          bind parameters ntp2 to ntp 
        }
      }
    }



Now, she does not have to worry that someone makes statements about nodes and roles that are not in the dev
environment. She can easily check this when doing a review. The dev environment does after all run in the same
master as production.

The experiment bindings are not included by default in site.pp. Akunna decides to have two different site.pp, one for
production, and one for dev. The production site.pp can use the defaults, so nothing is needed there. Akunna starts to
wtite things into the `$confdir/dev/manifests/site.pp`, but realises that is is an easily made mistake to merge `site.pp` from
dev into the master which is used for production, so she renames it to `devsite.pp`. As soon as she starts writing she
realizes that if the `envrionments.pp` refers to the `devsite.pp` directly, then basically the dev environment can perform
all sorts of bindings that Akuna does not quite trust others to tamper with.

Akuna and Vatsan discusses how they want to solve this, and they come up with an idea. They really do want to reserve the
right to define the layering of bindings for everything that runs on the master, so delegation to site.pp files that are not
strictly managed is a bad idea - they want to reserve the right to be able to override any bindings at the stie level (above
all environments, and the way to do this is to control which site.pp to use.

They decide to use three different site.pp files; one for production, one for dev and one for dynamic environments.
They do not yet know if the dynamic environments will be different, but they think they need to do different things
there later as the dynamic environments are mostly used to run various tests.

In `$confdir/manifests/site.pp` she writes this (this is how "production" works)

    site {
      bindings => [
        layer { 'site'    : include => 'confdir:/default' }
        layer { 'env'     : include => 'env:/default' }
        layer { 'modules' : include => 'module:/*::default' }
        ]
    }

i.e., the site level confdir overrides anything that comes from the environment, which in turn overrides anything
that comes from modules.

For development it is of value to override things for various reasons (testing, while refactoring, doing experiments)
and these bindings are not known in advance. If experiments are successful, the changes should be made in the defaults
bindings and later merged into the master.

Akunna now writes  `$confdir/manifests/devsite.pp`:

    site {
      bindings => [
        layer { 'site'        : include => 'confdir:/default' }
        layer { 'devpatches'  : include => 'env:/patches::*' }
        layer { 'env'         : include => 'env:/default' }
        layer { 'modules'     : include => 'module:/*::default' }
        ]
    }

Now developers can drop bindings into `$confdir/dev/bindings/patches` and they will be automatically picked up.
The bindings under `patches` will override those in the dev environments default, and will override all modules.

Akunna updates the experiments binding and moves it to `$confdir/dev/bindings/patches/experiment.pp`, and edits the file
to start with:

    bindings patches::experiment {
      # the rest is the same

Vatsan points out that they need to address the module path since some modules should always be included and
configured the approved way, so they need to layer the bindings for thos modules as well. They decide to
come back to that later.

> ISSUE: The `env:` scheme is not described yet in the ARM text.  
> DISCUSS: Are users more comfortable with file paths than symbolic names / URI's even if that means more
> using interpolated strings with paths (which is less robust/secure since the environments.pp defines the paths
> and can be made more secure.

Vatsan, points out that they must update the `environments.pp` as well to refer to `site.pp` and `devsite.pp` resectively
and Vatsan edits those parts to read:

      environment { 'production':
        envdir   => "$confdir/master"
        manifest => "$confdir/manifests/site.pp"
      }
      environment { 'dev':
        envdir   => "$confdir/dev"
        manifest => "$confdir/manifests/devsite.pp"
      }


A basic configuration
---------------------

I want to be able to understand exactly what I have to change from
"today's Puppet" to "tomorrow's Puppet" to meet the needs of the
little infrastructure at a past job.  Very simple: two environments,
dev and prod.  A bunch of modules, written by us, and around a hundred
machines, hand-classified via node declarations in Puppet.  The
classic "base" parameterised class that sets up all the core services,
and then per-service modules.  Automatically configured backups and
monitoring of all systems.

Nothing very fancy.  I really just need to understand enough to be
able to translate my current modules into the new world.  So, please,
concretely call out every new syntax and meaning that I have to care
about as a user.

Push anything that isn't part of that concrete syntax, or explanation
of how the syntax works, or the rules of evaluation change, to a
separate document.
