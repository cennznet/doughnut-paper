# Doughnut Format Specification

**Patent Pending**

This document outlines the format of Doughnut certificates. These are optimised for space efficiency, as we want the impact on-chain to be minimal.

## Encoding

A doughnut is little-endian binary encoded.

Both bit and byte order are little-endian (eg, `0b10000000_00000000 = 1_u16`).

## Structure

A doughnut contains:
* a `VERSION` definition
* the version specific certificate `PAYLOAD` encoding of the doughnut
* the certificate `SIGNATURE`.

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

## Verification
To verify a doughnut:

1. Verify the issuer's signature
    1. Separate `<VERSION><PAYLOAD>` from `<SIGNATURE>`
    2. Extract the issuer public key from `PAYLOAD` as `ISSUER`
        * How this is done depends on the payload version
    3. Verify `SIGNATURE` is valid given a message of `<VERSION><PAYLOAD>` and a signer of `ISSUER`
        * How this is done depends on the signing method
2. Verify the user's account is the `HOLDER`
    1. Extract the holder public key from `PAYLOAD` as `HOLDER`
        * How this is done depends on the payload version
    2. Verify the account attempting to use the doughnut matches the `HOLDER`
3. Verify the payload
    * How this is done depends on the payload version
4. Verify the length of the doughnut does not exceed what is inferred by the `PAYLOAD`
    * The expected length of the doughnut can be inferred by the `PAYLOAD` of the doughnut
    * Reject doughnuts which exceed their required length
    * How this is done depends on the payload version

## Payload

### 0

* 1 byte
    * 1 bit
        * Indicates NotBefore timestamp inclusion when 1
    * 2-8 bits
        * Permission domain count
        * Unsigned integer
        * To be incremented by 1 when read, shifting range from 0-127 to 1-128
* 32 bytes
    * Issuer public key
* 32 bytes
    * Holder public key
* 4 bytes
    * Expiry, UNIX timestamp
    * Unsigned integer
* If NotBefore
    * 4 bytes
        * NotBefore, UNIX timestamp
        * Unsigned integer
* List
    * Permission domain payload lengths
    * 16 bytes
        * Permission domain ID (E.g. "cennznet", a UUID, or a unique number)
    * 2 bytes
        * Length of permission domain payload in bytes
        * Unsigned integer
* List
    * X bytes
        * Permission domain payload

#### Payload Verification

To verify a v0 payload:
1. If set, NotBefore must be less than a current unix timestamp.
2. Expiry must be greater than a current unix timestamp.
3. The domain list must not contain duplicate domain names.

#### Notes
##### Permission domain list ordering
The permission domain lists must hold the same domain at the same index.

E.g. the cennznet permission domain must have the same index in both lists.

##### Restrictions
* Maximum of 128 permission domains per doughnut
* Public keys no larger than 256bit

##### Retrieving a permission domain payload
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

### Not Before Zero / Missing
A `0` or unspecified (`null`, `None`, etc.) 'not before' value should __not__ be encoded into the doughnut payload
- The 'not before' bit should be set off (`0`)
- When decoding a 'not before' bit set off (`0`). The 'not before' value should be decoded/presented as `0`

# Signature

Signature of `<VERSION><PAYLOAD>`, signed by the issuer.

### 0
* Schnorrkel signature
    * Separation:
        * The last 64 bytes

### 1
* Ed25519 signature
    * Separation:
        * The last 64 bytes

# Length

Doughnut compacts have no leading length value. This allows us to:

* exploit formats and transports that naturally express termination (E.g. strings, QR codes)
* save space

If a transport is not naturally terminating (E.g. embedded in binary streams), please prefix doughnuts with a relevant length header or other terminator.

