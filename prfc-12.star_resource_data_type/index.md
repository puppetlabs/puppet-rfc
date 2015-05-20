ARM-12: A Resource Data Format, and Associated Tooling for Manipulating Data
====================


Summary
-------

We should add a new data type to the Puppet DSL based on the resource declaration syntax, so that we can describe a resource without automatically declaring it. We should then add some tooling around these resource descriptions: certainly including declaration and filtering, and possibly including iteration.

This touches ARM2, but does not directly depend on it. It might block it, depending on the direction we take.


Goals
-----

* Enable better data manipulation in Puppet.
* Fulfill the original promise of virtual resources, with better control and more freedom.
* Enable Puppet-like declarative iteration, instead of Ruby-like mystery-meat iteration.
* Enable sophisticated/flexible interaction with PuppetDB, yea even unto agent-side queries/subscriptions. (Talk to Nick Lewis about this.)


Non-Goals
---------

Nope, this fixes everything. :)

Success Metrics
---------------

<#
If the success of this work can be gauged by specific numerical
metrics and associated goals then describe them here.
#>

Motivation
----------

Puppet's DSL is built around the RESOURCE DECLARATION.

    package {'apache2':
      ensure => latest,
    }

We presuppose here that resource declarations were a good idea in the first place. They're pithy, easy to write, easy to visually grok, good at representing the thing they represent.

But they're statements, not values -- they **always** have the side effect of adding the resource they describe to the catalog. Virtual and exported resources are statements, too: they add resources to the global catalog namespace and expect to have a clean spot there reserved for them, even if they don't end up getting realized and serialized to the agent.

The Puppet DSL lacks a way to **describe** a resource without having the side effect of **declaring** it. We should have one.

ITEM ONE: The iteration work (ARM2) has gotten divided reactions. Some think bringing large parts of Ruby (loops) into the DSL is good. Others (me) think we should be more declarative: disallow loops, focus on mapping (AKA collect), disallow side effects inside map blocks (no resource declarations, rvalue functions only). Problem is, mapping doesn't really get you much if you can't use it to construct resources.

ITEM TWO: Well, you actually CAN construct resources in a purely data-oriented way by using the hash format accepted by the `create_resources` function! But if we admit that resource declarations were a good idea in any respect, we also have to admit that the `create_resources` format sucks. It's fine as a data interchange format (nearly anything would be), but it's a terrible user interface, lacking everything that makes resource declarations good. As Andy points out, this is what DSLs are for: richly expressing unusual data formats (which may enter the DSL via some interchange format built around common primitives like hashes, but which we can deal with more usefully once they come in the door).


Description
-----------

### New Construct: The Resource Description (star resource)

We introduce a new language construct: the star-resource. This is based on the precedent of the at-resource (`@file{...}`) used by virtual resources.

    $myresource = *file {'/tmp/foo':
                    ensure  => file,
                    content => "blah blah blah",
                  }

Let's call these "resource descriptions," to distinguish them from existing constructs. ("Resource declarations" are the same thing without the star; "resource references" are like `File['/tmp/foo']`. I've also heard "resource pointer" as a reasonable answer to "what is that," but it might be a cognitive mismatch with the metaphors other languages use, in which case it might be worth avoiding. I'm equally not married to the star.)

A resource description is **NOT A STATEMENT.** It's a pure value, and therefore can't just stand on its own: it must be an argument to a function, or a value to a variable, or something. It has no side-effect: nothing gets added to the catalog when you utter one. No global namespace gets polluted: right after the above, you could also say `$myotherresource = *file {'/tmp/foo': content => "something else entirely"}`, and it wouldn't conflict at all. It's **just data.**

### New Tooling: Declare Function

So what do you do with resources? You declare them. Eventually.

    declare($myresource)

To be as clear as possible: The above statement, combined with the variable assignment further above, end up having the same effect as:

    file {'/tmp/foo':
      ensure  => file,
      content => "blah blah blah",
    }

It just gives you the option of splitting up the description step and the declaration step.

### New Tooling: Declare Function: Override Control

Since we're now in control of the declaration step, which we never were before, we can start asserting control over duplicate resources.

    declare($myresources, override => replace) # Current resources win no matter what.
    declare($myresources, override => merge) # Merge attributes, with current argument winning in the event of a conflict.
    declare($myresources, override => none) # default, only declares if it wouldn't conflict with a resource that's already in the catalog

Yeah, this has parse-order dependent problems. We need better late-binding facilities. Hopefully the utility is obvious, though.

### New Tooling: Select Function

Now that we have resource data objects, we can put them in arrays, and then operate on arrays of resource descriptions to exclude those we don't want.

    $myresources = [
                     *file {'/tmp/foo':
                       ensure  => file,
                       content => 'newerstuff',
                     },
                     *package {'foopackage':
                       ensure => latest,
                       tag    => [apache, wordpress],
                     }
                   ]

    declare( select( select($myresources, type == file), tag =~ /apache/) )

Note that this example would also require us to:

* Turn regexen into a normal data type instead of whatever they are now.
* Create an expression data type.

But I think we're already on track to allow that sort of thing with the new grammar prototype.

### Revision of Proposed Tooling: No Loops, Just Collect

Ruby loops with mystery-meat side-effect-rich code are anti-Puppetish. We should focus on Puppetish declarative ideas like data mapping (AKA collect). Now that we can treat resources like data, we actually have the option to do that.


    $mcosettings = {
      "connector => activemq",
      "direct_addressing => 1",
      "plugin.activemq.pool.size => 1",
      "plugin.activemq.pool.1.host => middleware.example.com",
      "plugin.activemq.pool.1.port => 61614",
      "plugin.activemq.pool.1.user => mcollective",
      "plugin.activemq.pool.1.password => sanotehusasoethu",
      "plugin.activemq.pool.1.ssl => 1",
      "plugin.activemq.pool.1.ssl.fallback => 1"
    }


    $myresources = $mcosettings.collect |$setting, $value| {
      *mcollective::setting {$setting:
        value => $value,
      }
    }

    declare $myresources

Basic collect syntax: set all the variables you want, do whatever, and the last value in the block is what gets added to the final data structure. If you want to do multiple resources per input set, you can make an array of resource descriptions and flatten the final array before handing it off to `declare`.

We should limit the syntax available inside collect blocks. Rvalue functions only, no include or create_resources or anything like that. It's possible to break it by creating an rvalue with side effects, of course, but we can culturally condemn that, and it's probably not going to be very common anyway.

### Rehabilitation of create_resources

Call it `make_resources` -- we can turn an incoming hash-of-hashes structure into an array of resource descriptions, which can then be handed to the normal `select` and `declare` functions.

Instead of "hand resource data to Puppet and force it directly into the catalog," the semantics become "hand resource data to Puppet," and Puppet can then do whatever with it.

### Virtual Resource-Like Behavior Without Polluting Global Namespace

You've already noticed that the combination of `select` and `declare` lets you emulate virtual resources, as long as you've got a variable containing some number of resource descriptions. I argue that this is better than the existing implementation. Sometimes you may want to reconcile between several possible versions of a resource, and being able to specify override logic when you combine and declare them will get you the benefit of duplicate resource declarations (I forget the issue number for this request) without having to reconcile things in a global namespace -- any reconciliation must be manual and deliberate.

My suspicion is that virtual resources are mostly used in intra-organizational code that doesn't get shared, so the "publish to global namespace, then search" capability isn't actually being used; most people would probably be happy to have a few fully-qualified variables full of virtual users or whatever, and select/declare several times as needed.

### New Tooling: Export Function

    export($myresources)

You get the idea.


Testing and Evaluation
----------------------

dunno yet.

Alternatives and Recommendation
-------------------------------

Dunno yet.

Risks and Assumptions
---------------------

Dunno yet.

Dependencies
------------

This ARM relates to ARM2 (iteration), but doesn't depend on it. It may block it, depending on what we decide.

Impact
------

Dunno yet.
