# Doughnut Format Specification

This document outlines the format of Doughnut certificates. These are optimised for space efficiency, as we want the impact on-chain to be minimal.

## Structure

A doughnut is binary encoded. It contains a version definition, the version specific encoding of the doughnut, and the certificate signature.

```
<VERSION><PAYLOAD><SIGNATURE>
```

## Version

The version specifies the version of the certificate payload encoding, and the signature method. This tells the reader how to interpret the payload, and how to separate and verify the signature.

* 2 bytes
    * 11 bits: Payload version number
        * Unsigned integer
        * LE
    * 5 bits: Signature method number
        * Unsigned integer
        * LE

## Payload

### 0

* 1 byte
    * 1 bit  

        * Indicates NotBefore timestamp inclusion when 1
    * 2-8 bits  

        * Permission domain count
        * Unsigned integer
        * LE
        * To be incremented by 1 when read, shifting range from 0-127 to 1-128
* 32 bytes

    * Issuer public key
* 32 bytes  

    * Holder public key
* 4 bytes
    * Expiry, UNIX timestamp
    * Unsigned integer
    * LE
* If NotBefore
    * 4 bytes  

        * NotBefore, UNIX timestamp
        * Unsigned integer
        * LE
* List  

    * Permission domain payload lengths
    * 16 bytes
        * Permission domain ID (E.g. "cennznet", a UUID, or a unique number)
    * 2 bytes
        * Length of permission domain payload in bytes
        * Unsigned integer
        * LE
* List
    * X bytes
        * Permission domain payload

##### Notes

Permission domain list ordering:

* The permission domain lists must hold the same domain at the same index. E.g. the cennznet permission domain must have the same index in both lists.

Restrictions:

* Maximum of 128 permission domains per doughnut
* Public keys no larger than 256bit

To retrieving a permission domain payload:

1.  Retrieve the permission domain count from the leading byte, and increment by 1
2.  Step forward to the start of the payload lengths list. If NotBefore is present:
    1.  Look ahead to the 70th payload byte
    2.  Else, look ahead to the 74th byte
3.  Begin recording a `domain_offset` variable
4.  Read 34 bytes
    1.  Check first 16 bytes for match with permissions domain
    2.  If they match, record the 2 byte payload length as `payload_length` and continue
    3.  If they don't match, add the 2 byte payload length to the `domain_offset` value and repeat
    4.  When iterations are greater than the permission domain count, the permission domain does not exist
5.  Step forward to the start of the payloads list
    1.  From the start of the payload lengths list, step forward by `(domain_count * 18)`
6.  Step forward by `domain_offset`, and extract the next `payload_length` bytes

# Signature

Signature of `<VERSION><PAYLOAD>`, signed by the issuer.

### 0

* Schnorrkel signature
    * The last 512 bits

# Length

Doughnut compacts have no leading length value. This allows us to:

* exploit formats and transports that naturally express termination (E.g. strings, QR codes)
* save space

If a transport is not naturally terminating (E.g. embedded in binary streams), please prefix doughnuts with a relevant length header or other terminator.
