PRFC-23 XPP - Exchangeable Puppet Files
====

Summary
---

This RFC proposes the introduction of a new file to be included in
Puppet modules alongside the existing ".pp" manifests: ".xpp" files,
for "e**X**changeable pu**PP**et".

The XPP file format contains pre-parsed and pre-validated
Puppet Language source in a format that can be quickly loaded at runtime,
eliminating the need to repeatedly parse and validate the source text.

Definitions
---

* **Lexing** - the process of turning puppet source text into tokens.

* **Parsing** - the process of turning tokens into an Abstract Syntax Tree
  (AST) that conforms to a very generalized form of puppet syntax as
  directed by the puppet language grammar.

* **(Puppet Language) Grammar** - defines how tokens are transformed into
  an AST. The grammar is the core of the parser.

* **Validation** - static validation (all validations that can be
  performed without access to a runtime context) of an AST that
  checks for semantic errors. Does not perform validation across
  multiple source files (for example, validation does not include
  checking resolution of names to  definitions).

* **Parse Result** - an AST plus any warnings or errors produced by
  parsing and validation. In cases where an AST cannot be produced,
  the parse result consists solely of errors/warnings.

Problems and People
---

The problems XPP helps solve include:

* Performance - we currently parse and validate unmodified puppet
  source code over and over again.

* Cross-language support - We are working on a C++ implementation of
  the Puppet Parser / Validator, and it needs to provide the parsed
  and validated result to the Puppet runtime in Ruby.

We currently need to parse and validate the puppet source for each catalog
compilation. While we can use environment caching to hold the parsed result
in memory, this is not always ideal. Parsing and validation comprise
a substantial part of the effort to compile a catalog.

We are working on a C++ implementation of Puppet. In order to get the benefits
of this as early as possible we want to use the C++ Parser and Validator
as it is several orders of magnitude faster than the implementation in Ruby.
Supporting the full compiler, with all that entails (custom Ruby functions,
types and providers), is a major undertaking and will take quite some time
until it is finished. The problem now becomes how to interoperate between
a C++ implementation of the parser and the runtime in Ruby.
Exploring serialization of the Puppet AST has shown that it is possible
to read AST faster than a full lexing/parsing/validation of Puppet source text.

**Requirements:**

* The use of XPP should be a runtime concern. No preparation should be
  required by module authors, publishers, or consumers.
* The use of XPP should be transparent to how the puppet source was
  consumed/installed/made-available to a puppet runtime environment.
* Until XPP has proven to provide value, it requires an opt in via a
  the boolean setting `xpp` or on the command line `--no-xpp`, `--xpp`
* XPP files *must* not be written into modules directories.
* XPP files *may* be written into a subdirectory of an environment.
* When XPP is fully supported (a non experimental feature), the
  production of XPP *should* either always be performed by the server
  when deploying new code, or be an opt-in.
* It is initially *acceptable* to run a manual command to ensure
  up to date XPP files have been produced.
* At runtime:
  * an error in loading an XPP *must* log the problem and
    continue by using the puppet source file.

**Additional requirements:**

Efficient serialization is a problem in general. Obviously XPP is a kind
of serialization of data. Unfortunately, the way serialization is normally
handled in Puppet falls quite short of being an optimal, fast loading format
for XPP (due to verbosity and lack of data types). It is desirable when
designing this serialization format that it is not a one-off solution specific
to XPP, as the same traits are of great benefit in many additional use-cases.


### Non-Goals

This PRFC does not specify the use of the XPP defined content in a REST scenario.

Defining a byte-code oriented format for Puppet is not a goal of this PRFC.
The intent is to use an efficient serialization of the AST as that requires
the least amount of work, and the most reuse of the current implementation.

It is not a goal that XPP is human readable without the aid of a tool that
re-serializes the content in a human readable format.

Proposal
===

XPP is a file format that allows the parsing and validation to take place
at another time and place than when a catalog is compiled.

This PRFC proposes that a feature switch is used for puppet that allows it
to use an .xpp file instead of a .pp file if an `.xpp` file exists.
Puppet will then skip parsing and validating the .pp file, and will
instead deserialize the content of the `.xpp` to get a parse result (which
may include warnings and errors).

The intent is to produce the .xpp file off-line using the C++ based parser.
At runtime an `.xpp` file it is simply there and gets used, or it is not,
in which case the runtime works exactly the same way as before.
If an `.xpp` is used, after deserialization the rest of the compilation
is exactly the same (evaluation and resource handling is done in Ruby as
if the AST came from the Ruby parser).

The expected workflow for users that opt into using XPP is to run one
command (the C++ based parser) that parses and validates all `.pp` files
in an environment once changes to `.pp` code has been made.

There are three main scenarios:

* User is doing everything manually. If opting in to XPP, user must
  manually run the native parser to produce the corresponding XPP.
  (Or opt out from XPP while developing). Any automation would be
  artisan.

* User is using r10k. A hook can be used to automatically run the
  native parser so that a code deploy gets an up to date set of XPP
  files.

* User is using Puppetserver with atomic deploy. The server will then
  take care of automatically producing the XPP by calling out to the
  native parser.

There are several other use cases for the native parser that are beyond
the scope of this PRFC such as using it during development to catch syntax
errors, perform validation etc. All without producing XPP.

API's, command line switches, network protocols
---

The behavior is controlled by a feature switch in Puppet that is turned on
off with a puppet setting:

```
--xpp --no-xpp
```

The parser implemented in C++ defines its own set of command line flags
to support different use cases; experimenting, validating, or for producing
XPP files that can be used at runtime. This PRFC does not define those flags.
The intent is that the output of the C++ based parser is compatible with the
expected location of .xpp files in puppet.

The initial release will also contain support for experimentation and troubleshooting,
for example writing the output in a debug format, parsing and validating single files
"manually", parsing and validating a single module, etc. There will be a tool for
dumping information about the content of an XPP file.

Interfaces modified or extended
---

Internally, the parser, when told to parse a path that ends with `.pp` and when
the `--xpp` flag is true, will look for and use a an .xpp file in a predetermined
location. There are no API changed in Ruby.

There will be one directory named `.xpp` in the environment root at runtime that
puppet will use to resolve `.xpp` files.

For all things loaded via the 4.x type loaders (4.x functions in puppet, and
data types), the `--xpp`flag will simply enable loaders that search the corresponding
module location under the `.xpp` directory.

For logic that directly asks the parser to parse a .pp file will, when the `--xpp`
flag is in effect first attempt to resolve that against an `.xpp` before proceeding
with the `.pp` file.

There will be no check at runtime that asserts that an `.xpp` is newer or equal
in modification time to the `.pp` it is matching as this will be too expensive.
It is instead expected that those that opt in to using `--xpp` will process all
`.pp` files in a r10k hook upon deploy of new code, or use other similar automation.
In manual workflows, if puppet is executed with `--xpp` on, a user will simply
have to issue one native parser command to process all files (or just the
one modified file).

### The `.xpp` directory

Under the `.xpp` directory there will be one directory named `modules`, and in this
directory there will be one directory per named module on the module path plus the
reserved module named "environment" which is for the content of the environment
itself; where it is possible to place functions and types, and where the `site.pp`,
or a recursive "manifest" directory is stored.

In the event that a user has configured a module path where the same module name
appears more than once, only the first encountered is processed for `.xpp`, the
duplicate module instance would continue to work as today and not benefit from
`.xpp` loading. Configuring a module path with multiple entries using different
version of the same module is considered to have unspecified behavior anyway,
and such configurations should not be used.

This scheme supports modules that are found anywhere on the system with a minimum of mapping.

For services that replicate the environment/module structure it is expected that
the `.xpp` file structure is also replicated. This to avoid having to redo
the production of the `.xpp` files on all servers replicated too.

It is expected that users do not check in the `.xpp` file structure in their
source control repos, and that all `.xpp` is produced when new code is deployed.

When running puppet with a `manifest` setting that is outside of
the environment directory, no XPP support is available for the `.pp` files thus
referenced. This is seen as arbitrary input akin to using `puppet apply` with `-e` input.

### Search order

All searches are directed by the order of modules are processed on an environment's
module path. The 4.x loaders use this to configure a graph of loaders.
Thus a search will take place in the expected order.

When the autoloader loads `.pp` files it will also search in module order and it
will arrive at loadable files based on the `.pp` file existence in certain locations.
Only if a `.pp` file is found will the corresponding `.xpp` file be considered.
Thus, the search order is maintained, and if a module path (despite causing
undefined behavior) has the same module in multiple locations, a found `.pp`
for subsequent copies/variants of the same module would thus not find
a `.xpp` file. (And the order is preserved).

### File location recorded in `.xpp`

Parsed `.pp` files retain the source file path information and this is the path
that is shown in error messages. With use of `.xpp` the `.xpp` files may have been
produced from files that are in some staging area before they are deployed into
their "in production" location. In such cases, the `.xpp` must include the
path effective at runtime. Since this filename is only used in errors messages,
it is acceptable if all references to files within the `environment/modules` directory
are relative, and all others are absolute.

Diagnostics of XPP (and how to back out)
---

The intent is to use the most efficient format which means that the content
is binary (msgpack) with additional semantics applied. This is naturally
impossible for a human to inspect by just opening the file.

A command line tool will be offered that outputs the content of the `.xpp` file
in human readable form for information and debugging purposes.

Backing out of using XPP in puppet is simply the feature switch `--no-xpp`
(which is also the default). When this binary setting is off, no XPP files will be loaded.


XPP Content / Grammar
---

The XPP format is an instance of a Pcore Serialization. The general format is
defined by the following grammar:

```
PcoreSerialization
  : Envelope Payload
  ;

Envelope
  : MimeType BlankLine
  ;

MimeType
  : PcorePart '+' BaseEncoding (';' Parameter '=' Value)*
  ;
 
PcorePart
  : 'application/vnd.puppet.pcore'
  ;

# The main machine to machine format is msgpack. The json format
# is intended for human use and alternative representations where
# msgpack is unsuitable.
# Other formats may be added over time.
#
BaseEncoding
  : 'msgpack'
  | 'json'
  ;

Parameter
  : 'document-type'
  | 'version'
  | /[\w-]+/   # other parameters as directed by a particular document-type
  ;

Value
  : /[\w-]+/
  ;

# A blank line is used as the delimiter as specified by the MIME RFC
# it enables delivering mime type parameters in the future without
# requiring that they are all on one line.
#
BlankLine
  : '\r'? '\n' '\r'? '\n'
  ;

Payload: <OPAQUE-MIME-TYPE-DATA> <EOF>
  ;
```

The XPP format is a specific serialization (document-type = xpp), and
it has a version that is probably superfluous as the opaque Pcore serialization
in itself contains versioning information, but it is there for unforeseen reasons.

The document-type defines the expectancy what the deserialized content is a ParseResult.

```
# application/vnd.puppet.pcore+msgpack
; document-type=xpp
; version=1

<OPAQUE-MIME-TYPE-DATA>
<EOF>
```
Prior Art
---

Prior art in this area exists.

Java compiles source code to byte codes and delivers the ready to run result in jar files.
Production of byte code is always done off-line with a Java compiler.

Python creates `.pyc` files automatically from their corresponding
`.py` source files; these contains compiled bytecode. See the
Link: [PyFAQ about pyc][1]. Python first stored the .pyc files in
the same directory as the `.py` files, but later in
a repository - see [PEP-3147 -- PYC Repository Directories][2] for more details.

The Ecore technology (which is already in use in Puppet via RGen) is an industry
standard for modeling and serialization.

Alternatives
---

Are there other approaches that we should consider?

Regarding the location of the structure that holds `.xpp` files - we considered:

* Each entry on the module path is given its own directory under
  the `.xpp` directory and an index file is maintained in the `.xpp`
  directory with a map between module root location on disk
  to a directory under `.xpp`.
* The entry under `modules` in the environment is processed relatively
  as in the main proposal. All other locations are taken as absolute
  locations on the machine, and turned to being relative under an 
  `absolute` (or similar) directory - for example a
  `/tmp/modules/mymodule/manifests/foo.pp` would resolve to
  `<envroot>/.xpp/absolute/tmp/modules/mymodule/manifests/foo.xpp`

These were rejected because of the extra logic needed to resolve a name and the
longer paths which also has a performance cost.

We could consider:

* Defining a byte code oriented format for the Puppet Language. Such a
  format would be more efficient (than evaluating an AST), but
  requires a lot more work.

* We could consider waiting until the native C++ compiler is
  completely implemented and feature compatible with the Ruby
  implementation. As initial exploration of XPP has shown, we think
  XPP is of value to users before a complete native compiler is ready.

* We could embed the C++ parser as a native library inside of the Ruby
  runtime, but would then have to build something of equal complexity
  to efficiently communicate between the Ruby and native side. This
  would also not have the other positive side effects XPP provides.
  This alternative is rejected because it adds technical complexity
  throughout the build, packaging and deliver pipeline for
  both JRuby and MRI.

* We could, if found to be desirable also produce the `.xpp` by
  serializing the result of the Ruby based parser. This is not
  expensive to add and it may be implemented for the sake of
  simplifying A/B comparison between parser implementations. However
  without the several order of magnitude performance improvement
  provided by the C++ parser the result of doing this on the Ruby side
  would not be as beneficial in supporting the imagined workflows
  (parse everything). We therefore decided to not focus on
  this option.

* We could invoke the C++ implementation via fork/exec, as a service
  or as a linked native library. As all of those choices require
  substantial effort, the intent is that the first release only
  supports off-line production of .xpp. The issues related to
  fork/exec and native library are not discussed in this PRFC. We
  decided to focus on the merits of having the XPP file and the
  quality of the C++ based parser rather than the technical packaging
  and invocation mechanisms.

* We imagine that the C++ based parser will be used as part of atomic
  code deploy offered by Puppet Server. It can execute a
  parse/validation of all code as part of making a new version of the
  code available.

Additional Concerns:

* Other Puppet components:
  * In the short term, an XPP file would transparently be
    selected by the Puppet parser instead of parsing. Everything
    else remains the same.
  * Longer term, the C++ based parser and XPP affects Puppet Server
    and atomic code deploys, as well as information services (index or
    all classes and parameters).

* Compatibility: This has no immediate impact on compatibility. Longer
  term it may affect module compatibility (if it is expected that
  published modules have XPP files already built).

* Security: XPP does not have a direct security impact. XPP files
  contain what .pp source files contain. When written they must be
  written with appropriate owner, group and mode.

* Performance: It is expected that the basic use case and some of the
  proposed later added features will have a positive effect on
  performance.

* User Experience: Initially the production of XPP may be somewhat
  disconnected from the main flow. Once past the experimental state,
  the use of XPP should mostly be transparent to users.

* I18N: The XPP format has one feature that directly relates to L10n
  as the intent is to provide error messages in a format that lends
  itself to translation at the point an XPP file is read/used.

* Accessibility: XPP has no impact on accessibility.

* Portability: XPP is a portability mechanism - it bridges between a
  C++ implementation, and a Ruby runtime. As spin off, the
  serialization format will help with additional portability use
  cases.

* Packaging and Installation: To benefit from the C++ implementation
  of the Puppet Parser it must be installed. It is expected that it
  will be included in the AIO package when it has reached maturity.

* Documentation: The XPP format does not require extensive
  user facing documentation. In the longer term, the workflows that
  XPP enables will lead to adjustments to documentation for those
  workflows (deploying new code, CI pipelines).

* Potential (non evaluated) Spin offs (some requiring additional
  work if wanted):
  * The serialization format is a major spin off. It is intended to be
    used in a wide range of use cases; from tight integration
    between a C++ based compiler calling out to co-processes written
    in other languages), to serialization of puppet data,
    to Java Script applications visualizing data.
  * The C++ based parser has better error messages and can show the
    error in context. This will probably be used by module authors to
    help them find and more quickly address problems in their code.
  * While not a direct feature of XPP; Due to the speed with which
    parsing and validation can take place
    in the C++ parser, it makes it possible to add what to date has
    deemed to be too expensive to implement as static validation in
    the Ruby version. This can potentially lead to that the C++
    parser can replace the linter (which  has problems in keeping even
    step with the language evolution). In order to replace the linter,
    it must be able to also handle plugin-rules. Any such work is a
    separate project and is not covered by the PRFC.
  * As a side effect we can also remove the use of RGen
    (reduces footprint, startup time, and speeds up the Ruby
    implementation).

[1]: http://effbot.org/pyfaq/how-do-i-create-a-pyc-file.htm
[2]: https://www.python.org/dev/peps/pep-3147/

