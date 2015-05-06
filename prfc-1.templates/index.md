Templates
=========
This is the ARM templates ARM. It contains the templates used when initiating new ARMs.
Templates are found under the templates directory. To create a new ARM:

1. get an ARM title and brief description from the author 
2. edit the table in README.md in the root of the repository to assign the next sequential number to the new ARM and add its title as a hyperlink of the form `arm-XX.title_with_underbars`, description, version 0.0.1, the current date, and status "New"
3. create a new ARM directory in the repository by copying the templates/ subdirectory here to the new ARMs name. edit the metadata.json file to (at the least) contain the correct title, description, and author's contact information.
4. commit your changes and push, so the ARM author can fork and submit pull requests against their directory from that point.

Note that the metadata.json in this ARM's root is for the template arm-1 itself and should not be copied.
