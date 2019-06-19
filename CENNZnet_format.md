# CENNZnet Doughnut Permission Domain Specification

**Patent Pending**

This document outlines the format of the CENNZnet Doughnut Permission Domain.

## Structure

The domain is binary encoded. It contains a version definition, and a permissions definition.

```
<VERSION><PERMISSIONS>
```

# Version

* 10 bits
    * Version number
    * LE unsigned integer
* 6 bits
    * Reserved/unused

# Permissions
## v0

**Note:** *Subject to change*

* 1 byte
    * Module count
    * LE unsigned integer
* Modules list
    * 1 byte
        * 1 bit
            * `block_cooldown` presence
        * 7 bits
            * Method count
            * LE unsigned integer
    * 32 bytes
        * Module name
        * String
    * If `block_cooldown`
        * 4 bytes
            * `block_cooldown`
            * LE unsigned integer
    * Methods list
        * 1 byte
            * 1 bit
                * `block_cooldown` presence
            * 7 bits
                * Reserved/unused
        * 32 bytes
            * Method name
            * String
        * If `block_cooldown`
            * 4 bytes
                * `block_cooldown`
                * LE unsigned integer

##### Notes

Decode steps:

1. Read the first byte, as `module_count`
2. Begin reading modules list items
    1. Read the first byte
        - Read the first bit as `has_block_cooldown` Boolean
        - Read the last 7 bits as `method_count`
    2. Read 32 bytes as `module_name`
    3. If `has_block_cooldown`
        - Read 4 bytes as 'block_cooldown`
    4. Begin reading methods list items
        1. Read the first byte
            - Read the first bit as `has_block_cooldown` Boolean
            - Ignore 7 bits
        2. Read 32 bytes as `method_name`
        3. If `has_block_cooldown`
            - Read 4 bytes as 'block_cooldown`
        4. Repeat until `method_count` matches iterations
    5. Repeat until `module_count` matches iterations


