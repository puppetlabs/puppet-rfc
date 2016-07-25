## PRFC-24: Facter Config File

Summary
Facter is a Puppet-related project, which is responsible for gathering inventory data ('facts') about the system it's run on. Puppet uses Facter's facts to learn about the details of each system so it can make intelligent decisions about, for example, which package managers are supported. Each fact is also available to module authors for decision making and variable expansion inside their modules.

Facter has historically not had any persistent run-time configuration; you could influence some elements of its operation with environment variables and command-line options, but these avenues are necessarily pretty limited. This matched its usage as a simple, embeddable library, but has restricted a number of useful capabilities which require a config file.

This PRFC aims to describe the use cases and user-facing surface area for a proposed 'facter.conf'.

### Definitions

**Facts** - any bit of data collected by facter, encompassing all of the below

**Structured Facts** - facts represented as complex data structures: arrays, key-value maps, nested combinations thereof

**Custom Facts** - Facts determined by code that uses Facter's API

**External Facts** - Facts which are loaded from data files, such as a json or yaml file, or by running an external program, rather than using Facter's API.

**Resolution** - The act of running a bit of code on a system in order to establish the value of a fact.

### Problems and People

Every Puppet user is also a Facter user, whether they realize it or not. Facter is not only ubiquitous in every off-the-shelf Puppet installation, it's also the easiest part of Puppet to customize and extend.

From the mailing list thread and excavation of past tickets about this, we've ascertained the following use cases for such a config file:

* Adding the ability to set a "TTL" (time to live) for particular fact values, so that facts which are computationally expensive to resolve do not run every time facter runs. (Redmine #1932)

* Blocking facts, so that some facts which are irrelevant for a system or cause errors when run can be completely eliminated from the resolution. (FACT-718, FACT-449)

* Global configuration of facter's behavior

    * External fact location(s)

    * Custom fact location (to eliminate the need to run 'facter -p')

    * Timeouts for fact resolution (not the lifetime of fact values, but how long to wait for an individual fact to resolve)

* Providing Configuration for individual facts as they run. For example, the fact which resolves EC2 metadata hits a hardcoded IP address that might need to change for different cloud providers

### Non-Goals

There are a number of feature requests that have come up aside from the ones listed in the goals section. Not everything is going to be in the first release, and some things won't ever make it in.  It is perhaps worth stating that, unlike puppet, it will not expose every config file option as a command-line setting. (Existing command line options are also valid settings in the 'global' section, however; see below.)

### Proposal
#### Properties

We're proposing a facter.conf which has the following properties:

* It's written in HOCON so it's readable/writable by the hocon toolchain - (Here's the [HOCON spec](https://github.com/typesafehub/config/blob/master/HOCON.md) and [HOCON for Puppet docs](https://docs.puppet.com/pe/latest/config_hocon.html))

* It supports `conf.d/` style snippets so the entire config doesn't have to be in one monolithic file

* Facter defaults to reading it from `/etc/puppetlabs/facter/facter.conf` and `C:\ProgramData\PuppetLabs\facter\etc\facter.conf` but this can be overridden on the command line.

* Any specification of a fact supports structured facts using `key.subkey.tertiarykey` notation as you can from the command line; configuration items with incomplete specifications for keys (i.e. only specifying `networking.interfaces` when `networking.interfaces.eth0` and `networking.interfaces.eth1` exist) will cause the configuration value to be applied to all subkeys.


#### Configuration Keys

The configuration namespace lives inside a top-level key called 'facter'. Under it are a few second-level keys supporting the major pieces of functionality (it should ignore unknown keys for future/backwards compatibility). In order of implementation priority, these are:

* `global` - configure facter's behavior, like paths to facts

    * `external_fact_dir`

    * `custom_fact_dir`

    * `fact_timeout`

* `blocklist` - list of facts which ought NOT be resolved

* `ttls` - configure time to live values for facts

* `timeouts` - max seconds to wait to resolve individual facts

* `configurations` - configuration for individual facts

So an example config file would look like:

    facter : {
      "global": {
        "fact_timeout": 10,
        "show-legacy": false
      },
      "ttls": {
        # give the AWS metadata facts more time
        "aws": 30
      },
    }

#### TTL Implementation

The per-fact TTL require back-end work to implement and is worthy of deeper discussion. The idea here is that the cache directory should look and act like an external facts directory, whose contents happen to be managed by Facter itself.  Facter will need to serialize its output with a timestamp for each fact, so it knows when it was last run and therefore whether the TTL will expire. A TTL expiring does not mean Facter will automatically detect that and re-run immediately, but rather that the fact will be resolved afresh upon facter's next invocation.

Some design details:

* Cache location on filesystem - `/opt/puppetlabs/facter/cache/cached_facts`, `C:\ProgramData\PuppetLabs\facter\cache\cached_facts`

* Serialization file format will be a json structure of the fact data, one file per fact specified in the config file (facts without explicit TTLs be resolved every time and not serialized), named as the fact name is specified in the configuration. In effect, the cache directory becomes an additional external facts directory, it's just that the files therein are created and managed by facter itself; for example:

```
    # /opt/puppetlabs/var/cache/facter/cached_facts/is_virtual.json
    {'is_virtual': true}
```

* The timestamp of the file indicates the last time it was run, so no additional metadata inside the file itself is necessary.

### Prior Art
There are a couple of dimensions to consider. For best practice on config files in general, we've considered puppet.conf's evolution over the years and are trying to avoid some of the problems that have emerged with it, like having 'eval'-able code available as part of every setting, automatically exposing every setting as a command-line option, and having different sections in the same file which are read and overridden in a particular way.

Puppet's ecosystem has evolved to standardize on HOCON for config files, as a human-readable, comment-friendly data source with good tooling around it. We also advocate using the 'conf.d' snippet pattern to allow composable bits of configuration to be added from and owned by separate sources (such as packages vs puppet-managed code).

The second dimension is configuration for system inventory data gathering tools like Facter: what are other tools doing? What requirements and "You Aint Gonna Need It" conclusions can we draw from the configurability that comparable tools provide?

[Ohai](https://docs.chef.io/ohai.html) performs the same role in the Chef ecosystem as Facter does for Puppet. It has two sets of configuration, but no separate config file. You can set the equivalent configuration items as our proposed 'global' section in client.rb (despite the ruby extension, this is essentially a config file for the chef client), and the values exposed seem similar, [see the docs here](https://docs.chef.io/ohai.html#ohai-settings-in-client-rb) for more details. Second, instead of a blocklist for individual facts, ohai presents a [whitelist](https://docs.chef.io/ohai.html#whitelist-attributes) for automatic attributes ("structured facts" in Facter's parlance) and anything not in the whitelist will not be resolved.

Osquery also does a similar job to Facter, though it's a stand-alone tool and provides a distributed interface to all systems under management. In addition to extensive command-line configuration, it [uses a configuration file](https://osquery.readthedocs.io/en/stable/deployment/configuration/) for its osqueryd daemon which implements TTLs for particular bits of data:

    {
      "options": {        "host_identifier": "hostname",        "schedule_splay_percent": 10      },      "schedule": {        "macosx_kextstat": {          "query": "SELECT * FROM kernel_extensions;",          "interval": 10        },        "foobar": {          "query": "SELECT foo, bar, pid FROM foobar_table;",          "interval": 600        }      }    }

Each of those keys (`macosx_kextstat` and `foobar` above) represents a data structure similar to Facter's top-level nested facts.

### Alternatives

We could:

* Do nothing; this perpetuates the status quo but it does seem like several desirable features are blocked from implementation due to the lack of run-time configurability

* Support configuration but do not use a file; options here are possible but pretty undesirable:

    * Environment variables: there is precedent in facter for supporting configuration from environment variables like `FACTER_LIB` and `FACTER_foo` (which will set a fact `foo` inside of facter). But these are pretty limited, and expanding them to support TTLs, for example, would simply off-load the management of the variables into another file that's a shell script sourced before running facter.

    * Support command-line arguments but no config file: this excludes a primary use-case which is running facter as a library from inside puppet, where no command-line is given.

* Support a configuration file but do not use HOCON: this is counter to Puppet's roadmap

### Additional Concerns
How will this work impact other parts of the platform, the product,and the contributors working on them?- Other Puppet components: The invocation for facter on the command line and from Puppet should remain the same.- Compatibility: Facter with no config file should continue to run as it does today, with sane defaults built in.- Security: This shouldn't introduce any additional security concerns.- Performance/scalability: This should improve performance by allowing users to remove expensive facts they don't need with the blacklist and setting long TTLs for expensive-to-resolve facts. Scalability is not applicable since facter runs on a single system only.- User experience: The main point of interaction is with the config file itself, so as long as the RFC process incorporates user feedback on the naming of various keys and the location/naming of the files, it should not require extensive UX design.- I18n/L10n: n/a- Accessibility: n/a- Portability: It should run anywhere we can build facter and the cpp-hocon library.- Packaging/installation: We should include an example config, installed inline, to show the possible top-level keys, exclude some commonly-problematic facts (partitions?), and give users a starting point for configuration.- Documentation: It will require documenting the available configuration items.- Spin-offs/Future work: As we move towards the asynchronous architecture described in PRFC-11, the execution model will be running facter periodically, uploading its facts to PuppetDB, independent of requesting a catalog. Having TTLs for expensive facts and some configuration will be helpful in reducing the execution cost of having facter run more frequently.- Other:  ...

- Serialization formats and representation of rich data - Puppet will likely base its serialization on Pcore in order to handle rich data in catalogs (binary, sensitive, encrypted, etc.). While probably not an immediate concern for Facter and this PRFC, it is of value to not invent yet another rich data type serialization scheme specific to Facter.

- Upgradeability and Versioning - the config file should probably have a version in it to ensure that an upgrade of Facter that includes a change of the config file format knows if it is reading with old or new semantics. (This is very hard to change later). The same applies to the cache information (it can be wiped if version changes, but Facter needs to know which version wrote the information.). Depending on how serialization of individual facts is done, the format of individual facts may change between versions of the fact adding additional complexity. It would be best if all facts are generically serialized without any detailed individual control over what is written to the cache files.
