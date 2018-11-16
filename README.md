# Voĉdoni

Status of the document: *work in progress*

## A fully verifiable decentralized anonymous voting system

Cryptography and distributed p2p systems have brought a new, revolutionary digital technology which might change the way society organizes: blockchain. Among many other applications, it allows decision making through a secure, transparent, distributed and resilent voting system.

In this document we propose the design, architecture of a decentralized anonymous voting platform using decentralized identities.

## Overview

We want to bring decentralized voting to mass adoption. This requires a solution that has a user experience at the level of current centralized solutions.

+ Minimize transactions to the blockchain 
+ Voter does not need to interact with the blockchain
+ Secure and anonymous voting using ZK-snarks
+ Data availability via distributed filesystems such as IPFS
+ Use static a web page or APP for interacting with the system
+ Incentivate third parties to participate (relays) by adding a reward system

![overall](https://github.com/vocdoni/docs/raw/master/img/overall_design.png)

## Identity
The system is agnostic to the identity scheme used.

We are developing our implementation using [Iden3](https://iden3.io), for having the best blance of decentralization vs scalability.

##Players
### Actors
`Voter`
+ A `voter` is the end user that will vote
+ Its digital representation we call it `identity`
+ Inside the voting process is identified by a public key
+ Can interact manage all interactions through a light client (web/app)

`Organizer`
+ The user or entity that creates and manages an especific voting process
+ Needs to interact with the blockchain
+ Pays for the costs of a process
+ Has the highest interest for the process to success

`Relay`
+ Is used as a proxy between the `voter` and the blockchain
+ Is a selfish actor. Good behaviour is ensure through economic incentives
+ It may have to be splitted into serveral `relay` types
+ Develops functions that would not be possible on a light client
  - It relays voting transactions to other `relays`
  - It aregates voting transactions and adds them into the blockchain
  - It validates zk-snarks proofs
  - It ensures data aviability on IPFS
  - It provides Merkle-proofs to `User` requests that they are in the `census merkle tree`
  - Provides RPC access to the blockchain

### Elements

`Process`
+ We call `process` to a especific instance of a `voting process`

`Census Merkle Tree`
+ A Merkle-tree made of the `public keys` of all the `voters`
+ The Merkle-root is hosted in the blockchain as a proof the cenus
+ The tree needs to be publicly available (IPFS) for everyone to verify it.
+ The sk-snarks circuit will use its root to validate if `voter` `public key` is part of it

`Voting smart-contract`
+ All metadata that defines a `process` is stored here
+ The `User` light clients retrieve the `process` data from here
+ All the agragated `vote packages` hashes are added here
+ It holds the funds used to pay the `relays`
+ It holds the funds that the `relays` need to stake to ensure their good behaviour
+ When a `process` is successfully finished it transfers the funds to the `relays`

`ProcessId` (previously "election ID")
+ A unique identifier for each process
+ Generated by the `Voting smart-contract` when a new process is created

`Vote package`
+ Is the set of data sent by the `Voter` to the `relay` in order to vote 
  - Zk-snarks proof
  - Encrypted vote: encrypt(selected `vote options` + random nonce)
  - `Nullifier` : hash( `ProcessId` + `Voter` `private key` )
  - `ProcessId`
  - Anti spam proof of work

`Vote encryption key`
+ Is the public key used by he `Voters` to encrypt their vote
+ Its private key needs to be made publish at the end of the process, for everyone to validate the votes
+ Multiple `vote encryption keys` can be used to ensure that no one has access to the results before the `process` is finished

`Voting options`
+ A potential option for the user to choose when she votes
+ They are published when a `process` is created
+ They could be exclusive or not

`Zk-snarks proof`
+ A `Zk-snarks proof` that proves that the `user` can vote
+ It is generated in the `user` light-client
+ It is an CPU and memory intensive process

### Voting process chronology

1. `Identity` creation
  + Before the process in itself starts `voters` must have their digital `identity`  already created
  + The unique requirement for the those `identities` is that they need to be represented by a `public key`.
  + The system is agnostic to the identity system used but an integration will be required for each of them.

1. The `organizer` generates a census
  + This first design iteration, assumes that the `organizer` has a list of all the `voters` that can participate
  + It agregate all the `public keys` of the `voters` and generates the `census merkle tree` with them.

2. The `organizer` publishes a new voting process.
  + Via a user interface, it fills the required `process metadata` regarding a voting process.
    - Merkle Root  of the `census Merkle tree`
    - `Vote encryption key`
    - Available `voting options`
    - `Process` start time (block number)
    - `Process` end time (block number)
  + Sends a transaction to the `voting smart-contract`
    - It includes the `process metadata`, so its public for the other players
    - The funds sent in the transaction will be used to pay the `relays`
    - The amount sent is proportional to the needs of the `process` (number of participants, relay redundancy...)
  + In parallel it also publishes the `census Merkle tree` to IPFS, to make it available to everyone else.

3. `Voter` votes

  + Gets the `process metadata` from the `voting smart-contract`
  + Gets the census Merkle-proof from a `relay`
  + Checks that can vote
  + Selects the desired `voting options` from the `process metadata`
  + Encrypts the `voting options` and a random nonce with the `vote encryption keys`
    - `vote = encrypt(selected_voting_options + random_nonce)`
  + Generates the nullifier
    - `nullifier = hash( process_id + user_private_key)`
  + Generates a zk-snarks proof to demostrate that she can vote (owns the `public key` and is included in the `census Merkle-tree`)
    - **private input**: `Private Key`, `census tree` Merkle-proof
    - **public input**: `census tree` Merkle-root, `Nullifier`, `ProcessId`, `encrypted vote`
    - **output**: Zk-Snarks proof that she can vote
  + It compiles the `vote package` with
    - `Zk-snarks proof`
    - `Encrypted vote`
    - `Nullifier`
  + Potentially it can encrypt the `vote package` with one or more `relay` public keys in order to minimize IP mapping
  + Generates a Proof-of-Work nonce  (to avoid relay node spaming)
  + It sends the `vote package` and the nonce the `relays` pool:

4. The p2p relay pool receives the data from the user
  + Relay nodes veriffy the PoW and the Zk-snarks proof, if not valid the packet is discarted
  + Choose a set of pending votes to relay
  + Aggregate the votes from several voters in a single packet of data
  + Add the aggregated data to IPFS
  + Upload the IPFS hash to the Blockchain (Ethereum)
  + Keep the data until the end of the election


4. Once the election is finished
  + The organizer publishes the private key, so the votes and proofs are available
  + The organizer checks the votes and validate the relay operation
  + Relay nodes will be rewarded according their contribution

![voting_process](https://github.com/vocdoni/docs/raw/master/img/voting_process.png)


### Unresolved questions
+ How to check `relay` bad behaviour?
  + Time limits for an un-returned hash?

+ How to choose which relay to encrypt the `vote package` with? How can it be randomized?
+ How does the `relay` pool looks like?

### Known Weaknesses