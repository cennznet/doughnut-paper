# CENNZnet Doughnut Permission Domain Specification

**Patent Pending**

This document outlines the format of the CENNZnet Doughnut Permission Domain.

## Encoding

A CENNZnut represents values as little-endian.

## Structure

A CENNZnut contains:
* a `VERSION` definition
* a `PERMISSIONS` definition

```
<VERSION><PERMISSIONS>
```

# Version

* Represented as a 16-bit value with the following structure:
    * bits 0..9
        * Version number
        * Unsigned 10-bit integer
    * bits 10..15
        * Reserved/unused

# Permissions
## v0

**Note:** *Subject to change*

* 1 byte
    * `module_count`
    * Unsigned 8-bit integer
    * Increment by 1 when read (range 1 - 256)
* Modules list
    * 1 byte
        * bit 0
            * `has_block_cooldown` flag
        * bits 1..7
            * `method_count`
            * Unsigned 7-bit integer
            * Increment by 1 when read (range 1 - 128)
    * 32 bytes
        * `module_name`
        * String
    * If `has_block_cooldown`
        * 4 bytes
            * `block_cooldown`
            * Unsigned 32-bit integer
    * Methods list
        * 1 byte
            * bit 0
                * `has_block_cooldown` flag
            * bit 1
                * `has_pact` flag
            * bit 2..7
                * reserved/unused
        * 32 bytes
            * `method_name`
            * String
        * If `block_cooldown` is set
            * 4 bytes
                * `block_cooldown`
                * Unsigned 32-bit integer
        * If `has_pact` is set
            * 1 byte
                * `pact_length`
                * Unsigned 8-bit integer
                * Increment by 1 when read (range 1 - 256)
            * `pact_length` bytes
                * `pact`
* 1 byte
    * `contract_count` = value
* Contracts list
    * 1 byte
        * bit 0
            * `has_block_cooldown` flag
        * bits 1..7
            * Reserved
    * 32 bytes
        * `contract_address`
    * If `has_block_cooldown`
        * 4 bytes
            * `block_cooldown`
            * Unsigned 32-bit integer

### Decode Steps

1. Read the first byte, as `module_count` and increment it by 1
2. Begin reading module list items
    1. Read the first byte
        - Read bit 0 as `has_block_cooldown` Boolean
        - Read bits 1..7 as `method_count` and increment it by 1
    2. Read 32 bytes as `module_name`
    3. If `has_block_cooldown`
        - Read 4 bytes as `block_cooldown`
    4. Begin reading methods list items
        1. Read the first byte
            - Read bit 0 as `has_block_cooldown` Boolean
            - Read bit 1 as `has_pact` Boolean
            - Ignore remaining bits
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
        - Read the bit 0 as `has_block_cooldown` Boolean
    2. Read 32 bytes as `contract_address`
    3. If `has_block_cooldown`
        - Read 4 bytes as `block_cooldown`
    4. Repeat until `contract_count` matches iterations

### Smart Contracts

Any CENNZnut with smart contract permissions must also assign runtime module permissions for the contracts module:
```json
{
    "modules": {
        "contracts": {
            "call": {}
        }
    },
    "contracts": {
        "d0a98...56d7e9c": {},
        "b2150...2f40a21": {},
    }
}
```

### Wildcards

1.  Any module named `"*"` is a wildcard
    1.  A wildcard module enables all modules
    2.  Constraints applied to a wildcard module such as `methods` and `block_cooldown` apply
    3.  Named module constraints take priority over wildcard module constraints
2.  Any contract with address `[0x00; 32]` is a wildcard
    1.  A wildcard contract enables all contracts
    2.  Constraints applied to a wildcard contract such as `block_cooldown` apply
    3.  Contraints of explicitly identified contracts take priority over contraints of a contract wildcard
3.  Any method named `"*"` is a wildcard
    1.  A wildcard method enables all methods for the module or contract it is listed under
    2.  Constraints applied to a wildcard method such as `block_cooldown` and `pact` apply
    3.  Named method constraints take priority over wildcard method constraints
