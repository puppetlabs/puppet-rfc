ARM-16: Managing ACLs (Access Control Lists)
============================================

Summary
-------

This ARM describes the goals and motivation for managing ACLs (Access Control Lists) on Windows systems. This proposal is to add a type and provider for doing so. This proposal could have implications for multiple operating systems, but is focused on a Windows specific provider implementation. The motivation for managing ACLs is in high demand by Windows system admins as it enables more fine grained control of security with puppet.

Goals
-----

 * Provide a DSL for managing ACLs.
 * Provide a Windows specific implementation for DACL (discretionary access control lists).
 * Update puppet core not to clobber permissions when mode is not specified.
 * Update puppet core not to push permissions from the master down for an external resource.
 * Provide the ability to enhance permission sets and exact permission sets.
 * Provide sensible conventional defaults to the fine-grained security aspects. This will allow ease of use for both beginner and advanced scenarios.

Non-Goals
---------

 * Not managing SACLs (System Access Control Lists). This includes not creating `audit` access types. At least on the first release this will not be a goal (it could be a future goal).

Success Metrics
---------------

 * The module becomes the widely accepted way of managing permission sets in Puppet with Windows systems.

Motivation
----------

This is important to the long term beneficial use of Puppet on Windows. Puppet understands `mode` for permissions but `mode` doesn't translate well to Windows systems. Windows systems need a more viable use case for managing permission sets. This is something that has been identified as lacking on Puppet by our users. At least one of our competitors has a good solution for this. What we are proposing here gives us at least as much as they have and a little more with respect to fine-grained control of rights assignment and identifying users.

Description
-----------

With ACLs you can get into fine-grained detail or you can stay at a high level. Hopefully the defaults of the minimum necessary and the level you can move into is both self-documenting and conventional and useful for beginners all the way up to advanced users.

### Format

The ACL format is as follows:

    acl { 'file/folder/absolute/path':
      ensure => present,
      permissions => [
        <identity> => [<rights>,<type>,<inheritance>,<propagation>]
        ],
    }

### Target resource

At present the target is expected to be the fully qualified path to a directory or file. In the future this could be expanded to other areas like registry keys.

The ACL provider will autorequire the resource to ensure that the ordering is correct. This will likely be done in a similar fashion to how `registry_value` type requires its parent `registry_key`: <https://github.com/puppetlabs/puppetlabs-registry/blob/master/lib/puppet/type/registry_value.rb#L122-L134>

**Note**: In the future as we support more types, it is expected that we will include a `type` parameter that will default to `type => file`. This will allow an non-breaking upgrade path.

### Ensure Parameter
`ensure` could be any of the following:

 * `present` - ensures that the permission set is present. The resource target could have other permissions as well.
 * `absent` - ensures that the permission set is not present, but would not change any additional permissions. This will also not change inherited permissions, only explicitly set permissions. This only looks at the identity of each permission.
 * `exact` - ensures that the permission set on the target resource is exactly as specified in the permissions. This will remove other explicitly defined permissions and any inherited permissions.
 * `exact_explicit` - ensures that the permissions set on the target resource is as specified in the permissions, but only applies to explicitly set permissions on the target resource. This will not remove any inherited permissions.

For `absent`, `exact` and `exact_explicit`, see [specific examples](#ensure-exact-exact_explicit-and-absent-examples) below.

**NOTE**: With `ensure => present` one may run into a validation warning at runtime that is something to the effect of `Due to unmanaged inherited permissions, a narrowing permission cannot be set`. This is simply stating that there was an inherited permission found that won't allow a permission to be set since it would narrow a particular identity's permissions (which cannot be done on Windows when a resource inherits permissions). This can be overcome by using `ensure => exact`, by managing the identity at the level where the inheritance is passed down (and possibly altering it there), or widening your current permission to match. The recommended way would be to manage the identity's permissions at the top level as widening could be a maintenance issue if the top level were to change and your permissions would be narrowing again.

**Note about exact and empty permission sets**: Using `ensure => exact` with an empty `permissions => []` will result in an validation error. Please see the [specific examples](#ensure-exact-exact_explicit-and-absent-examples) for more details.

**Warning**: While managing ACLs you could lock the Puppet Agent user completely out of managing resources. Extreme care should be used when using `exact` and `exact_explicit`.

### Permissions Property

The format of the `permissions` section is:

 * An array
 * Each element of the array is an ACE (Access Control Entry)
 * Each element of the array is an array containing (each element discussed in more detail in the following sections):
    * `identity` - trustee, principal
    * access `rights` mask
    * access `type` - defaults to `allow`
    * `inheritance` strategy - defaults to `inherit`
    * `propagation` strategy - defaults to `propagate`
 * The least amount of detail required would be the identity and access rights (i.e. `'bob' => [modify]`)
 * Each permission element in the hash is considered an ACE (access control entry)

**NOTE**: Order is important with ACEs, as Windows will evaluate ACEs in order until it finds an explicit permisison. Therefore one should start with the deny ACES, followed by user then groups. See <http://msdn.microsoft.com/en-us/library/windows/desktop/aa379298(v=vs.85).aspx>.

#### The `identity` element

The `identity` is also known as a trustee or principal - what gets assigned a particular set of permissions (or access rights). This could be in the form of

 * User - `'Bob'`
 * Group - `'Administrators'`
 * SID (Security ID) - `'S-1-5-18'`
 * Domain\UserOrGroup - `'TheNet\Bob'` or `'BUILTIN\Administrators'`
 * UserOrGroup@fqdn - `'Bob@TheNet'` or `'SomeGroup@TheNet'`

This is a string value that is translated to SID every time when using Windows.

 **Note**: In the future this could be expanded to include uid/gid.

#### The access `rights` mask element

Access `rights` define what permissions to give to the `identity`.

`rights` takes the following forms:

 * `'string access mask'` - a string value like `'mwrx'` defining the permission rights.
    * **Note** that with Windows, the only valid values here are `'mwrx'`, `'wrx'`, `'rx'` and `'r'`.
 * `'full'` - full permissions, there is no equivalent mask value. This is also known as `GENERIC_ALL`. In addition to 'mwrx', an `identity` with full permissions also can change permissions and take ownership of the target resource.
 * `'modify'` - `'mwrx'`.
 * `'write'` - maps to `'wrx'` This is also known as `GENERIC_WRITE`
 * `'read_execute'` - maps to `'rx'`
 * `'read'` - maps to `'r'`. This is also known as `GENERIC_READ`
 * **NOTE**: `read` permission implies `list` permission. If you grant `GENERIC_READ` you also get `list` permissions.
 * `binary hex flag access mask` - this maps to the binary flags for advanced permissions - i.e. `0x00010000` for DELETE

#### The access `type` element (optional)

Access `type` is one of the two following types:

 * `'allow'` - This allows the specified `rights`
 * `'deny'` - this disallows the specified `rights` - this always overrides any allowed settings, unless inherited and there is an explicit allow. See <http://msdn.microsoft.com/en-us/library/cc246052.aspx> for more information.

`type` defaults to `'allow'` - in most cases this is the permission type that is wanted.

#### The `inheritance` element (optional)

The `inheritance` strategy can be one of the following:

 * `'inherit'` - allow both objects (files) and containers (directories) to inherit this permission. This maps to `OI` `CI` security descriptors.
 * `'inherit_containers_only'` - allow inheritance on containers (directories) only. This maps to the `CI` security descriptor.
 * `'inherit_objects_only'` - allow inheritance on objects (files) only. This maps to the `OI` security descriptor.
 * `'no_inherit'` - the permission specified will only apply to the current resource.

With `inheritance` the default value is `'inherit'` as this maps directly to how it defaults with Windows.

A good resource to look at is <http://msdn.microsoft.com/en-us/library/windows/desktop/aa374928(v=vs.85).aspx>.

Inheritance is specific to containers (directories) and will only apply when the target resource is a container.

#### The `propagation` element (optional)

The `propagation` strategy defines how `inheritance` is applied. The following values can be used:

 * `'propagate'` - apply `inheritance` to all children and grandchildren
 * `'one_level'` - apply `inheritance` only to children, not to grandchildren.
 * `'inherit_only'` - apply `rights` only to children and grandchildren, not the target resource.


With `propagation` the default value is `'propagate'` is this maps directly to how it defaults with Windows.

There is a great resource at <http://msdn.microsoft.com/en-us/library/ms229747.aspx> explaining the different combinations of `inheritance` and `propagation`.

If `inheritance => 'no_inherit'` propagation will not apply and should not be specified.

### Examples

#### Simple container ACL:

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'bob' => ['full'],
        'tim' => ['read_execute']
      ],
    }

This applies the permission set to 'C:\windows\temp' with the following permissions:

 * User `'bob'` will have full permissions to this folder, subfolders, and files. This is the equivalent of `'bob' => ['full','allow','inherit','propagate']`.
 * User `'tim'` will have read/execute permissions to this folder, subfolders and files. This is the equivalent of `'tim' => ['rx','allow','inherit','propagate']`.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to `ensure => present`.

#### Simple object ACL:

    acl { 'C:/windows/temp/tempfile.txt':
      ensure => present,
      permissions => [
        'bob' => ['mwrx'],
        'tim' => ['rx','deny'],
        'Administrators' => ['full']
      ],
    }

This applies the permission set to `C:\windows\temp\tempfile.txt` with the following permissions:

 * Group `'Administrators'` will have full access to this file. This is the equivalent of `'Administrators' => ['full','allow']`.
 * User `'bob'` will have modify/write/read/execute permissions to this file. This is the equivalent of `'bob' => ['mwrx','allow']`.
 * User `'tim'` will be denied read/execute access to this file, even if part of the `'Administrators'` group.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to `ensure => present`.

**Note**: With respect to Windows, if `'tim'` is part of the `'Administrators'` group, his effective permissions will still allow him to modify/write/take ownership/change permissions. This will allow `'tim'` to give himself read and execute access, even though it was explicitly denied. When using `deny`, one needs to understand how this is applied.

#### Inherited deny with explicit allow:

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['mwrx','deny']
      ],
    }
    acl { 'C:/windows/temp/tempfile.txt':
      ensure => present,
      permissions => [
        'tim' => ['rx']
      ],
    }

This maps out to the following:

 * User `'tim'` will be denied modify/write/read/execute access to `C:\Windows\temp`, subfolders and files.
 * However, user `'tim'` will be allowed read/execute access to `C:\Windows\temp\tempfile.txt` because it was explicitly allowed without an explicit deny on the file resource.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to `ensure => present`.

#### Domain Users and SIDs

    acl { 'C:/windows/temp':
      ensure => present,
      permissions => [
        'TheNet\bob' => ['full'],
        'S-1-5-18' => ['mwrx']
      ],
    }

This maps to the following:

 * User `'TheNet\bob'` will be granted full access to `C:\Windows\temp`, subfolders, and files.
 * SID `'S-1-5-18'` will be granted modify/write/read/execute access to `C:\Windows\temp`, subfolders and files.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to `ensure => present`.

#### Ensure Exact, Exact_Explicit, and Absent Examples

##### `ensure => absent`:

    acl { 'C:/windows/temp/anotherfolder':
      ensure => absent,
      permissions => [
        'tim' => []
      ],
    }

This maps out to the following:

 * User `'tim'` will be removed from the explicit permission set, but might still have access depending on the inherited permission set.

##### `ensure => absent` with user still inherited:

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['mwrx'],
        'Administrators' => ['full']
      ],
    }
    acl { 'C:/windows/temp/anotherfolder':
      ensure => absent,
      permissions => [
        'tim' => []
      ],
    }

This maps out to the following:

 * Group `'Administrators'` will be granted full privileges to `C:\Windows\temp`, its subfolders and files.
 * User `'tim'` will be granted 'mwrx' privileges to `C:\Windows\temp`, its subfolders and files.
 * `'tim'` will be removed from the explicit permission set if he was there, but will still have 'mwrx' access due to the inherited permission set.

##### `ensure => exact`:

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['mwrx'],
        'Administrators' => ['full']
      ],
    }
    acl { 'C:/windows/temp/anotherfolder':
      ensure => exact,
      permissions => [
        'tim' => ['rx']
      ],
    }

This maps out to the following:

 * Group `'Administrators'` will be granted full privileges to `C:\Windows\temp`, its subfolders and files (except where noted below).
 * User `'tim'` will have 'mwrx' permissions to `C:\Windows\temp`, its subfolders and files (except where noted below).
 * Due to the secondary `exact` permission, `'Administrators'` will NOT have access to `C:\Windows\temp\anotherfolder`, its subfolders and files. This is because the inherited permission is removed.
 * Also due to the secondary `exact` permission, `'tim'` will have 'rx' permissions to `C:\Windows\temp\anotherfolder`, its subfolders and files. This is because the inherited permission is removed.

##### `ensure => exact` with empty permission set - INVALID Scenario:

    acl {'C:/windows/temp':
      ensure => exact,
      permissions => [],
    }

This will result in an error. While this is a perfectly legitimate permission set, it results in leaving the resource unmanageable as it removes all inherited permissions and locks down the folder completely. The only way to get into the resource from there is to by the owner.

It is suggested that instead of this, when you want to lock down a folder to one or two folks, lock it down to an admin group that at least the Puppet Agent user is in. And when you need access, add yourself to that group.

##### `ensure => exact_explicit`:

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'Administrators' => ['full']
      ],
    }
    acl { 'C:/windows/temp/anotherfolder':
      ensure => exact_explicit,
      permissions => [
        'tim' => ['rx']
      ],
    }

This maps out to the following:

 * Group `'Administrators'` will be granted full privileges to `C:\Windows\temp`, its subfolders and files.
 * Even with the secondary `exact_explicit` permission, `'Administrators'` will have full access to `C:\Windows\temp\anotherfolder`, its subfolders and files. This is because this is an inherited permission.
 * User `'tim'` will have 'rx' permissions to `C:\Windows\temp\anotherfolder`, its subfolders and files.

##### `ensure => exact_explicit` with inherited user (narrowing permissions) - INVALID Scenario:

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['mwrx']
      ],
    }
    acl { 'C:/windows/temp/anotherfolder':
      ensure => exact_explicit,
      permissions => [
        'tim' => ['rx']
      ],
    }

This maps to:

 * User `'tim'` will have modify/write/read/execute access to `C:\Windows\temp`, its subfolders and files.
 * User `'tim'` will have modify/write/read/execute access to `C:\Windows\temp\anotherfolder` because of inheritance.
 * No permission will be on `C:\Windows\temp\anotherfolder` explicitly for user `'tim'`.
 * All other existing explicitly defined users will be removed and the DACL for `C:\Windows\temp\anotherfolder` will be removed.

**NOTE**: This is considered an invalid scenario since inherited permissions cannot be narrowed.

##### `ensure => exact_explicit` with inherited user (widening permissions):

    acl {'C:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx']
      ],
    }
    acl { 'C:/windows/temp/anotherfolder':
      ensure => exact_explicit,
      permissions => [
        'tim' => ['mwrx']
      ],
    }

This maps to:

 * User `'tim'` will have read/execute access to `C:\Windows\temp`.
 * User `'tim'` will have modify/write/read/execute access to `C:\Windows\temp\anotherfolder`.
 * All other existing explicitly defined users will be removed and the DACL for `C:\Windows\temp\anotherfolder` will be removed.


#### Inheritance strategies

For each of the below, this is the structure we will be evaluating:

    file { 'c:/windows/temp/tempfile.txt':
      ensure => 'file',
      content => 'some content',
      }
    }
    file { 'c:/windows/temp/items_dir':
      ensure => 'directory',
      notify => File['c:/windows/temp/items/itemfile.txt'],
    }
    file { 'c:/windows/temp/items_dir/specific_dir':
      ensure => 'directory',
      subscribe => File['c:/windows/temp/items_dir'],
    }
    file { 'c:/windows/temp/items_dir/itemfile.txt':
      ensure => 'file',
      content => 'item file content',
    }

Given the above, we have a structure of:

    C:\Windows\temp (folder, target resource in all subsequent examples)
     -> items_dir (folder, child container)
        -> specific_dir (folder, grandchild container)
        -> itemfile.txt (file, grandchild object)
     -> tempfile.txt (file, child object)

**Note**: Even though it isn't specifically shown here, everything further down the file/folder tree is also considered just grandchildren. So if there was a file at `C:\Windows\temp\items_dir\specific_dir\some_dir\afile.txt`, `afile.txt` and `some_dir` would also be considered grandchildren of `C:\Windows\temp`.

**Note**: Narrowing permissions (removing access from an inherited permission) on a target resource is not allowed on Windows. If you want to narrow permissions, you must use `ensure => exact` which will remove the inherited permissions and set the permissions to exactly what has been specified.

##### Default Inheritance:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit','propagate']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
    * `C:\Windows\temp\tempfile.txt` (child object)

##### Container Only Inheritance:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit_containers_only','propagate']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

##### Object Only Inheritance:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit_objects_only','propagate']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)

##### No Inheritance:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','no_inherit']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp` (target resource)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)

This will create the permission with no inheritance to children or grandchildren.

##### Default Inheritance with One Level Propagation:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit','one_level']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\tempfile.txt` (child object)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

##### Default Inheritance with Inherit Only Propagation:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit','inherit_only']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp` (target resource)

##### Container Only Inheritance with Inherit Only Propagation:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit_containers_only','inherit_only']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

##### Object Only Inheritance with One Level Propagation:

    acl {'c:/windows/temp':
      ensure => present,
      permissions => [
        'tim' => ['rx','allow','inherit_objects_only','one_level']
      ],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\tempfile.txt` (child object)
 * User `'tim'` will not have read/execute access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

##### Other examples:

A couple of combinations have been left off as one should be able to understand the implications based on the examples above. If you feel that one or more are necessary, please submit a PR.

Open Questions
--------------

 1. Should puppet attempt to reorder ACEs so that the order meets <http://msdn.microsoft.com/en-us/library/windows/desktop/aa379298(v=vs.85).aspx>?
 1. With respect to `rights` - Should we support `'string access mask'` such as `'mwrx'` or is it enough just to have `'modify'`?
 1. Do users want a very specific `list` `right`? Is it enough to understand that `GENERIC_READ` already grants this right?
 1. Would users want to be able to specify multiple `rights` so we could put them together? For example `'bob' => [[read,delete],allow]`?

Future Considerations
---------------------

 * Would this translate well to other securable objects like registry, services, etc?
 * Would this translate to POSIX ACLs with minimal work?
 * With POSIX there is a default permissions mask, how would we model that?

Alternatives and Recommendation
-------------------------------

 1. We could enhance `mode` to accept acl permission sets. This is undesirable as it changes the core type and could break quite a bit of functionality unintentially. This would necessitate the need to have it become part of the next major version.
 1. We could do nothing. This is undesirable as it wouldn't give us a good solution to the permissions on Windows story.
 1. It's recommended we move forward with this ARM as it gives Puppet the best leverage with respect to Windows permissions.


Dependencies
------------

We need to enhance Puppet so that it doesn't clobber permissions on Windows when `modes` are not set (and doesn't clobber the `SYSTEM` user's permissions). This would need to be in prior to having this feature set work properly.

Impact
------

This feature will enable users to manage a critical piece of configuration management on Windows.
