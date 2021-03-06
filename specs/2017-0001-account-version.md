# Generalized Account Versioning Scheme for Hard Fork

    Number: ella-2017-0001
    Title: Generalized Account Versioning Scheme for Hard Fork
    Author: Wei Tang <hi@that.world>
    Status: Proposed
    Type: Standards Track
    Layer: Meta/Consensus
    Discussion: https://github.com/ellaism/specs/issues/4
    Created: 2017-12-30
    
## Abstract

This defines a method of hard forking while maintaining the exact functionality of existing account by allowing multiple versions of the virtual machines to execute in the same block. This is also useful to define future account state structures when we introduce the on-chain WebAssembly virtual machine.

## Motivation

When EIP150 was activated on Ethereum Classic, it was critised that it changes the behaviour of many existing smart contracts on the blockchain, and made several ones unusable. The same thing will also happen in the future if we decided to add some of the Byzantium hard forks to Ethereum Classic.

Take the REVERT opcode as an example. Currently there're already many contracts on the Ethereum Classic blockchain with the REVERT opcode. This opcode is now considered invalid. When EVM executes to REVERT, it will treat it as an invalid opcode, consume all the gases, and return an error. However, when we hard fork to define the actual REVERT, existing users would suddenly find that the behavior of their existing contract would have changed -- instead of consuming all gases, it will now only consume less than the given gas limit and return. Several on-chain contract might rely on the old behavior for some of their functionality. Besides, if a dapp developer does not follow the hard fork, he or she will suddenly find that the dapp might suddenly break because the existing method to detect whether a contract execution succeeds or not does not work any more.

One of the important mission of blockchain is to "become your own bank". In Bitcoin, that means once a transaction is executed, the fund will always be controlled by the receiver unless he or she sends it out again. In Ethereum, an additional important property is that, once a smart contract is deployed, it should always execute in the original expected manner. As a result, it is important to have a way to **maintain compatibility of existing contracts** and at the same time **easily add new protocol features**.

Note that this specification might not apply to all hard forks. We have emergency hard forks in the past due to network attacks. Whether they should maintain existing account compatibility should be evaluated in individual basis. If the attack can only be executed once against some particular contracts, then the scheme defined here might still be applicable. Otherwise, having a plain emergency hard fork might still be a good idea.

## Specification

### Account State

After the first hard fork using this scheme, the account state stored in the world state trie is changed to become a five-item RLP encoding: `nonce`, `balance`, `storageRoot`, `codeHash` and `version`. The `version` field defines that when a contract call transaction or `CALL` opcode is executed against this contract, which version of the virtual machine should be used. Four-item RLP encoding account state are considered to have the `version` 0.

`CREATE` opcode and the contract creation transaction, however, would only deploy contracts of the newest version.

The behavior of `CALLCODE` and `DELEGATECALL` are not affected by the hard fork -- they would fetch the contract code (even if the contract is deployed in a newer version) and still use the virtual machine version defined in the current contract (or in the case of within the input of contract creation transaction or `CREATE` opcode, the newest virtual machine version) to execute the code.

If a message call transaction creates a new account, the newly created account will have the newest account version number in the network.

There is no versioning for precompiled contracts. Once hard fork, they're available for both old and new accounts.

### Handle Receipts

For simplicity, transaction receipts are always of the newest format, but they should maintain compatibility by either having a default value or allowing null value for newly defined fields. In this way, virtual machines executed in older versions would not need to deal with the new fields in receipts.

## Example

Consider we would like to have the REVERT opcode hard fork in Ethereum Classic using this scheme. After the hard fork, we have:

* Existing accounts are still of four-item RLP encoding. When a transaction has `to` field pointing to them, they're executed using version 0 of EVM. REVERT is the same as INVALID and consumes all the gases.
* A new contract creation transaction is executed using version 1 virtual machine, and would only create version 1 account on the blockchain. When executing them, it uses version 1 of EVM and REVERT uses the new behavior.
* When a version 0 account issues a CALL to version 1 account, sub-execution of the version 1 account uses the version 1 virtual machine.
* When a version 1 account issues a CALL to version 0 account, sub-execution of the version 0 account uses the version 0 vritual machine.
* When a version 0 account issues a CREATE, it always uses the newest version of the virtual machine, so it only creates version 1 new accounts.

## Discussions

### Performance

Currently nearly all full node implementations uses config parameters to decide which virtual machine version to use. Switching vitual machine version is simply an operation that changes a pointer using a different set of config parameters. As a result, this scheme has nearly zero impact to performance.

### Smart Contract Boundary and Formal Verification

Many current efforts are on-going for getting smart contracts formally verified. However, for any hard fork that introduces new opcodes or change behaviors of existing opcodes would break the verification of an existing contract has previously be formally verified. Using the scheme described here, we define the boundary of how a smart contract interacts with the blockchain and it might help the formal verification efforts:

* A smart contract has only immutable access to information of blockchain account balances and codes.
* A smart contract has only immutable access of block information.
* A smart contract or a contract creation transaction can modify only its own storage and codes.
* A smart contract can only interact with the blockchain in a mutable way using `CALL` or `CREATE`.

### WebAssembly

This scheme can also be helpful when we deploy on-chain WebAssembly virtual machine. In that case, WASM contracts and EVM contracts can co-exist and the execution boundary and interaction model are clearly defined as above.
