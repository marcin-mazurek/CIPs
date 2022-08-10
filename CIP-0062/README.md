---
CIP: 62
Title: Cardano dApp-Connector Governance extension
Authors: Bruno Martins <bruno.martins@iohk.io>
Status: Draft
Type: Standards
Created: 2021-06-11
License: CC-BY-4.0
---

# **Abstract**

This document describe the interface between webpage / web-based stack and cardano wallets. This specificies that API of the javascript object that need to be injected into the web applications in order to support all the Governance features.

These definitions extend [CIP-30 (Cardano dApp-Wallet Web Bridge)](https://cips.cardano.org/cips/cip30/) to provide specific support for vote delegation.

# **Motivation**
The goal for this CIP is to extend the dApp-Wallet web bridge to enable the construction of transactions containing metadata that conforms to
[CIP-36 (Catalyst/Voltaire Registration Transaction Metadata Format - Updated)](https://cips.cardano.org/cips/cip36/),
enabling new functionality including vote delegation to either private or public representatives (dReps),
splitting or combining of private votes,
the use of different voting keys or delegations
for different purposes (Catalyst etc).

# **Specification**

## `Types`

### **GovernanceKey**

```
type GovernanceKey = {
  votingKey: string,
  weight: number
}

```

`votingKey`: Ed25519 pubkey 32 bytes HEX string  

`weight`: Used to calculate the actual voting power using the rules described
in 
[CIP-36](https://cips.cardano.org/cips/cip36/).


### **Purpose**

```
type enum Purpose = {
  CATALYST = 0,
  OTHER = 1
}

```

`Purpose`: Defines the purpose of the delegations. This is used to limit the scope of the delegations.  For example, a purpose might be a subset of Catalyst proposals, a council election, or even some private purpose (agreed by convention).

## **Namespace**

### **cardano.{walletName}.governance.enable(): Promise\<API>**
The `cardano.{walletName}.governance.enable()` method is used to enable the governance API. It should request permission from the wallet to enable the API. If permission is granted, the rest of the API will be available. The wallet should maintain a specific whitelist of allowed clients for this API. This whitelist can be used to avoid asking for permission every time.

This api being an extension of [CIP-30 (Cardano dApp-Wallet Web Bridge)](https://cips.cardano.org/cips/cip30/), expects that `cardano.{walletName}.enable()` to be enabled and added to CIP-30 whitelist implicitly.

When both this API and **CIP-30** is being enabled, is up to the wallet to decide the number of prompts requesting permissions to be displayed to the user.

# **`Jormungandr API`**

## **api.submitVotes**(votes: Vote[], spendingCounter: number): Promise\<hash32>

### `spendingCounter`: 
The spending counter is used to prevent double voting. The current spending counter for the account should be provided and the implementation should increment it for each vote before submission and attach it to the vote according to [Jormungandr Voting] (https://input-output-hk.github.io/jormungandr/jcli/vote.html#voting). This needs to be in sequential order.

### **`votes`**: 

```
interface Vote {
  proposal: Proposal,
  choice: number,
  expiration: BlockDate
}
```
#### `expiration`: 
chain epoch \& slot for when the vote will expire. The type used is: 

```
interface BlockDate {
    epoch: number
    slot: number
}
```

#### `choice`: 
The choice **index** we want to vote for. An `UnkownChoiceError` should be thrown is the value is not within the `proposal` option set.

#### **`proposal`** : 
proposal information. Include the range of options we can use to vote. This defines the allowed values in `choice`.

```
interface Proposal {
  votePlanId: number
  voteOptions: number[]
}
```

##### `votePlanId`: 
the vote plan id. This is used to identify the vote plan. This should be the same as anchored on chain.

##### `voteOptions`: 
The vote options. This is the set of options we can vote for.

### **Returns**

`hash32` - This is the hash of the transaction that will be submitted to the node.

#### **Errors**
`InvalidArgumentError` - Generic error for errors in the formatting of the arguments.

`UnknownChoiceError` - If the `choice` is not within the `proposal` option set.

`InvalidBlockDateError` - If the `validUntil` is not a valid block date.

`InvalidVotePlanError` - If the `votePlanId` is not a valid vote plan.

`InvalidVoteOptionError` - If the `index` is not a valid vote option.

# **`Delegation API`**

## **api.getVotingKeys**(): Promise<cbor<PublicKey\>[]>
Should return a list of all the voting keys for the current wallet. 

### **Returns**
An array with the cbor hex encoded public keys.

## **api.rotateVotingKey**(): Promise<cbor<PublicKey\>>
This call should explicitly rotate the current in-use voting key. Given the current `address_index` in the derivation path defined in [CIP-36](https://cips.cardano.org/cips/cip36/), it should be incremented by 1.

The key should be derived from the following path. 

```
m / 1694' / 1815' / account' / role' / address_index'
```

`1694` (year Voltaire was born) Sets a dedicated `purpose` in the derivation path for the voting profile.  

`address_index` - index of the key to use. 

### **Returns**
cbor hex encoded representation of the public key


## **api.getCurrentVotingKey**(): Promise\<cbor<PublicKey\>>

Should return the current in-use voting public-key. The wallet should maintain a reference to the current `adress_index` counter and return the public key for that index.

### **Returns**
cbor hex encoded representation of the public key.

## **api.submitDelegation(delegation: Delegation): Promise\<SignedDelegationMetadata>**

This endpoint should construct the cbor encoded delegation certificate according to the specs in [CIP-36 Example](https://github.com/Zeegomo/CIPs/blob/472181b9c69feeedae0b5b2db8b42d0cf4eb1a11/CIP-0036/README.md#example).

It should then sign the certificate with the staking key as described in the same example as above.

The implementation of this endpoint can make use of the already existing [CIP-30](https://cips.cardano.org/cips/cip30/) `api.signData` and `api.submitTx` to perform the broadcasting of the transaction containing the metadata. 

Upon submission of the transaction containing the delegation cert as part of metadata, the wallet should store the delegation in its local storage and return an object that contains the delegation cert, the signature and the txhash of the transaction that the certificate was submitted with.

## **`Delegation`**

```
export interface Delegation {
    delegations: GovernanceKey[],
    purpose: Purpose
}
```

Defines the structure to be crafted and signed for delegation of voting & their respectively voting power.   Embeds the stake key and reward address from the wallet, and constructs a suitable nonce.

***`delegations`***: List of keys and their voting weight to delegate voting power to.

This should be a call that implicitly cbor encodes the `delegation` object and uses the already existing [CIP-30](https://cips.cardano.org/cips/cip30/) `api.submitTx` to submit the transaction. The resulting transaction hash should be returned.

This should be trigger a request to the wallet to approve the transaction.

### **Returns**

```
interface SignedDelegationMetadata {
  certificate: DelegatedCertificate,
  signature: string, // signature on the CIP-36 delegation certificate
  txHash: string // of the transaction that submitted the delegation 
}
```

#### **DelegatedCertificate**
```
interface DelegatedCertificate {
  delegations: GovernanceKey[] // key delegations,
  stakingPub: string // staking public key
  rewardAddress: string // reward address
  nonce: number //nonce used in the transaction
  purpose: Purpose
}
```

Errors: `APIError`, `TxSendError`

## **Delegation Cert process**

1. **`Get Voting Key`** - use the method **api.getVotingKey** to return a ed25519 32 bytes public key (x value of the point on the curve).

2. **`Collect Voting Keys`** - Collect the keys to delegate voting power to.

3. **`Submit delegation`** - Submit the metadata transaction to the chain using **api.submitDelegation** which implicitly sign and/or can use the already existing **api.submitTx**,  available from [CIP-30](https://cips.cardano.org/cips/cip30/)