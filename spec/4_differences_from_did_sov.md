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

The idea is using of a basic mapping between other DIDs identifiers and ethereum accounts instead of introducing a new DID method.

* Use `LegacyMappingRegistry` smart contract which holds mapping of legacy identifiers to ethereum accounts/new ids:
    ```
    contract LegacyMappingRegistry {
        // Mapping storing indy/sov DID identifiers to the corresponding account address
        mapping(bytes16 legacyIdentifier => address account) public didMapping;
  
        // Mapping storing indy/sov formatted identifiers of schema/credential-definition to the corresponding new form
        mapping(string legacyId => string newId) public resourceMapping;
    
        function createDidMapping(
            address identity,
            string calldata identifier,
            bytes32 ed25519Key,
            bytes calldata ed25519Signature
        )
            // check signature
            // check legacyDid is derived from key
            didMapping[identifier] = msg.sender;
        }
    
        function createDidMappingSigned(
            address identity,
            uint8 sigV,
            bytes32 sigR,
            bytes32 sigS,
            string calldata identifier,
            bytes32 ed25519Key,
            bytes calldata ed25519Signature
        )
            // check signatures
            didMapping[identifier] = identity;
        }
    
        // resolve mapping done through `didMapping(bytes16 identifier)` function available after contract compilation
    
        function createResourceMapping(
            address identity,
            string calldata legacyIssuerIdentifier,
            string calldata legacyIdentifier,
            string calldata newIdentifier
        )
            // fetch issuer did from legacy schema/credDef id 
            // check issuer did is derived from key
            // check msg.sender is owner of issuer did
            // check identity is owner of schema / cred def
            // check signature
            resourceMapping[legacyIdentifier] = newIdentifier;
        }
    
        function createResourceMappingSigned(
            address identity,
            uint8 sigV,
            bytes32 sigR,
            bytes32 sigS,
            string calldata legacyIssuerIdentifier,
            string calldata legacyIdentifier,
            string calldata newIdentifier
        )
            // fetch issuer did from legacy schema/credDef id 
            // check issuer did is derived from key
            // check identity is owner of issuer did
            // check identity is owner of schema / cred def
            // check signatures
            resourceMapping[legacyIdentifier] = newIdentifier;
        }
    
        // resolve mapping done through `resourceMapping(string legacyIdentifier)` function available after contract compilation
    }
    ```
  * Note, that user must pass signature over identifier to prove ownership
* On migration, DID owners willing to preserve resolving of legacy formatted DIDs and id's must do:
  * add mapping between legacy
    identifier and ethereum account identifier by
    executing `LegacyMappingRegistry.createDidMapping(...)` method where he must pass:
    * DID identifier itself
    * Associated public key
    * Ed25519 signature owner identifier proving ownership
      * Signature must be done over `legacyDid` bytes
  * add mapping between legacy schema/credDef id's and new ones executing `LegacyMappingRegistry.createResourceMapping()` method he must pass:
    * Legacy DID
    * Legacy schema/credDef id
    * New id of corresponding schema/credDef
* After migration, clients in order to resolve legacy identifiers:
  * for DID document firstly must resolve ethereum account
    using `LegacyMappingRegistry.didMapping(legacyIdentifier)`, and next resolve DID ether document as it described in the
    corresponding specification.
  * for Schema/Credential Definition firstly must resolve new identifier
    using `LegacyMappingRegistry.resourceMapping(legacyIdentifier)`, and next resolve Schema/Credential Definition as it described in the
    corresponding specification.