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

## Migration from Indy Ledger

This document contains a plan of attack for migration of customers using `indy-node` with `indy-plenum` ledgers to Indy 2.

### Ledger migration

All Issuers need to run migration by itself to move their data (DID Document, Schema, Credential Definition) to Indy Besu ledger.

### Step by step Indy based applications migration flow

This section provides example steps demonstrating the process of migration for applications using Indy ledger to Indy Besu Ledger.
The parties involved into the flow:
* Trustee - publish Issuer DID on the Ledger
  * Write data to Ledger
* Issuer - publish DID, Schema, and Credential Definition on the Ledger + Issue Credential for a Holder
  * Write data to Ledger
* Holder - accept Credential and share Proof
  * Read data from the Ledger
* Verifier - request Proof from a Holder
  * Read data from the Ledger

### Before migration

At this point, all parties acts as usual and use Indy Ledger as a verifiable data registry.

> Consider that these steps happened some time at the past before the decision to migrate from Indy Node to Indy Besu Ledger.

1. Issuer setup (DID Document, Schema, Credential Definition):
  1. All parties create an indy wallet and uses some Indy Ledger client
  2. Trustee publish Issuer's DID to Indy Ledger using NYM transaction
  3. Issuer publish Service Endpoint to Indy Ledger using ATTRIB
  4. Issuer create and publish Credential Schema to Indy Ledger using SCHEMA
  5. Issuer create and publish Credential Definition to Indy Ledger using CLAIM_DEF
2. Credential issuance
  1. Issuer create Credential Offer
  2. Holder accept Credential Offer:
    1. Resolve Schema from Indy Ledger using GET_SCHEMA request
    2. Resolve Credential Definition from Indy Ledger using GET_CLAIM_DEF request
    3. Create Credential Offer
  3. Issuer sign Credential
  4. Holder store Credential
3. Credential verification
  1. Verifier create Proof Request
  2. Holder accept Proof Request and create Proof
  3. Verifier verify Proof
    1. Resolve Schemas from Indy Ledger using GET_SCHEMA request
    2. Resolve Credential Definitions from Indy Ledger using GET_CLAIM_DEF request
    3. Verify Proof

### Migration

At some point company managing (Issuer,Holder,Verifier) decide to migrate from Indy to Indy Besu Ledger.

In order to do that, their Issuer's applications need to publish their data to Indy Besu Ledger.
Issuer need to run migration tool manually (on the machine containing Indy Wallet storing Credential Definitions) which migrate data.

* Issuer:
  * All issuer applications need run migration tool manually (on the machine containing Indy Wallet with Keys and Credential Definitions) in order to move data to Indy Besu Ledger properly. The migration process consist of multiple steps which will be described later.
  * After the data migration, issuer services should issue new credentials using Indy Besu Ledger.
* Holder:
  * Holder applications can keep stored credentials as is. There is no need to run migration for credentials which already stored in the wallet.
  * Holder applications should start using Indy Besu Ledger to resolve Schemas and Credential Definition once Issuer completed migration.
* Verifier:
  * Verifier applications should start using Indy Besu Ledger to resolve Schemas and Credential Definition once Issuer completed migration.
  * Verifier applications should keep using old styled restriction in order to request credentials which were received before the migration.

1. Wallet and Client setup. All applications need to integrate Besu vdr library
2. DID ownership moving to Indy Besu Ledger:
  1. Issuer create Ed25519 key (with seed) in the Besu wallet
  2. Issuer create a new Secp256k1 keypair in Besu wallet
  3. Issuer publish Secp256k1 key to Indy ledger using ATTRIB transaction: `{ "besu": { "key": secp256k1_key } }`
    * Now Besu Secp256k1 key is associated with the Issuer DID which is published on the Indy Ledger.
    * ATTRIB transaction is signed with Ed25519 key. No signature request for `secp256k1_key`.
3. Issuer build DID Document which will include:
  * DID - fully qualified form should be used: `did:besu:network:<did_value>` of DID which was published as NYM transaction to Indy Ledger
  * Two Verification Methods must be included:
    * `Ed25519VerificationKey2018` key published as NYM transaction to Indy Ledger
      * Key must be represented in multibase as base58 form was deprecated
    * `EcdsaSecp256k1VerificationKey2019` key published as ATTRIB transaction to Indy Ledger
      * Key must be represented in multibase
      * This key will be used in future to sign transactions sending to Indy Besu Ledger
        * Transaction signature proves ownership of the key
        * Besu account will be derived from the public key part
  * Two corresponding authentication methods must be included.
  * Service including endpoint which was published as ATTRIB transaction to Indy Ledger
4. Issuer publish DID Document to Indy Besu Ledger:
    ```
     let did_doc = build_did_doc(&issuer.did, &issuer.edkey, &issuer.secpkey, &issuer.service);
     let receipt = DidRegistry::create_did(&client, &did_document).await
    ```
  * Transaction is signed using Secp256k1 key `EcdsaSecp256k1VerificationKey2019`.
    * This key is also included into Did Document associated with DID.
    * Transaction level signature validated by the ledger that proves key ownership.
  * `Ed25519VerificationKey2018` - Indy Besu Ledger will not require signature for proving ownership this key.
    * key just stored as part of DID Document and is not validated
    * potentially, we can add verification through the passing an additional signature
5. Issuer converts Indy styled Schema into new style (anoncreds specification) and publish it to Indy Besu Ledger.
6. Issuer converts Indy styled Credential Definition into new style (anoncreds spec) and publish it to Indy Besu Ledger
  * Migration tool will provide a helper method to convert Credential Definition.
  * Credential Definition ID must include schema seq for achieving backward-compatibility
  * Same time `schemaId` field must contain actual id of schema published to Indy Besu Ledger

### After Migration

All parties switch to use Indy Besu Ledger as verifiable data registry.

Now credential issuance and credential verification flow can run as before but with usage of another vdr library.

1. Credential verification
  1. Verifier create Proof Request
  2. Holder accept Proof Request and create Proof
    1. Holder resolve Schema from Indy Besu Ledger (VDR converts indy schema id representation into Besu form)
       ```
       let schema_id = SchemaId::from_indy_format(&indy_schema_id);
       let schema = SchemaRegistry::resolve_schema(&client, &schema_id).await
       ``` 
      * Migration tool will provide helper to convert old style indy schema id into new format
    2. Holder resolve Credential Definition from Indy Besu Ledger (VDR converts indy cred definition id representation into Besu form)
       ```
       let cred_def_id = CredentialDefinitionId::from_indy_format(cred_def_id);
       let cred_def = CredentialDefinitionRegistry::resolve_credential_definition(&client, id).await
       ``` 
      * Migration tool will provide helper to convert old style indy credential definition id into new format
    3. Create Proof
  3. Verifier verify Proof
    1. Holder resolve Schema from Indy Besu Ledger (VDR converts indy schema id representation into Besu form)
       ```
       let schema_id = SchemaId::from_indy_format(&indy_schema_id);
       let schema = SchemaRegistry::resolve_schema(&client, &schema_id).await
       ``` 
      * Schema id must be converted as well because proof will contain old style ids
    2. Holder resolve Credential Definition from Indy Besu Ledger (VDR converts indy cred definition id representation into Besu form)
       ```
       let cred_def_id = CredentialDefinitionId::from_indy_format(cred_def_id);
       let cred_def = CredentialDefinitionRegistry::resolve_credential_definition(&client, id).await
       ``` 
    3. Verify proof
2. Credential Issuance goes as before but another ledger is used as a verifiable data registry.

[Detailed technical description is here.](https://github.com/hyperledger/indy-besu/blob/main/docs/migration/migration.md)