# DIDLink: Providing IPNS functionality in coordination with decentralized IDs

Author: Wes Biggs &lt;wes@tralfamadore.com>

Version: draft-20240210 (update 1)


## Problem


### Challenges with IPNS



1. IPNS using DHT/PubSub presents challenges (particularly in WAN scenarios) because a client cannot know if the IPFS server they are talking to is presenting them with the latest version of an IPNS link. While PubSub enables more rapid dissemination of changes, it still cannot provide a deterministic mechanism for knowing if a version is the latest, and has scalability challenges related to the number of active subscriptions a given node can support. IPNS publishing using DHT is also notoriously slow.
2. IPNS using DNSLink solves the currency problem (for the most part; records are still subject to the trickle-down effect of minimum TTLs) but introduces a reliance on internet infrastructure that is ultimately centralized (ICANN and TLD registrars as gatekeepers). While it is possible to support non-ICANN domains using DNS over HTTPS (DoH), to use these, IPFS code would need to enshrine specific alternative DNS systems. DNSLink also presents a technical or economic barrier for users as they must have control of or access to a DNS zone in order to register and maintain the mapping to a CID, regardless of whether the domain is acquired through ICANN-sanctioned or alternative registries.


## Objective

Introduce a new strategy for IPNS that accomplishes the following:

* Gives readers confidence that they are viewing the latest version of a named entity
* Reduces network and storage requirements for IPFS nodes
* Enables coordination with consensus systems without endorsing a particular system
* Retains the security profile of IPNS
* Provides a straightforward implementation path for IPFS developers and developers using IPFS in their projects.


## Proposal

We introduce an intentionally constructed "DIDLink" strategy, encompassing a hash algorithm, document format and DID service type, with specified semantics.
Users can map an ownership proof using a known challenge (nonce) to a deterministic multihash value (and thus CID) for files that act as cryptographically signed links for use with IPNS.
IPNS resolution uses the DID protocol to discover the current nonce from the state of an external system.

This approach allows the link update functionality of IPNS to be offloaded to DID providers, including consensus systems.

### Workflow overview

To publish a new link, a user does the following:

1. Generate a keypair. Ensure the private key is retained for future updates. (This is functionally equivalent to the existing `ipfs key gen` command in Kubo.)
2. Generate a nonce value. This could be a simple counter, a timestamp, or a random value. The important thing is that it is never reused for a given public key.
3. Use the private key of the keypair to create a digital signature of the concatenation  of the public key and nonce.
4. Create a DIDLink file containing the public key, nonce, signature, and link target CID.
5. Generate a CID for the DIDLink file using the did-link-sha2-256 multihash.
6. Publish the DIDLink file to IPFS.
7. Inform their DID provider of the public key and nonce value in use and define a DID with a URI fragment that uniquely identifies the link. This is the permanent IPNS data value for the link associated with the key.

A rough CLI version of the above, with a hypothetical command that interacts with the DID provider, looks like the following:

```
% ipfs key gen mykey
% ipfs name publish --key=mykey --nonce=1 /ipfs/QmatmE9msSfkKxoffpHwNLNKgwZG8eT9Bud6YoPab52vpy
Published to Qxyz with nonce 1
% did-provider link create did:somemethod:123#profile --key=mykey --nonce=1
% ipfs name resolve did:somemethod:123#profile
/ipfs/QmatmE9msSfkKxoffpHwNLNKgwZG8eT9Bud6YoPab52vpy
```

Updating the link target follows the same steps except that the existing keypair is used. Once the DID provider record is updated, the old DIDLink file can be removed (or unpinned) from IPFS (subject to TTL-based caching).

Removing the link is an interaction with the DID provider.

### DIDLink file format and hash derivation

Given an input file:

```
{
  "publicKeyMultibase": "{multibaseMultikeyString}",
  "nonce": "{multibaseString}",
  "signature": "{multibaseString}",
  "target": "{IPFS_or_IPNS_path}"
}
```

We define the _DIDLink_ hash function as follows (pseudocode):

```
let { publicKeyMultibase, nonce, signature } = parse_json(input);
let message = to_binary_string(publicKeyMultibase) +
  to_binary_string(nonce);
if (verify_signature(publicKeyMultibase, message, signature)) {
  return sha256(message);
} else {
  // Invalid input; revert to sha256
  return sha256(input);
}
```

In other words, if the signature proof is valid (proving that the user who created the file controls the public key), the resulting CID will be based on the SHA-256 hash of the public key and the specified nonce.

This file is then added to IPFS in the same manner as any other file; it can be pinned, unpinned, distributed, etc.

Note that _DIDLink_ is intentionally _not_ collision resistant. If the public key controller creates and signs two files with the same nonce and different target values, _DIDLink_ will generate the same CID for both. Hence, in order to avoid conflicts, the same nonce value should never be reused with the same key. However, no two files with distinct `{ publicKeyMultibase, nonce }` values will ever collide.

_DIDLink_ should be assigned a multihash prefix in [https://github.com/multiformats/multicodec/blob/master/table.csv](https://github.com/multiformats/multicodec/blob/master/table.csv). (Technically the hash function has nothing to do with DIDs, so perhaps a different name is warranted.)

### DID document

In order for the published _DIDLink _file to be found on the network, we need a way to allow consumers of the data to determine the current CID. We envision the state management of the nonce value to be the responsibility of the user interacting with a consensus system that allows them to control data associated with a DID. The DID resolver for that DID must be capable of generating a DID document that has a service list with entries that (1) indicate where the CID where the DIDLink file associated with the key and nonce can be found, and (2) prove that the controller of the key used for the DIDLink file also has control of the IPNS DID (with fragment).

(1) is accomplished using the `serviceEndpoint` value, which should link to the IPFS CID of the DIDLink file.

(2) is accomplished by generating a cryptographic signature using the key as applied to the fully qualified DID with fragment used as the service `id` (in other words, the value used with IPNS).

```
{
  // within DID Document
  "service": [
    {
      "id":"did:dsnp:123#profile",
      "type": "DIDLink",
      "serviceEndpoint": "ipfs://{CID_of_DIDLink_file}",
      "ttl": 300,
      "verificationMethod": "did:dsnp:123#z6MkvgdbwobNffEBuGCjtTMtFPGZE9jKDf9DhyTatMtRqXEd",
      "proofValue": "z5kQT4XJDq17hcs3S9yytQbLz56iHhNMR4ZrAQov6okv3r5Diu6oVfuBw7GcFJ5W4c5cn4MPTj4pMCok7zvHfQumD"
    }
  ]
}
```

This example indicates that a _DIDLink_ file for the "profile" data label can be found at the URL indicated by the service endpoint value (in this case, an IPFS CID).

An alternative approach would be to have the service entry explicitly return the public key and nonce (without performing the hash operation). However, this step is easily performed by the DID resolver to produce a usable `serviceEndpoint` value. That said, a DID resolver could optionally return these values for clarity.

This DID service definition should ideally be registered and published with [https://www.w3.org/TR/did-spec-registries/#service-types](https://www.w3.org/TR/did-spec-registries/#service-types).

### DID resolver implementation

There are multiple potential strategies for a DID provider to use in order to generate the `serviceEndpoint` value associated with the public key. A straightforward and space-efficient approach would be to store only the nonce associated with the public key as part of the state of a consensus system, and have the DID resolver code perform the hashing. However, it is also possible to implement a simple key-value store mapping the DID fragment identifier to the DIDLink CID directly.

### IPNS resolution

When an IPNS resolver encounters an IPNS URI that is a DID (with a fragment), it does the following:

1. Retrieve the DID document associated with the DID using the registered DID method resolver.
2. Examine the service list within the DID document. Find the first service entry with type "DIDLink" and a matching id and determine the serviceEndpoint.
3. Check the ownership proof for the service by verifying the signature over the service `id` with the public key indicated.
4. Retrieve the CID indicated by the serviceEndpoint value from the IPFS network.
5. Ensure that the public key used in the retrieved file is the same as that used with the service proof.
6. Verify the signature field within the retrieved file.
7. Use the target property within the retrieved file to respond to the IPNS lookup.

Note that the "ttl" field can be used to implement a caching strategy to limit the number of lookups required for both the DID and the DIDLink CID.

## Further considerations

### IPFS implementation requirements

An IPFS implementation needs to be capable of generating a CID using a _DIDLink_ multihash; thus, it must include a hasher as described in this document; perhaps named `did-link-sha2-256.`

Similarly, an IPFS server must be able to regenerate and verify the hash.

### IPNS implementation requirements

The following requirements apply to the IPNS resolution module:

* An IPNS resolver implementation must be updated to handle IPNS values matching `did:*` using this approach. These can be unambiguously differentiated from IPNS keys and DNSLink domains.
* The IPNS resolver must be network-connected (this is already the case for DNSLink, so should not present difficulty).
* The resolver must be configured and instrumented with the ability to resolve a given DID method and DID URI. The allowed or enabled DID methods do not need to be specified in this protocol definition, but are the domain of the implementation (e.g. Kubo, Helia). We propose an initial implementation allowing out-of-the-box DID resolution of `did:dsnp:*` DIDs using DSNP over Frequency. However, similar support could easily be added for Bluesky's `did:plc` method, if Bluesky chooses to implement this proposal.

## Design questions

_Why not just have the DID document reference the target CID directly?_

This would require the consensus system to store the full target CID. The use of a nonce (which can be as short as a single byte) is designed to minimize consensus system storage requirements. Additionally, static analysis of publicly visible data in the consensus system cannot be used to determine the target CID value, which provides a layer of separation and allows for secure usage with private or access-controlled IPFS networks.

_What happens in the event of "hash collisions"?_

Firstly, this is only possible if the key holder signs multiple documents with differing targets but the same public key and nonce. If that happened, you could not know which version you would get from a given IPFS node. In the worst case, a sophisticated attacker with a high level of network knowledge could potentially control which response was provided from each server. As an attack vector, this seems limited. In the accidental case, it can be easily corrected by publishing a file with a new nonce.

_Who's pinning all these files?_

Anyone can pin them (but no one has to). This is an improvement over DHT/PubSub which does not allow for pinning of mappings today (which is a disincentive for a disinterested third party to participate). By making the links normal CID-based files, there is a level playing field for storage providers.

_Does DNSLink go away?_

It doesn't need to. There are valid use cases where it is important that IPFS data be tied to the ICANN trust model (and the human readability aspect is nice).

_Does PubSub/DHT IPNS go away?_

Those could be deprecated over time. Or they may continue to be appropriate in environments (IPFS cluster, LANs) where coherence can be well maintained.

_When can I delete (unpin) old versions?_

To ensure 100% availability, old versions can be removed from the network after the configured TTL seconds have elapsed since the DID document was updated.

_What are some practical applications of this approach?_

* In DSNP over Frequency, this would allow easy access to per-user data like an Activity Content Profile without the need for a separate content indexer. In general, it could enable any user data item to be moved off-chain.
* A WebID profile (used in Solid) could be stored in IPFS and referenced from a DID, enabling a consensus system-based discovery mechanism for public profiles.
* Filecoin or other decentralized storage providers could implement their own DID resolvers to add IPNS functionality to their services.

_Can the existing IPNS Record format be leveraged?_ (cf. [https://specs.ipfs.tech/ipns/ipns-record/#record-fields](https://specs.ipfs.tech/ipns/ipns-record/#record-fields))

Possibly. The "Sequence" field would either be ignored (and the nonce value would become a data extension within the extensible data field), or the "Sequence" field could be used for the nonce value. Serializing this way would be more space efficient (but less human readable) than JSON. I intentionally did not use this in the example to avoid confusion with IPNS DHT/PubSub routing records. (That is, we could leverage the compact data serialization format, but it would not be used in the same way.)

## Document history

### draft-20240210 (update 1)

Added `proof` object to DID `service` definition to link control of DID document to control of DIDLink file, and corresponding verification steps to the algorithm.

Removed obsolete note about DID controllers not needing to match.
