## Differences between `did:sov`, `did:indy` and `did:indy:besu`

Early instances of Indy Networks used the `did:sov` DID Method. The following summarizes the differences between that `did:sov` and `did:indy`.

- A `did:indy` DID includes a namespace component that enables resolving a DID to a specific instance of an Indy network (e.g. Sovrin, IDUnion, etc.).
- Identifiers for Indy ledger objects other than [[ref: NYM]]s are adjusted to contain a namespace component.
- `did:indy` DID validation is determined by original NYM transaction `version`, as described in the [DID Creation](#nym-transaction-version) section. These restrictions MUST be enforced by the ledger.
- `did:indy:besu` DID validation placed in [[def: Indy Besu VDR]].
- For original indy the specification includes rules for transforming a [[ref: NYM]] into a DIDDoc that meets the DID Core Specification.
    - An optional [[ref: NYM]] data item allows entities to extend the DIDDoc returned from a [[ref: NYM]] in arbitrary ways.
    - The controller can decide whether the DIDDoc will be JSON or JSON-LD.
    - Before writing a [[ref: NYM]] to the ledger, the [[ref: NYM]] content is verified to ensure that transforming the [[ref: NYM]] to a DIDDoc produces a valid JSON and may include DIDDoc validity checking.
    - The transformation of a read [[ref: NYM]] to a DIDDoc is left to the client of an Indy ledger.
- For `did:indy:besu` transforming of ledger data into a DIDDoc is not needed. Objects are stored already in the [Decentralized Identifiers (DIDs) Core specification](https://https://www.w3.org/TR/did-core/) compatible format.
- A convention for storing Indy network instance config ("genesis") files in a Hyperledger Indy project GitHub repository ("indy-did-networks") is introduced.

### Compatibility `did:indy:besu` with other identifiers

The idea is using of a basic mapping between other DIDs identifiers and Ethereum accounts instead of introducing a new DID method.

* There is a mapping structure `legacyIdentifier => ethereumAccount` for storing `did:indy` and `did:sov` DID identifiers to the corresponding account address
    * Note, that user must pass signature over identifier to prove ownership
* On migration, DID owners willing to preserve resolving of legacy formatted DIDs and id's must add mapping between legacy identifier and ethereum account defining
    * DID identifier itself
    * Associated public key
    * Ed25519 signature owner identifier proving ownership
* After migration, clients in order to resolve legacy identifier for DID document:
  * firstly should resolve ethereum account
  * next resolve DID ether document

Detailed description of migration plan is [here.](https://github.com/hyperledger/indy-besu/blob/main/docs/migration/migration.md)