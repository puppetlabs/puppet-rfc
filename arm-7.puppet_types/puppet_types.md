(ARM-7) Puppet Types
====================

This ARM introduces the idea that Puppet Types should be expressible in the Puppet Language.

Why is it of value to be able to express Types in the Puppet Language?

* Users do not have to use Ruby.
* Reduces (longer term Removes) the need to have a Ruby runtime in order to operate on types.
* Enables different implementations which is especially beneficial for devices (say a C or Lua runtime).

Background
----------
The current type system in Puppet is based on resource types that are implemented using an internal Ruby DSL.
There are many features that are almost never used, and with the open nature of DSL's in Ruby the definitions are somewhat
open ended. In practice (after having investigated the types in the puppet distribution), it was found that most of the
types were reasonable well implemented. There is however unnecessary complexities between providers and types
that blurs their boundaries in some cases. 

This proposal is based on what is believed to be the actual needs for expressing types (resource types, as well as
other kinds of "structured data"). 

Goals
-----
This ARM should:

* define a model and concrete syntax for definition of types in Puppet.
* define a Type so that it can represent a Resource Type, as well as represent a general data structure.
* show how to support Polyglot implementations (different target languages).
* show a migration path for current Ruby implemented types.

Dependencies
------------
This ARM requires:

* ARM-2.Iteration (Lambdas)
* ARM-TBD.Expression Based Grammar

Type
====

Proposed Grammar
----------------
Here is a proposed grammar for Type (in EBNF).

* `*` - zero or more
* `?` - zero or one
* `[x | y]` - reference to an existing object of type x by parsing the rule/token y
* `'x'` - token with text x
* CAPS - token
* `()` - grouping

All semantic aspects not described in grammar - see details.

    Type
      : 'type' QualifiedName ('inherits' [Type | QualifiedReference])? '{' 
          statements += TypeStatement* 
        '}'
      ;
          
    TypeStatement
      : Attribute | Reference | Invariant | Abstract
      ;
          
    Attribute
      : 'attr' NAME ',' TypeRefOrEnum ('{' ':' AttributeSetting (',' AttributeSetting)* ','? '}')?
      ;
    
    Reference
      : 'has' NAME ',' TypeRefOrEnum ('{' ReferenceSetting (',' ReferenceSetting)* ','? '}')?
    
    FeatureSetting
      : 'min'        '=>' INTEGER       # min bound, default 0
      | 'max'        '=>' IntOrUnbound  # max bound, default 1
      | 'unsettable' '=>' BOOLEAN       # if true, empty is different than not set
      | 'validate'   '=>' Validation
      | 'derived'    '=>' Lambda        # computed, not part of serialization
      | 'transient'  '=>' BOOLEAN       # not included in serialization, true if derived
      | 'setable'    '=>' BOOLEAN       # default true for regular, false for derived
      ;
    
    ReferenceSetting
      : FeatureSetting 
      | 'containment' '=>' BOOLEAN     # default true, as this is the most common 
      ;
    
    AttributeSetting
      : FeatureSetting 
      | 'default'    '=>' LiteralValue # in string form, converted to datatype
      | 'check'      '=>' Validation   # optional validation (after set)
      | 'set'        '=>' Lambda       # default sets a value (any datatype)
      | 'get'        '=>' Lambda       # default gets a value of given datatype
      ;
    
    TypeRefOrEnum
      : 'Integer'
      | 'Float'
      | 'String'
      | 'Boolean'
      | Enum
      | QualifiedReference
      ;
    
    LiteralValue
      : INTEGER
      | FLOAT
      | STRING
      | BOOLEAN
      | NAME 
      ;
    
    IntOrUnbound
      : INTEGER
      | 'unbound'
      ;
    
    Enum
      : 'enum' '[' NAME (',' NAME)* ','? ']'
      ;
    
    Invariant
      : 'invariant' (DoubleQuotedString)? Validation
      ;
            
    Validation
      : BlockExpression
      | Lambda
      ;
    
    Abstract : 'abstract' ; # marks type as abstract (can not be instantiated)
    
    Lambda
      : # As proposed in ARM-2.Iteration 
      ;

    QualifiedName       : '::'? NAME ('::' NAME)* ;
    QualifiedReference  : '::'? UCNAME ('::' UCNAME)* ;
    NAME                : [_a-zA-Z][_a-zA-Z0-9]* ;
    UCNAME              : [A-Z][_a-zA-Z0-9]* ;


This grammar is constructed in a way that only `type` becomes a keyword that may clash with existing logic (name
of resource type, name of function). All other keywords are only recognized inside of type declarations.

It is also of value to allow invariants as a general expression when performing checks in a `define`. If this is
added, then `invariant` also becomes a new keyword visible in general puppet logic.

Semantics
---------

### Data Types
New Datatypes can not be constructed (only the built in "universally available" data types can be used as attributes;
Integer, Float, String, Boolean, and Enum, where Enum defines a set of symbolic values).

### Attribute, Reference and Feature
An Attribute is a contained value of a basic data type, it is a kind of feature of the object. A Reference is a reference
to another Type, is is also a kind of Feature. All feature have a name, which must be unique within the inheritance chain
of a type.

A Reference is by default a containment reference; this means the referenced object is part of the object it is contained in,
and it is serialized as an integral part of its container. A non containment reference is serialized as (typically) a
URI that must be resolved to an object (automatically on/after "load", or on demand).

### Multiplicity
All attributes (and references) have
multiplicity (min/lower bound, and max/upper bound). This means that any attribute/reference can be a list of values.
The default is min = 0, max = 1 (i.e. optional single value). If min =1, the default for max =1 (i.e. required single value).
If min > 1, the default for max is 'unbound'. For other combinations both min and max should be specified.

By convention, multi valued features are named with a plural s. (e.g. "a user has groups")

### Unsettable
If a value is unsettable, there is a difference between not having a value, and having an empty / nil value.

### Transient
A transient attribute is not included in a serialization.

### Derived
A derived attribute is computed. It is read only by default, but may be settable (thus setting several other attributes).

### Set, Get
By default, a value of the data type is set/obtained without any additional logic being applied (it may later be validated).
In order to support "munging/unmunging" it is possible to specify set and get logic that may perform data transformation
(say from a string to an internal numeric representation, and back again). 

These are implemented as lambdas. The getter receives the internal value as an argument, the setter receives the user supplied
value as an argument (any data type). The getter and setter must return a value of the specified datatype. (The produced getter
value is returned to the user, and the produced setter value is set internally in the object).

In complex cases a derived setter may be used to split a value into several attributes, the derived getter then re-assembles
them. The setter/getter logic has elevated permission to set/change values as the various attributes may be read only for
a user.

### Available variables
All features are directly available as variables.

In addition, the variable `$meta` is bound to the
feature in question (when performing validation, set/get). Additional information can be obtained via this variable.
 
* `$meta[name]` - name of feature
* `$meta[container]` - reference to the (meta) type containing the feature
* `$meta[container][name]` - the name of the containing type

TODO: Is $meta a good name? (It should express that this is description level/meta, and thus $this or $self are not as
good).

### Assignment to feature variables
It is allows to assign to a feature variable in a setter. If setting the feature in question the value produced by
the setter is then not automatically assigned. A derived value that is settable should have a setter lambda
that sets other variables (or the given value is lost).

### Check, Invariant
A `check` in an attribute allows validation logic to be associated with an attribute. Likewise, the type as a whole
may have multiple invariant expressions.

A check expression can be a `BlockExpression` (where the variable `$it` is bound be default), or a `Lambda`
(with a specified variable name available instead of `$it`).

A check expression for a type has all features bound as variables (but this is not the case for an attribute check; it should
only be concerned with checking the feature in question).

The value produced by a check expression is as follows:
<table>
<tr>
  <td>String</td>
  <td>Unacceptable value/invariant. The string is the message</td>
</tr>
<tr>
  <td>nil, undef, true</td>
  <td>Acceptable</td>
</tr>
<tr>
  <td>false</td>
  <td>Unacceptable, a standard message "Illegal value: ${it} is not an acceptable value for ${$meta[name]}"
      is issued for an attribute. For an invariant, the invariant title string is used as a message, or if not
      specified, the message "Illegal invariant" is issued.
  </td>
</tr>
</table>

#### Examples of Check and Invariant

    # Using invariant title as message
    invariant "Can not specify ensure => absent, and content at the same time" {
      $ensure == absent and $content != undef
    }
    
    # Invariant using logic to produce message
    invariant {
      if not ($one_thing and $another_thing) { "Both one_thing, and another_thing must be set" }
    }
    
    # Using default message for attribute check
    attr foo {
      check => $it != ""
    }
    
    # Checking attribute, using its name
    attr bar {
      check => { if $bar == "" { "bar can not be set to an empty string" } }
    }

    # Checking attribute using a lambda (with shorter name).
    attr the_thing_with_a_long_name {
      check => |$x| { if $x == "" { "the_thing_with_a_long_name can not be set to an empty string" } }
    }
    
    # A more complex check using a case expression
    attr fee {
      check => { case $fee {
        "" :              { "bar can not be an empty string" }
        "blue", "green" : { "bar can not be set to 'blue' nor 'green'" }
        /:/ :             { "bar may not contain a colon" }
        default :         { true }       
        }
      }

Examples
--------

    type User {
      attr password, String {
        check   => { if $password =~ /:/ { A ':' is not allowed in a password } }
      }
      
      attr password_min_age, Integer

      attr groups, String {
        min        => 0,
        max        => unbound,
        unsettable => true,      # diff between not set, and empty list
        check      => { if $it =~ /:/ { "Use group names not GID. '${it}' not acceptable as group name." } }
      }
      
      has roles, UserRole {
        max         => unbound,   # min 0 implied
      }

    type UserRole {
      attr gid, String {
        check => { if $gid =~ /\d/ { "gid must be non numeric. '${gid}' not acceptable." }
      }
    }

Instantiation
-------------
Instantiation is based on a `new` function that takes a hash with values. Further detailing can be done with
a lambda that is given the newly (not yet fully) instantiated object.

    MyType.new {
      my_attr => 10
    }
    
    new(MyType)
    new(MyType, {my_attr => 10})
    
    new(MyType) |$t| { $t[my_attr] = 10 }
    MyType.new |$t| { $t[my_attr] = 10 }
    
Each setting triggers validation of set feature's check expression. When the initialization is finished, all invariants
for the type are checked. Once the type has been initialized it becomes immutable.

If the hash contains a key for which there is no corresponding feature an error is raised. If there are non consumed
key/value pairs left on return from the call to new an error is raised.

Logic may append to multi valued attributes by using += operator. (This is different from regular puppet logic
where += always produces a new array). An initializer may naturally initialize a multi-valued feature with a literal array).

### Default values
Default values are applied as the first step of instantiation.
    
Models
------
Type is implemented by using two models (combined in one model/document, or separated as the situation demands).
The base type model is a standard ecore model - this model is free from implementation concerns (and thus does not
contain any validation or methods for derived values.

The use of a standard ecore model means that standard tools can be used to generate a implementation in some target
language, and/or to read an instance dynamically (depends on the ecore implementation support in that language).

The second model is an implementation model that associates logic in a Puppet "AST" model; an instance of an ecore model
with the type (standard ecore) model.


### Puppet Language Model restrictions

The puppet language model (used in types) is restricted; it may contain literals, arithmetic, variables, lambdas,
conditional expressions (if, unless, case, ?, in), and may call a well defined (restricted) set of functions (primarily the iteration
functions each, select, reject, collect, and reduce). (It is not allowed to create resources, defines, nor nodes or classes).
It is allowed to instantiate types that are not managed resources.

TODO: The set of functions should be modeled (to allow it to have a version; compare if a recipient has an implementation
of this version etc.).

### Association of Ecore (base type) with additional logic

This can be done in several ways:

* Using ecore annotations on the model itself.
* A separate model that links one to the other, but is then not visible to a user of the base ecore model in any way.

Using annotations has the advantage that the reader of the type metamodel is aware of the extra information (it may choose to
ignore it if the implementation does not need this information e.g. it does not construct new values), or it may have
knowledge about the extension and use this information to wire the validation etc. into its support for the model. This
may require work on the code generator/dynamic loading of a model in the particular target environment. This may be
somewhat complicated.

The separate model has the advantage that the metamodel is free from all implementation concerns and can be processed by
all standard ecore tools. An implementor must still be aware of the additional rules, but can use these with a specific
implementation that glue the pieces together.



