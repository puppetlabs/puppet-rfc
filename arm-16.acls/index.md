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

 * Not managing SACLs (System Access Control Lists). This includes not creating `audit` access types. On the first release this will not be a goal (it could be a future goal).

Success Metrics
---------------

 * The module becomes the widely accepted way of managing permission sets in Puppet with Windows systems.

Motivation
----------

This is important to the long term beneficial use of Puppet on Windows. Puppet understands `mode` for permissions but `mode` doesn't translate well to Windows systems. Windows systems need a more viable use case for managing permission sets. This is something that has been identified as lacking on Puppet by our users. At least one of our competitors has a good solution for this. What we are proposing here gives us at least as much as they have and a little more with respect to fine-grained control of rights assignment and identifying users.

Terminology
-----------

 * ACE (Access Control Entry) - a set of rights with inheritance and propagation for an identity.
 * ACL (Access Control List) - simply an ordered list of ACEs.
 * DACL (Discretionary Access Control List) - this type of ACL holds ACEs that allow or deny access to a securable object.
 * SACL (System Access Control List) - this type of ACL holds ACEs that audit access to securable objects.
 * Security Descriptor - an object that holds security information for a securable object. This holds a DACL, possibly a SACL, the owner of the securable object, and whether the inherited permissions are allowed onto the securable.

References:

 * <http://msdn.microsoft.com/en-us/library/windows/desktop/aa374862(v=vs.85).aspx>
 * <http://msdn.microsoft.com/library/windows/desktop/aa374872(v=vs.85).aspx>

Description
-----------

With ACLs you can get into fine-grained detail or you can stay at a high level.

### Format

#### ACL Format

    security_descriptor { 'absolute/path':
      owner => <owner>,
      dacl => <acl>,
      inherit_parent_permissions => <true | false>,
    }

    acl { 'absolute/path':
      ensure => <present>,
      purge_unspecified_explicit_permissions => <purge_unspecified_explicit_permissions>,
      permissions => [
        Ace['resource_name']
        ],
    }

    ace {'resource_name':
      identity => <identity>,
      rights => [<rights>],
      type => <type>,
      inherit => <inheritance>,
      propagate => <propagation>,
    }

where with `security_descriptor`: `owner` and `inherited_permissions` have sensible defaults; with `ace`: `type`, `inherit` and `propagate` have sensible defaults (discussed in their appropriate sections below). `Rights` is an array of rights that can contain one or more rights (discussed in the rights section below).

#### ACL::File_ACL Defined Type

    acl::file_acl($path = $title, $permissions, $ensure, $purge_unspecified_explicit_permissions, $inherit_parent_permissions, $owner)

where the only required elements are `$path` and `$permissions`.

### Security Descriptor Type

    security_descriptor { 'absolute/path':
      owner => <owner>,
      dacl => <acl>,
      inherit_parent_permissions => <true | false>,
    }

#### Title: Target Resource Property

At present the target is expected to be the fully qualified path to a directory or file. In the future this could be expanded to other areas like registry keys.

The Security Descriptor provider will autorequire the resource to ensure that the ordering is correct. This will likely be done in a similar fashion to how `registry_value` type requires its parent `registry_key`: <https://github.com/puppetlabs/puppetlabs-registry/blob/master/lib/puppet/type/registry_value.rb#L122-L134>

**Note**: In the future as we support more types, it is expected that we will include a `type` parameter that will default to `type => file`. This will allow an non-breaking upgrade path.

#### Inherit_Parent_Permissions Property

`inherit_parent_permissions` specifies whether to inherit permissions from parent ACLs or not. The default is `true`.

**NOTE**: With `inherit_parent_permissions => true` (which is the default), one may run into a validation warning at runtime that is something to the effect of `Due to  inherited permissions, a narrowing permission cannot be set`. This is simply stating that there was an inherited permission found that won't allow a permission to be set since it would narrow a particular identity's permissions (which cannot be done on Windows when a resource inherits permissions). This can be overcome by using `inherit_parent_permissions => false` on the `security_descriptor`, by managing the identity at the level where the inheritance is passed down (and possibly altering it there), or widening your current permission to match. The recommended way would be to manage the identity's permissions at the top level as widening could be a maintenance issue if the top level were to change in a way that would cause your permissions to be narrowing again.

**Warning**: While managing ACLs you could lock the Puppet Agent user completely out of managing resources. Extreme care should be used when using `purge_unspecified_explicit_permissions => true` on `acl` with `inherit_parent_permissions => false` on the `security_descriptor`. Almost never should an admin also include `acl` `permissions => []`, which would cause the provider to remove all permissions to a resource.

#### Owner Property

The owner expressed in the same format as the `identity`. See the Identity property below in ACE Type below.
The `owner` is an `identity` that

This could be in the form of

 * User - `'Bob'`
 * Group - `'Administrators'`
 * SID (Security ID) - `'S-1-5-18'`
 * Domain\UserOrGroup - `'TheNet\Bob'` or `'BUILTIN\Administrators'`

This is a string value that is translated to SID every time when using Windows.

 **Note**: In the future this could be expanded to include uid/gid.

### ACL Type

    acl { 'unique_title':
      ensure => <present>,
      purge_unspecified_explicit_permissions => <true | false>,
      permissions => [
        Ace['resource_name']
        ],
    }

#### Title Parameter
This can be anything that uniquely identifies the ACL.

The ACL provider will autorequire the resource to ensure that the ordering is correct. This will likely be done in a similar fashion to how `registry_value` type requires its parent `registry_key`: <https://github.com/puppetlabs/puppetlabs-registry/blob/master/lib/puppet/type/registry_value.rb#L122-L134>

#### Ensure Parameter
`ensure` could be any of the following:

 * `present` - ensures that the permission set is present. The resource target could have other permissions as well.
 * `absent` - ensures that each ACE element in the permission set is not present, but would not change any additional permissions. This will also not change inherited permissions, only explicitly set permissions. **Note**: This only looks at the identity of each permission.

Ensure defaults to `present`.

#### Purge Unspecified Explicit Permissions Parameter

`purge_unspecified_explicit_permissions` specifies whether to remove other explicit permissions if not specified in the permissions set. Defaults to `false`.

#### Permissions Property

The format of the `permissions` section is:

 * An array
 * Each element of the array is an ACE (Access Control Entry) - which maps to the `ace` type.
 * Each ACE can be a defined ace resource or an inlined ACE hash.
 * Each element of the array is a hash containing (each element discussed in more detail in the following sections):
    * `identity` - trustee, principal
    * access `rights` mask - an array of `rights`
    * access `type` - defaults to `allow`
    * `inheritance` strategy - defaults to `inherit`
    * `propagation` strategy - defaults to `propagate`
 * The least amount of detail required would be the identity and access rights (i.e. `'bob' => [modify]`)
 * Each permission element in the hash is considered an ACE (access control entry)

**NOTE**: Order is important with ACEs, as Windows will evaluate ACEs in order until it finds an explicit permisison. Therefore one should start with the deny ACES, followed by user then groups. See <http://msdn.microsoft.com/en-us/library/windows/desktop/aa379298(v=vs.85).aspx>.

### ACE Type

    ace {'resource_name':
      identity => <identity>,
      rights => [<rights>],
      type => <type>,
      inherit => <inheritance>,
      propagate => <propagation>,
    }

#### Title Parameter
This can be anything that uniquely identifies the ACE.

#### Identity Property

The `identity` is also known as a trustee or principal - what gets assigned a particular set of permissions (or access rights). This could be in the form of

 * User - `'Bob'`
 * Group - `'Administrators'`
 * SID (Security ID) - `'S-1-5-18'`
 * Domain\UserOrGroup - `'TheNet\Bob'` or `'BUILTIN\Administrators'`

This is a string value that is translated to SID every time when using Windows.

 **Note**: In the future this could be expanded to include uid/gid.

#### Access Rights Mask Property (Array)

Access `rights` define what permissions to give to the `identity`. This is an array of permissions to apply.

`rights` takes the following forms:

 * `'string access mask'` - a string value like `'mwrx'` defining the permission rights.
    * **Note** that with Windows, the only valid values here are `'mwrx'`, `'wrx'`, `'rx'` and `'r'`.
 * `full` - full permissions, there is no equivalent mask value. This is also known as `GENERIC_ALL`. In addition to 'mwrx', an `identity` with full permissions also can change permissions and take ownership of the target resource.
 * `modify` - `'mwrx'`.
 * `write` - maps to `'wrx'` This is also known as `GENERIC_WRITE`
 * `list` - With respect to containers, this grants listing the items in a container.
 * `read` - maps to `'r'`. This is also known as `GENERIC_READ`
 * `execute` - maps to `'x'`. This is also known as `GENERIC_EXECUTE`.
 * `binary hex flag access mask` - this maps to the binary flags for advanced permissions - i.e. `0x00010000` for DELETE

This can be specified as an array (i.e. `rights => [read,list]`) or just as a string (i.e. `rights => read`).

#### Access Type Property (optional)

Access `type` is one of the two following types:

 * `allow` - This allows the specified `rights`
 * `deny` - this disallows the specified `rights` - this always overrides any allowed settings, unless inherited and there is an explicit allow. See <http://msdn.microsoft.com/en-us/library/cc246052.aspx> for more information.

`type` defaults to `allow` - in most cases this is the permission type that is wanted.

#### Inheritance Property (optional)

The `inheritance` strategy can be one of the following:

 * `inherit` - allow both objects (files) and containers (directories) to inherit this permission. This maps to `OI` `CI` security descriptors.
 * `inherit_containers_only` - allow inheritance on containers (directories) only. This maps to the `CI` security descriptor.
 * `inherit_objects_only` - allow inheritance on objects (files) only. This maps to the `OI` security descriptor.
 * `no_inherit` - the permission specified will only apply to the current resource.

With `inheritance` the default value is `inherit` as this maps directly to how it defaults with Windows.

A good resource to look at is <http://msdn.microsoft.com/en-us/library/windows/desktop/aa374928(v=vs.85).aspx>.

Inheritance is specific to containers (directories) and will only apply when the target resource is a container.

#### Propagation Property (optional)

The `propagation` strategy defines how `inheritance` is applied. The following values can be used:

 * `propagate` - apply `inheritance` to all children and grandchildren
 * `one_level` - apply `inheritance` only to children, not to grandchildren.
 * `inherit_only` - apply `rights` only to children and grandchildren, not the target resource.

With `propagation` the default value is `propagate` is this maps directly to how it defaults with Windows.

There is a great resource at <http://msdn.microsoft.com/en-us/library/ms229747.aspx> explaining the different combinations of `inheritance` and `propagation`.

If `inheritance => 'no_inherit'` propagation will not apply and should not be specified.

### File_Acl Defined Type

    define acl::file_acl(
      $path = $title,
      $permissions,
      $ensure                                  => present,
      $purge_unspecified_explicit_permissions  => ignore,
      $inherit_parent_permissions              => true,
      $owner                                   => 'Administrators'
    ) {
      acl { $path:
        ensure                                 => $ensure,
        purge_unspecified_explicit_permissions => $purge_unspecified_explicit_permissions,
        permissions                            => $permissions,
      }

      security_descriptor { $path:
        owner                                  => $owner,
        dacl                                   => Acl[$path],
        inherit_parent_permissions             => $inherit_parent_permissions,
      }
    }

This allows you to specify file/folder permissions without needing to be extremely verbose.

## Examples

### Minimalist ACL:

    ace {'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
    }
    acl::file_acl {'c:/windows/temp':
      permissions => [Ace['tim_rxl']],
    }


This applies the permission set to 'C:\windows\temp' with the following permissions:

 * User `'tim'` will have list/read/execute permissions to this folder, subfolders and files.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to the defaults of `inherit_parent_permissions => true, purge_unspecified_explicit_permissions => false`.

### Verbose ACL:

    ace {'tim_ace':
      identity => 'tim',
      rights => [read,execute,list],
      type => allow,
      inheritance => inherit,
      propagation => propagate,
    }

    acl {'c:/windows/temp':
      ensure => present,
      purge_unspecified_explicit_permissions => false,
      permissions => [
        Ace['tim_ace']
      ],
    }

    security_descriptor {'c:/windows/temp':
      inherit_parent_permissions => true,
      dacl => Acl['c:/windows/temp'],
      owner => 'Administrators',
    }


This example is has exactly the same affect as the Minimalist ACL above.

### Simple container ACL:

    ace {'bob_full':
      identity => 'bob',
      rights => [full],
    }

    ace {'tim_rxl':
      identity=> 'tim',
      rights => [read,execute,list],
    }

    acl::file_acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['bob_full'],Ace['tim_rxl']],
    }

This applies the permission set to 'C:\windows\temp' with the following permissions:

 * User `'bob'` will have full permissions to this folder, subfolders, and files. This is the equivalent of `ace {'bob_full': identity => 'bob', rights => [full], type => allow, inheritance => inherit, propagation => propagate}`.
 * User `'tim'` will have list/read/execute permissions to this folder, subfolders and files. This is the equivalent of `ace {'tim_rxl': identity => 'tim', rights => [read,execute,list], type => allow, inheritance => inherit, propagation => propagate}`.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to the defaults of `inherit_parent_permissions => true, purge_unspecified_explicit_permissions => false`.

### Simple object ACL:

    ace { 'tim_rx_deny':
      identity => 'tim',
      rights => [read,execute],
      type => deny,
    }
    ace {'bob_mwrx': identity=> 'bob', rights => [modify,write,read, execute]}
    ace {'Admins_full': identity=> 'Administrators', rights => [full]}

    acl::file_acl { 'c:/windows/temp/tempfile.txt':
      ensure => present,
      permissions => [ Ace['tim_rx_deny'], Ace['bob_mwrx'], Ace['Admins_full']],
    }

This applies the permission set to `C:\windows\temp\tempfile.txt` with the following permissions:

 * User `'tim'` will be denied read/execute access to this file, even if part of the `'Administrators'` group.
    * With a `type => deny`, this right will be ordered first in the ACL so that it is evaluated ahead of allow permissions.
 * Group `'Administrators'` will have full access to this file.
 * User `'bob'` will have modify/write/read/execute permissions to this file.

 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to the defaults of `inherit_parent_permissions => true, purge_unspecified_explicit_permissions => false`.

**Note**: With respect to Windows, if `'tim'` is part of the `'Administrators'` group, his effective permissions will still allow him to modify/write/take ownership/change permissions. This will allow `'tim'` to give himself read and execute access, even though it was explicitly denied. When using `deny`, one needs to understand how this is applied.

### Inherited deny with explicit allow:

    ace {'tim_mwrx_deny': identity => 'tim', rights=> [modify, write, read, execute, list]}
    acl::file_acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_mwrx_deny']],
    }

    ace {'tim_rx': identity => 'tim', rights=>[read,execute]}
    acl { 'c:/windows/temp/tempfile.txt':
      ensure => present,
      permissions => [Ace['tim_rx']],
    }

This maps out to the following:

 * User `'tim'` will be denied modify/write/read/execute/list access to `C:\Windows\temp`, subfolders and files.
 * However, user `'tim'` will be allowed read/execute access to `C:\Windows\temp\tempfile.txt` because it was explicitly allowed without an explicit deny on the file resource.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to the defaults of `inherit_parent_permissions => true, purge_unspecified_explicit_permissions => false`.

### Domain Users and SIDs

    ace {'TheNet\bob_full': identity => 'TheNet\bob', rights => [full]}
    ace {'S-1-5-18_mwrx': identity => 'S-1-5-18', rights=> [modify,write,read,execute,list]}
    acl::file_acl { 'c:/windows/temp':
      ensure => present,
      permissions => [Ace['TheNet\bob_full'],Ace['S-1-5-18_mwrx']],
    }

This maps to the following:

 * User `'TheNet\bob'` will be granted full access to `C:\Windows\temp`, subfolders, and files.
 * SID `'S-1-5-18'` will be granted modify/write/read/execute access to `C:\Windows\temp`, subfolders and files.
 * Any other identities that were are not explicitly noted would be left alone on the ACL of the target resource due to the defaults of `inherit_parent_permissions => true, purge_unspecified_explicit_permissions => false`.

### Absent Examples

#### `ensure => absent`:

    ace {'tim_rxl': identity => 'tim', rights => [read,execute,list]}

    acl::file_acl { 'c:/windows/temp/anotherfolder':
      ensure => absent,
      permissions => [Ace['tim_rxl']],
    }

This maps out to the following:

 * User `'tim'` will be removed from the explicit permission set, but might still have access depending on the inherited permission set.
 * The rights are not evaluated with ensure absent, only the identity.

#### `ensure => absent` with user still inherited:

    ace {'Admins_full': identity=> 'Administrators', rights => [full]}
    ace {'tim_mwrx': identity=> 'tim', rights=> [modify,write,read,execute,list]}
    acl::file_acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['Admins_full'],Ace['tim_mwrx']],
    }
    acl::file_acl { 'c:/windows/temp/anotherfolder':
      ensure => absent,
      inherit_parent_permissions => true,
      permissions => [Ace['tim_mwrx']],
    }

This maps out to the following:

 * Group `'Administrators'` will be granted full privileges to `C:\Windows\temp`, its subfolders and files.
 * User `'tim'` will be granted 'mwrx' privileges to `C:\Windows\temp`, its subfolders and files.
 * `'tim'` will be removed from the explicit permission set if he was there, but will still have 'mwrx' access due to the inherited permission set.
 * **Note**: Even though `inherit_parent_permissions => true` is shown, it is the default, so this line is only expressed for clarity.

### Unspecified Permissions Examples

#### `purge_unspecified_explicit_permissions => true`:

    ace {'Admins_full': identity => 'Administrators', rights => [full]}
    ace {'tim_mwrx': identity => 'tim', rights => [modify,write,read,execute,list]}
    acl::file_acl {'C:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_mwrx'],Ace['Admins_full']],
    }
    ace {'tim_rxl': identity => 'tim', rights => [read,execute,list]}
    acl::file_acl { 'C:/windows/temp/anotherfolder':
      ensure => present,
      purge_unspecified_explicit_permissions => true,
      permissions => [Ace['tim_rxl']],
    }

This maps out to the following:

 * Group `'Administrators'` will be granted full privileges to `C:\Windows\temp`, its subfolders and files (except where noted below).
 * User `'tim'` will have 'mwrx' permissions to `C:\Windows\temp`, its subfolders and files (except where noted below).
 * Due to the secondary `exact` permission, `'Administrators'` will NOT have access to `C:\Windows\temp\anotherfolder`, its subfolders and files. This is because the inherited permission is removed.
 * Also due to the secondary `exact` permission, `'tim'` will have 'rx' permissions to `C:\Windows\temp\anotherfolder`, its subfolders and files. This is because the inherited permission is removed.

#### `purge_unspecified_explicit_permissions => true` with `inherit_parent_permissions => false` and an empty permission set

    acl::file_acl {'C:/windows/temp':
      ensure => present,
      purge_unspecified_explicit_permissions => true,
      inherit_parent_permissions => false,
      permissions => [],
    }

While this is a perfectly legitimate permission set, it results in leaving the resource unmanageable as it removes all inherited permissions and locks down the folder completely. The only way to get into the resource from there is by the owner.

It is suggested that instead of this, when you want to lock down a folder to one or two folks, lock it down to an admin group that at least the Puppet Agent user is in. And when you need access, add yourself to that group.

#### `purge_unspecified_explicit_permissions => true` with `inherit_parent_permissions => true`:

    ace {'Admins_full': identity => 'Administrators', rights => [full]}
    acl::file_acl {'C:/windows/temp':
      ensure => present,
      permissions => [Ace['Admins_full']],
    }

    ace {'tim_rxl': identity => 'tim', rights => [read,execute,list]}
    acl::file_acl { 'C:/windows/temp/anotherfolder':
      purge_unspecified_explicit_permissions => true,
      permissions => [Ace['tim_rxl']],
    }

This maps out to the following:

 * Group `'Administrators'` will be granted full privileges to `C:\Windows\temp`, its subfolders and files.
 * Even with the secondary `purge_unspecified_explicit_permissions` permission, `'Administrators'` will have full access to `C:\Windows\temp\anotherfolder`, its subfolders and files. This is because this is an inherited permission.
 * User `'tim'` will have read/execute/list permissions to `C:\Windows\temp\anotherfolder`, its subfolders and files.

#### `purge_unspecified_explicit_permissions => true` with `inherit_parent_permissions => true` where attempting to narrow permissions on an inherited user - INVALID Scenario:

    ace {'tim_mwrx': identity => 'tim', rights => [modify,write,read,execute,list]}
    acl::file_acl {'C:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_mwrx']],
    }

    ace {'tim_rxl': identity => 'tim', rights => [read,execute,list]}
    acl::file_acl { 'C:/windows/temp/anotherfolder':
      ensure => present,
      purge_unspecified_explicit_permissions => true,
      permissions => [Ace['tim_rxl']],
    }

This maps to:

 * User `'tim'` will have modify/write/read/execute access to `C:\Windows\temp`, its subfolders and files.
 * User `'tim'` will have modify/write/read/execute access to `C:\Windows\temp\anotherfolder` because of inheritance.
 * No permission will be on `C:\Windows\temp\anotherfolder` explicitly for user `'tim'`.
 * All other existing explicitly defined users will be removed and the DACL for `C:\Windows\temp\anotherfolder` will be removed.

**NOTE**: This is considered an invalid scenario since inherited permissions cannot be narrowed.

#### `purge_unspecified_explicit_permissions => true` with `inherit_parent_permissions => true` where attempting to widen permissions on an inherited user:

    ace{'tim_rxl': identity => 'tim', rights => [read,execute,list]}
    acl {'C:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }
    ace{'tim_mwrx': identity => 'tim', rights => [modify,write,read,execute,list]}
    acl { 'C:/windows/temp/anotherfolder':
      inherit_parent_permissions => true,
      purge_unspecified_explicit_permissions => true,
      permissions => [Ace['tim_mwrx']],
    }

This maps to:

 * User `'tim'` will have read/execute access to `C:\Windows\temp`.
 * User `'tim'` will have modify/write/read/execute access to `C:\Windows\temp\anotherfolder`.
 * All other existing explicitly defined users will be removed and the DACL for `C:\Windows\temp\anotherfolder` will be removed.

### Inheritance strategies

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

**Note**: Narrowing permissions (removing access from an inherited permission) on a target resource is not allowed on Windows. If you want to narrow permissions, you must use `inherit_parent_permissions => false` which will remove the inherited permissions.

#### Default Inheritance:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit,
      propagate => propagate,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
    * `C:\Windows\temp\tempfile.txt` (child object)

#### Container Only Inheritance:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit_containers_only,
      propagate => propagate,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

#### Object Only Inheritance:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit_objects_only,
      propagate => propagate,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)

#### No Inheritance:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => no_inherit,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp` (target resource)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)

This will create the permission with no inheritance to children or grandchildren.

#### Default Inheritance with One Level Propagation:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit,
      propagate => one_level,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\tempfile.txt` (child object)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

#### Default Inheritance with Inherit Only Propagation:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit,
      propagate => inherit_only,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp` (target resource)

#### Container Only Inheritance with Inherit Only Propagation:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit_containers_only,
      propagate => inherit_only,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\tempfile.txt` (child object)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

#### Object Only Inheritance with One Level Propagation:

    ace{'tim_rxl':
      identity => 'tim',
      rights => [read,execute,list],
      inherit => inherit_objects_only,
      propagate => one_level,
    }
    acl {'c:/windows/temp':
      ensure => present,
      permissions => [Ace['tim_rxl']],
    }

Evaluating the structure specified at the beginning of this section the following will be applied:

 * User `'tim'` will have read/execute/list access to:
    * `C:\Windows\temp` (target resource)
    * `C:\Windows\temp\tempfile.txt` (child object)
 * User `'tim'` will not have access to:
    * `C:\Windows\temp\items_dir` (child container)
    * `C:\Windows\temp\items_dir\specific_dir` (grandchild container)
    * `C:\Windows\temp\items_directory\itemfile.txt` (grandchild object)

#### Other examples:

A couple of combinations have been left off as one should be able to understand the implications based on the examples above. If you feel that one or more are necessary, please submit a PR.

Open Questions
--------------

 1. Should puppet attempt to reorder ACEs so that the order meets <http://msdn.microsoft.com/en-us/library/windows/desktop/aa379298(v=vs.85).aspx>?
 1. With respect to `rights` - Should we support `'string access mask'` such as `'mwrx'` or is it enough just to have `'modify'`?
 1. Do users want a very specific `list` `right`? Does it confuse folks coming from *nix if read and list are separate?
 1. For those that want to manage one or two permissions on an ACL, would it make sense to also create an `acl::file_ace`?

    acl::file_ace($title,$path,$identity,$rights,$type,$inherit,$propagate)

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
