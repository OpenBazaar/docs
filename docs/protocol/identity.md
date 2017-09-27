# Identity

OpenBazaar nodes can be identified with hashes (unique identifiers) that are machine readable but not particularly human readable. OpenBazaar uses ed25519 elliptic identity keys; your peer ID is the multihash encoded SHA256 hash of the public key. OpenBazaar currently generates keys deterministically based on the initial mnemonic which can be used to regenerate the OpenBazaar and Bitcoin identities in case of database loss (see backups).

It is planned that human readable names mapping to these unique identifers will be implemented in the near future using Blockstack's subdomain system. More details on this system are [here](https://github.com/blockstack/blockstack-core/blob/master/docs/subdomains.md). This feature won't be ready for the early versions of OpenBazaar 2.0.