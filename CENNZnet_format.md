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
    * `module_count`
    * LE unsigned integer
* Modules list
    * 1 byte
        * 1 bit
            * `has_block_cooldown` flag
        * 7 bits
            * `method_count`
            * LE unsigned integer
    * 32 bytes
        * `module_name`
        * String
    * If `has_block_cooldown`
        * 4 bytes
            * `block_cooldown`
            * LE unsigned integer
    * Methods list
        * 1 byte
            * 1 bit
                * `has_block_cooldown` flag
            * 1 bit
                * `has_pact` flag
            * 6 bits
                * reserved/unused
        * 32 bytes
            * `method_name`
            * String
        * If `block_cooldown` is set
            * 4 bytes
                * `block_cooldown`
                * LE unsigned integer
        * If `has_pact` is set
            * 1 byte
                * `pact_length`
                * LE unsigned integer
                * `pact_length` = byte_value + 1
            * `pact_length` bytes
                * `pact`
* 1 byte
    * `contract_count`
    * LE unsigned integer
* Contracts list
    * 1 byte
        * 1 bit
            * `has_block_cooldown` flag
        * 7 bits
            * Reserved
    * 32 bytes
        * `contract_address`
        * LE unsigned integer
    * If `has_block_cooldown`
        * 4 bytes
            * `block_cooldown`
            * LE unsigned integer

### Decode Steps

1. Read the first byte, as `module_count`
2. Begin reading module list items
    1. Read the first byte
        - Read the first bit as `has_block_cooldown` Boolean
        - Read the last 7 bits as `method_count`
    2. Read 32 bytes as `module_name`
    3. If `has_block_cooldown`
        - Read 4 bytes as `block_cooldown`
    4. Begin reading methods list items
        1. Read the first byte
            - Read the first bit as `has_block_cooldown` Boolean
            - Read the second bit as `has_pact` Boolean
            - Ignore 6 bits
        2. Read 32 bytes as `method_name`
        3. If `has_block_cooldown`
            - Read 4 bytes as `block_cooldown`
        4. If `has_pact`
            - Read 1 byte as `pact_length` and increment it by 1
                - ∵ There is a bit to indicate the presence of pact constraints (i.e., `has_pact`), which allows us to define `pact_length` in the range of `[0 + 1, 255 + 1]`.
            - Read `pact_length` bytes as `pact`
        5. Repeat until `method_count` matches iterations
    5. Repeat until `module_count` matches iterations
3. Read the next byte, as `contract_count`
4. Begin reading contract list items
    1. Read the first byte
        - Read the first bit as `has_block_cooldown` Boolean
    2. Read 32 bytes as `contract_address`
    3. If `has_block_cooldown`
        - Read 4 bytes as `block_cooldown`
    4. Repeat until `contract_count` matches iterations

### Wildcards

1.  Any module named `"*"` is a wildcard
    1.  A wildcard module enables all modules
    2.  Constraints applied to a wildcard module such as `methods` and `block_cooldown` apply
    3.  Named module constraints take priority over wildcard module constraints
2.  Any contract named `"*"` is a wildcard
    1.  A wildcard contract enables all contracts
    2.  Constraints applied to a wildcard contract such as `block_cooldown` apply
    3.  Named contract constraints take priority over wildcard contract constraints
3.  Any method named `"*"` is a wildcard
    1.  A wildcard method enables all methods for the module or contract it is listed under
    2.  Constraints applied to a wildcard method such as `block_cooldown` and `pact` apply
    3.  Named method constraists take priority over wildcard method constraints
