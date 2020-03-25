# Revocation

**Patent Pending**

Revocation requires that a verifiable revocation notice for a given Doughnut be posted somewhere
accessible to all Doughnut revokers and verifiers.

Because Doughnuts support decentralisation it makes sense for a Doughnut to tell verifiers
how and where to revoke it, and where to check its revocation status.
This removes the need for centralised revocation services.

What follows are specifications for decentralised Doughnut revocation methods.
A Doughnut version may specify a specific revocation method, or allow the issuer to
make a choice of revocation method.

Different strategies for achieving revocation are explored in the [`Strategies`](#strategies) section below.





# Methods



## 0
Method 0 specifies a basic HTTP revocation interface with issuer/holder revocation.


### Hashing
The hashes specified below are BLAKE2 unless otherwise stated.


### Doughnut
The Doughnut will store a URI that points to a valid Method 0 Revocator.

```
    ...
    revocation_uri: String
    ...
```


### Revocator API

A Method 0 Revocator is expected to provide the following HTTP(S) interface.

#### `POST /revoke`

An endpoint for revoking a Doughnut.

##### Body
```
{
    doughnut: String,
    revoker: String,
    method: Number,
    signature: String,
}
```
##### Parameters
* `doughnut`:
    1. Hex encode the Doughnut binary
* `revoker`:
    1. Hex encode the revoker public address
        * The revoker must either be the Doughnut issuer or a holder of it.
* `method`:
    1. Define a number corresponding to a valid [signature method](../format.md#signature), and
    matching the signing method of `signature`
* `signature`:
    1. Set a variable `DOUGHNUT_HASH` as the BLAKE2 hash of the Doughnut binary
    1. Set a variable `REVOCATION_WORD` as the ASCII byte encoding of the string `"revoke"`
        * "revoke" in ASCII byte encoding is `114, 101, 118, 111, 107, 101`
    1. Set the variable `MESSAGE` as `<DOUGHNUT_HASH><REVOCATION_WORD>`, concatinating both byte arrays
    1. Set the variable `SIGNATURE` as the signature of `MESSAGE`, signed using the signing method
    specified in `method`
    1. Hex encode the signature binary

##### Description
The `POST /revoke` endpoint registers the revocation of a Doughnut.

##### Steps
1. The revocator **MUST** calculate the BLAKE2 hash of the Doughnut.
1. The revocator **MUST** check to see whether the Doughnut hash has been stored already.
    * If it has been stored already, return a 200 HTTP status code (OK).
1. The revocator **MUST** verify that the Doughnut can be decoded.
    * If the revocator cannot decode the Doughnut, return a 400 HTTP status code (Bad Request).
1. The revocator **MAY** check whether a domain or domains that it knows about can also be decoded,
    and make additional revocation permission checks on these where applicable.
    * If these checks fail, return any one of:
        * Failure to decode / other codec failure: 400 HTTP status code (Bad Request).
        * Permissions failure: 403 HTTP status code (Forbidden).
        * Expiry: 410 HTTP status code (Gone).
1. The revocator **MUST** verify that the Doughnut has not expired
    * If it has, return a 410 HTTP status code (Gone).
1. The revocator **MUST** verify that the `revoker` is either a holder or an issuer of the Doughnut.
    * If they are not, return a 403 HTTP status code (Forbidden).
1. The revocator **MUST** verify that the signature field is correct.
    * Reconstruct `MESSAGE` by following the parameter encoding steps taken by the sender,
        (listed in `Parameters` above) and verify that `revoker` signed `MESSAGE` to create the signature.
    * The verify method used must conform to the signing method defined in `method`.
    * If this check fails, return a 403 HTTP status code (Forbidden).
1. The revocator **MUST** store the BLAKE2 Doughnut hash.
    * The revocator remembers whether the Doughnut was revoked by storing the hash.
1. The revocator **MAY** also store the expiry time alongside the hash.
    * Recommendeded.
    * This allows the revocator to prune expired Doughnuts that no longer need their
      revocation notice.
1. Return a 200 HTTP status code (OK).

---

#### `GET /check?[hash=<HASH> | doughnut=<DOUGHNUT>]`
##### Parameters
* The revocator **MUST** accept one of either `DOUGHNUT` or `HASH`, but not both.
  * `HASH`:
    A hex encoded BLAKE2 hash of the Doughnut to be checked for revocation.
  * `DOUGHNUT`:
    A hex encoded Doughnut to be checked for revocation.
##### Description

The `GET /check?hash` endpoint checks whether a Doughnut has been revoked.

Calling with `HASH` provides a slightly faster and cheaper revocation check,
with the requirement falling to the client to compute the BLAKE2 hash before sending.

Calling with `DOUGHNUT` provides a more convenient revocation check, with the hash
calculation falling to the revocator, and a larger and slightly more time consuming transmission.

Note that if `DOUGHNUT` is too large to fit into a query parameter, you may use the `POST /check`
interface.

##### Steps
1. If `DOUGHNUT` was given:
    * The revocator **MUST** calculate the BLAKE2 hash of the Doughnut.
2. The revocator **MUST** check whether the hash has been stored by the `POST /revoke` endpoint
  previously.
  * If it has not, return a 404 HTTP status code (Not Found).
  * If it has, return a 200 HTTP status code (OK).

---

#### `POST /check`
##### Body
```
{
    [hash: HASH | doughnut: DOUGHNUT]
}
```
The revocator **MUST** accept one of either `doughnut` or `hash`, but not both.
##### Description
Identical to the `GET /check` endpoint.




---




# Strategies

There are several revocation strategies that may be employed, each optimised for different things.

Each revocation method may support one or more of these strategies.



## Hash/ID revocation

With this strategy either a hash is derived from or a unique ID placed inside of a Doughnut.
This identifier can then be used to revoke the Doughnut.

This approach relies on storing a black/whitelist of hashes/IDs within a revocation service.
Revocation services may be either centralised or decentralised.

### Pros:

* Revokes a specific Doughnut, providing targeted revocation control.

### Cons:

* Storage cost scales in step with number of revoked Doughnuts.
* Some added complexity in culling Doughnuts from state once the expiry has passed.
* If the issuer is to revoke a Doughynut, they must generate the ID or calculate
    the hash and remember it until it is needed.



## Nonce revocation

A rolling nonce can be used, where every address has a revocation nonce associated with it, and only certificates with a matching nonce are valid.

An issuer may increment their nonce at any time, effectively revoking all currently issued certificates
(assuming they all used the current revocation nonce - it's possible that doing this could validate a previously invalid certificate with an n+1 nonce and a valid expiry).

This allows for a simple "log out all apps" revocation service.

### Pros:
* Smallest conceivable storage footprint.
* Does not require storing hashes/IDs.
* Does not require an issuer to remember or discover which Doughnuts are in circulation.
* Does not encourage active pruning of expired revocations.

### Cons:
* Cannot revoke a single Doughnut, making it less ideal for normal use.



## Issuer-holder revocation

In some scenarios holder key pairs are only intended to live for a short period before being discarded.
In these scenarios, this revocation strategy is optimal.

This strategy revokes all Doughnuts issued by a given issuer to a given holder, forever.

This approach relies on storing a black/whitelist of issuer-holder address pairs.

### Pros:
* Allows an address to have their Doughnuts revoked without impacting other Doughnut holders.
* issuer may discover the address of a malicious party via auditing and revoke access without needing to know which certificates the attacker has access to.

### Cons:
* Complexity in culling old blacklisted Doughnuts from state.
    * Requires a non-optimal user experience.
* Storage cost is greater than nonce.


## Combined revocation

Combinations of the above are possible. For instance:

* Issuer-holder nonce revocation.
* A Doughnut payload encoding that supports either Hash/Id Revocation and/or nonce revocation.

These strategies may be used as needed, based on the scenario being optimised for.
