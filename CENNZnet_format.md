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
            * 1 bit
                * `constraints` presence
            * 6 bits
                * reserved/unused
        * 32 bytes
            * Method name
            * String
        * If `block_cooldown` is set
            * 4 bytes
                * `block_cooldown`
                * LE unsigned integer
        * If `constraints` is set
            * 1 byte
                * Constraints length


##### Notes

Decode steps:

1. Read the first byte, as `module_count`, and increment it by 1
    - ∵ A valid `permissions` object requires at least one module to be specified, resulting in `module_count` always being greater than 0. This allows us to avoid representing 0, such that our `module_count` is defined in the range, `[0 + 1, 256 + 1]`.
2. Begin reading modules list items
    1. Read the first byte
        - Read the first bit as `has_block_cooldown` Boolean
        - Read the last 7 bits as `method_count`
    2. Read 32 bytes as `module_name`
    3. If `has_block_cooldown`
        - Read 4 bytes as `block_cooldown`
    4. Begin reading methods list items
        1. Read the first byte
            - Read the first bit as `has_block_cooldown` Boolean
            - Read the second bit as `has_constraints` Boolean
            - Ignore 6 bits
        2. Read 32 bytes as `method_name`
        3. If `has_block_cooldown`
            - Read 4 bytes as `block_cooldown`
        4. If `has_constraints`
            - Read 1 byte as `constraints_length` and increment it by 1
                - ∵ There exists a bit to indicate the presence of constraints (i.e., `has_constraints`), which allows us to define `constraints_length` in the range of `[0 + 1, 256 + 1]`.
            - Read `constraints_length` bytes as `constraints`
        5. Repeat until `method_count` matches iterations
    5. Repeat until `module_count` matches iterations

Wildcards:
1.  Any module named `"*"` is a wildcard
    1.  A wildcard module enables all modules
    2.  Constraints applied to a wildcard module such as `methods` and `block_cooldown` apply
    3.  Named module constraints take priority over wildcard module constraints
2.  Any method named `"*"` is a wildcard
    1.  A wildcard method enables all methods for the module it is listed under
    2.  Constraints applied to a wildcard method such as `block_cooldown` and `constraints` apply
    3.  Named method constraists take priority over wildcard module constraints
