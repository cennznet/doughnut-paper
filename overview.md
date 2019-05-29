# Doughnut Overview

Blockchain technology has need of powerful, seamless authentication and access control.

This need leads from the success of centralised services in providing excellent user sign-in experiences, most notably in the widespread adoption of Single Sign-On (SSO) models.

The failure of DApps to achieve mainstream adoption persists while the authentication experience lags so far behind centralised applications.

Given the dangers of private key transmission and the high consumer demand for good UX and ideal security, some new system other than signing devices and identity contracts is needed.

`Doughnut`s solve this problem. The concept is simple yet powerful, enabling a classic "centralised" user experience on a trustless, decentralised technologies.

### Naming

Why "`Doughnut`"? It's a take on the "Cookie" and "[Macaroon](https://ai.google/research/pubs/pub41892)" naming convention. `Doughnut`s are decentralised 🍩.

## Goals

-   Permissions **MUST**  be given rather than assumed
-   Permissions **MUST** be enforceable for all addresses and contracts
-   Permissions **MUST** protect users as much as possible from malicious applications
-   Doughnuts  **MUST**  be cheap to generate, store, use, transmit and modify
-   Doughnuts **MUST**  be optional, and not modify default blockchain behaviour when not in use
-   Doughnuts  **MUST** be non-viable if stolen
-   Doughnuts  **MUST**  be revokable by the issuer or holder at any time
-   Doughnuts **SHOULD** optionally support off-chain service authentication

## Existing solutions:

Blockchain access control is currently focussed on two solutions:

-   Transactions submitted by applications are signed by a signing device
-   Transactions signed by an application are "executed" by a permissioned identity smart contract

These approaches may be used together in different ways if necessary, but the end result is largely the same.
  
|  |Pros|Cons|
|--|--|--|
| Signing device |<ul><li>Applications don't need to hold keypairs</li><li>No fee associated with sharing authority between apps</li></ul>|<ul><li>User must approve every signature manually<ul><li>User must check every transaction for malicious activity</li><li>No way to prevent an unexpected internal transaction call</li></ul></li><li>User must have access to the device while using applications</li></ul>|
| <p>Identity Contract</p><p></p><p><em>(<a href="https://github.com/ethereum/EIPs/issues/725">ERC725</a>, <a href="https://github.com/ethereum/EIPs/issues/1056">ERC1056</a>, etc)</em></p> |<ul><li>Supports multiple applications controlling one address without another application or device signing</li><li>May support&nbsp;granular permissions</li><li>May support TTL and revocation</li><li>May&nbsp;authenticate for off-chain services</li><li>May support one-use access control</li></ul>|<ul><li>No way to prevent an unexpected internal transaction call</li><li>Must pay regular network gas fees</li><li>Gas fee associated with managing authority between apps</li><li>Gas fee involved in setting up some contracts (does not apply to ERC1056 or other similar registry implementations)</li><li>Applications cannot rely on one standard, shared access control specification</li></ul>|
| `Doughnut` |<ul><li>Limited permissions by default</li><li>Enforces permissions for the full internal transaction tree</li><li>Supports TTL and revocation</li><li>Cheap off-chain generation, modification, transmission and storage</li><li>Full support for granular blockchain permissions</li><li>May be used to flag suspicious app activity to users</li><li>May&nbsp;authenticate for off-chain services</li><li>May&nbsp;support third-party authenticators (Similar to Macaroons)</li><li>May support one-use access control</li><li>And more</li></ul>|<ul><li>Requires some modification of the blockchain proof validator and VM</li><li><code>Doughnut</code>s&nbsp;will add to transaction sizes<ul><li>Possible to offset cost in-chain&nbsp;</li><li>Possible to reduce size w/ transaction-specific <code>doughnut</code>s and optimised storage</li></ul></li></ul>|

## What is a `Doughnut`?

A `Doughnut`  is an attenuated permissions `certificate`  issued off-chain from one private key (the  `issuer`), usually to another private key (the `holder`).

On CENNZnet, `Doughnut`s will allow:
* Identity delegation: the `holder` acts as the `issuer,` with a subset of permissions  
* Fee delegation: the `issuer` pays for the fees of the `holder` when specific resources are accessed
* Contract access control: the `holder` may access special features in a smart contract or runtime module
* and more

`Doughnut`s are generated, stored and transmitted off-chain, and may be verified anywhere. This makes them cheap to manage and use.

If a `holder` is specified, the `Doughnut` is only valid when paired with a  `signature` from the  `holder`, helping to mitigate cases where the `certificate`  is stolen.

To recap, `Doughnut`s are cryptographically verifiable access tokens, carrying permissions, generated off-chain.

## Format

A doughnut is binary encoded. It contains a version definition, the version specific encoding of the doughnut, and the certificate signature.

```
<VERSION><PAYLOAD><SIGNATURE>
```

Read more about versions, payload encodings, and signature methods in the doughnut  [format specification](./format.md).


#### Payload:
These fields are common to doughnut certificates.

|Path|Description|Format|Required|
|--|--|--|--|
|`expires`|Expiry UNIX time (TTL)|Number|*|
|`not_before`|UNIX time indicating when the `doughnut`  becomes viable|Number||
|`holder`|`Holder`'s public key|String + Hex|*|
|`issuer`|`Issuer`'s public key|String + Hex|*|
|`permissions`|Permissions object||*|
|`permissions.[domain]`|Permission domains|Any|*|


##### Payload example:
```
{
	"expires": 1547500527,
	"not_before": 1547327727,
	"holder": "97254c471e56eefb1ec988ad0a98150f327e3b791bb26799413b6cf2f6d9b19f13",
	"issuer": "5fe33be52021fbe7056d7e9ceb0d12dfc938193fe3b54c3ee05021f639f0d95df2",
	"permissions": {
		"cennznet": {
			"GenericAsset": {
				"transfer": {}
			},
			"d0a98...56d7e9c": {
				"bet": {}
			}
		}
	}
}
```


## `Doughnut` lifecycle

1.  A  `doughnut` is requested by an application or service from an `issuer`.
	 The requester will become the `holder`  of the  `doughnut`. 
    -   A  `doughnut` request **MUST**  specify:
        -   Requested permissions object (non-empty)
        -   The `holder`'s public key
    -   A  `doughnut` request  **MAY** specify:
        -   The target `issuer`'s public address
            -   If not, this is chosen by the `issuer`  application
    -   The `holder` **MAY**  request many  `doughnut`s, each for a specific transaction
        -   Allows smaller transaction sizes
2.  If approved, the `issuer`  generates the `doughnut`
3.  The `issuer`  transmits the `doughnut`  back to the `holder`
4.  `holder`  may now use the `doughnut`  as the permissions allow
5.  CENNZnet validates the  `doughnut` as a new kind of proof
6.  The `holder`  or the `issuer`  may revoke the `doughnut`  at any time before `expires`
7.  Once `expires`  has passed, the  `doughnut` becomes invalid

#### Executing with a `doughnut`  on CENNZnet

1.  The `doughnut`  bearing extrinsic **MUST**  be valid
2.  The `doughnut`  itself **MUST**  be valid
3.  An extrinsic bearing a  `doughnut`  **MUST** contain a valid non-empty `cennznet`  permissions domain
4.  The  `doughnut.certificate.permissions.cennznet` `permission set` is to be extracted
5.  The extrinsic's specified target and arguments, and all consequent internal transactions **MUST**  be a subset of the `permission set`

