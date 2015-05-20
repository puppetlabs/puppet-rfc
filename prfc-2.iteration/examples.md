(ARM-2) Iteration Examples
==========================

This document contains examples of the Recommended Implementation.
### Update
The examples have been updated to Puppet 3.4 w.r.t renamed iterative functions.

### Creating an Array by appending to each element of another Array

Question from mailing list:
> How can I append `"/var/bricks"` to each item in the array? Lack of a
> looping construct makes this challenging in puppet, such that:
> `brick_array = ['gfs01:/var/bricks', 'gfs02:/var/bricks', ... ]` ?

Here is how:

    $nodes = ['gfs01' ,'gfs02', 'gfs03', 'gfs04']
    $brick_store = "/var/bricks"
    $brick_array = $nodes.map |$x| { "$x:$brick_store" }

### Combining Two Structures
From the mailinglist:

> I need to generate a file with this content:
>
>    /bin/mount --bind /home/some/path/ /home/someuser/www  
>    /bin/mount --bind /home/comple/tely/different/path/ /home/differentuser/www  
>    /bin/mount --bind /home/another/path/ /home/anotheruser/www  
>  
> For each row I need to insert two variables, the `path` (different per user), and `.../user/www`.

**By reducing a hash**

    $data = {
      'someuser'     => 'some/path', 
      'differentuser'=>'complete/tely/different/path', 
      'anotheruser'  => 'another/path'
    }
    $content = $data.reduce("") |$memo, $x| { "$memo
    /bin/mount --bind /home/${x[1]}/ /home/${x[1]}/www" }
    
    # content contains the desired text (although with a blank line first)

**By map and join**

This can also be implemented
as mapping to an array, and applying the standard library `join` function on the result (joining entries with a new line).

    # function call style
    $content = join($data.map |$x| "/bin/mount --bind /home/${$x[1]}/ /home/${x[0]}/www" }, "\n")
    
    # or method call style
    $content = $data.map |$x| "/bin/mount --bind /home/${$x[1]}/ /home/${x[0]}/www" }.join("\n")


**zip of two arrays, map, and join**

If the structure is in two arrays, the stdlib function `zip` can be used to combine them:

    $users = ['someuser', 'differentuser', 'anotheruser']
    $paths = ['some/path', 'complete/tely/different/path', 'another/path']
    $content = zip($users, $paths).map |$x| "/bin/mount --bind /home/${x[1]}/ /home/${x[0]}/www" }.join("\n")

**slize one array, map, and join**

If the structure is in one array, the `slice` function (in this ARM) can be used:

    $users = ['someuser', 'some/path', 'differentuser', 'complete/tely/different/path', 'anotheruser', 'another/path']
    $content = $users.slice(2).map |$x| { "/bin/mount --bind /home/${paths[$x]}/ /home/$x/www" }.join("\n")

### Iterating over an Array - Creating Resources

> Question:
> I have a list of users and need to create a file resource for each user and set the owner to that user.
> (Sure, I can pass an array as title, but I need to also set owner...)
>

Here is how this can be done:

    $usernames.each |$x| { file { "/home/$x/.somerc": owner => $x } }

### Iterating over pairs in an array or hash

> Question:
> I have an array with user-names and modes, I need to create a file resource with corresponding mode.

Here is how, using `slice` function to pick pairs from the array

    $users_with_mode = ['fred', 0666, 'mary', 0777 ]
    $users_with_mode.slice(2) |$x| {
      file {"/home/{$x[0]}/.somerc":
        owner => $x[0],
        mode  => $x[1]
      }
    }

> And if they are in a hash?

Easier, that can be written as:

    $users_with_mode = ['fred' => 0666, 'mary' => 0777 ]
    $users_with_mode.each |$user, $mode| {
      file {"/home/$user/.somerc":
        owner => $user,
        mode  => $mode
      }
    }


### Creating, Collecting, and Relating Resources

    $r1 = $a1.map |$x| {file {"/somewhere/$x": owner => $x}}
    $r2 = $a2.map |$x| {file {"/elsewhere/$x": owner => $x}}
    $r1 -> $r2

or

    $a1.map |$x| { file {"/somewhere/$x": owner => $x}} ->
      $a2.map |$x| { file {"/elsewhere/$x": owner => $x}}

### Filtering Data Before Creating a resource

    $a.filter |$x| { $x =~ /com$/ }.each |$x| {
      file { "/somewhere/$x":
        owner => $x
      }
    }

### Including Classes Derived From Facts

> From user group:  
> Variable `$roles` contains names of roles (obtained via a fact), and the need is to map these
> to inclusion of classes.

Here is a very simple interpolation of the name, but could naturally use a hash lookup or similar.

    $roles.each |$x| { 
      include "our_$x"
    }

