# BEP-100: BSC Application Sidechain

- [BEP-100: BSC Application Sidechain](#bep-100-bsc-application-chain)
    - [1. Summary](#1-summary)
    - [2. Abstract](#2-abstract)
    - [3. Status](#3-status)
    - [4. Motivation](#4-motivation)
    - [5. Specification](#5-specification)
    - [6. License](#6-license)

## 1. Summary

BSC Application Sidechain (BAS) is an infrastructure for developers that helps them to build large scale BSC-based apps with higher throughput and much lower or even zero transaction fees. It is reached by using a separate consensus engine and modern execution environment that can be specified by developers or community.

## 2. Abstract

This paper describes the design of BAS-compatible sidechains and a protocol for interaction between BSC and BAS native assets. Since BAS applications are independent from the BSC consensus set BSC can't rely on their security model. To protect users' funds from malicious actions BSC doesn't introduce direct internal protocol for cross chain communication between BSC and different side chains except native assets, because one compromised BAS application can bring loss of users funds in BSC or other sidechains. The standard will help us to define requirements for such sidechains.

## 3. Status

This BEP is a draft.

## 4. Motivation

Today BSC is experiencing network scalability problems and Binance have proposed to use BAS in their Outlook 2022 paper to solve this problem because these side chains can be designed for much higher throughput and lower gas fees. In this document we want to define a protocol for consensus management and messaging between BAS and BSC so that it is easier for developers to use a ready-made solution and it is easier for BSC to integrate with them.

## 5. Specification

Literally BAS is a modular framework for creating BSC-compatible side chains that defines requirements for integration with the BSC ecosystem and brings development-ready EVM-compatible features like staking, RPC-API or smart contracts. Since BSC doesn’t rely on the BAS security model, that is why there is no default embedded production-ready bridge solution between BSC and BAS networks, but instead of it BAS can provide protocols and standards for integrating third party bridges that can be managed by the BAS validator set of other projects like AnySwap or Celer Network cBridge, of course if they trust the BAS developer team.

BAS brings several programmable and configurable modules that can be used or modified by developers to reach their business goals.

Here is an example of such modules:
+ Networking - for p2p communication between different BAS nodes;
+ Blockchain & EVM - for block producing and EVM transaction execution, of course each BAS can define their own runtime execution environment based, for example, on WebAssembly;
+ Web3 API - for basic compatibility with Web3 ecosystem including MetaMask and other applications;
+ Transaction Pool - for managing internal BAS policies for transaction filtering and for charging fees for the system operational;
+ PoA & PoS Consensus - for users to be able to vote for the honest validators in the BAS network and guarantee the safeness of actions applied on the chain;
+ Storage & State - for persisting local data.

### 5.0 Circulation Model and Native Asset Bridge

The key parts of each BAS application are native token circulation model and cross chain bridge for native assets. Native assets of the BAS chain are located in the BAS application and managed by sidechain directly. BAS is designed to provide cross chain functionality for the native assets. Since native assets are fully managed by BAS developers they can compromise token supply or mint/burn tokens. We leave such a decision on the conscience of the developers, because by manipulating the supply of their tokens they only risk their reputation. The only thing required for native cross chain bridge is block header verification that allows one to keep an active validator set and verify the correctness of cross chain transactions.

BAS is not designed to have a BSC-compatible consensus or EVM execution environment, because BAS is technology-agnostic solution. Here we provide only basic functionality for the BAS applications and help them to set up their work and be a part of the BSC ecosystem. We can’t verify each operation on the chain and here we fully trust BAS developers, but we must be sure that validator transition is not compromised that is strictly required for the cross chain operations for the native asset. To reach this we’d like to introduce a block header verification function (BHVF) that must be specified by the BAS development team. BHVF is a function in the Solidity language that is able to verify block headers from BAS applications. This is a simplified version of the state verification function from Polkadot. BHVF is one of the required parameters to register BAS in the BSC’s smart contract. Also BHVF is responsible for verifying transaction receipt from blockchain that allows to prove correctness of cross chain transfer from BAS to BSC chain.

Block header verification is not a very complicated function, but we will have to verify block headers to be sure that validator transition is well-done. Also using BLS/BN we don’t have to pass all block headers into it, it’s enough to publish only epoch blocks that contain new validator sets and signatures from all previous validators.

If we assume that block verification consumes ~50k gas (w/ state modification), then we have next calculations:
+ per each block: 50k gas/block, then its ~16k gas/sec
+ per epoch: 50k gas/epoch (epoch can be from 5 minutes up to 1 day), then it's between ~160 gas/sec (for 5 min epoch) and 0.57 gas/sec (for 1 day epoch)

Of course gas consumption depends on the epoch length and it's not required for BAS to have very short epochs. Epoch length of one day is recommended to be used for validator transition. It can be reduced based on application needs to 6-12 hours, but it doesn’t change gas consumption a lot.

BAS brings native cross chain bridge and its embedded to BAS as a system smart contract. Here we specify interfaces for the EVM version of BAS:

```solidity
interface INativeAssetBridge {

  function deposit() external payable;

  function withdraw(bytes[] validatorSetSignatures, bytes transactionReceipt) external;
}
```

Native asset bridge supports next user flows:
+ Deposit (BAS -> BSC). When a user calls a deposit function it locks his native tokens in the smart contract and emits events. To be able to mint peg tokens in BSC chain, the user generates a proof that contains information about transaction receipt (including emitted events) with merkle patricia trie proof. This proof should be uploaded into the BSC chain where a `BASValidatorHub` smart contract can verify this proof and validators signatures from the BAS chain.
+ Withdrawal (BSC -> BAS). Withdrawal of the funds is opposite to deposit flow. The difference here is that the user must burn his pegged tokens in the BSC chain to use this proof to withdraw tokens from native asset bridge smart contracts. Validators in the BAS network must verify the correctness of this operation and are responsible for preventing double spend attacks.

BAS developers might specify fees for inter-chain operations and cross chain operations between BAS and BSC chains. Such policy should be specified in the BAS smart contract and fees or fines related mechanisms shouldn’t be presented in BSC smart contracts or block verification functions.

### 5.1  BAS Validator Hub

By default BSC brings only native asset cross chain functionality to the BAS applications. To reach this, each BAS application must be registered in the BSC smart contract. This smart contract assigns chain id to the BAS application and specifies BHVF (block header verification function). This function is used to verify the block header, but w/o state transitions. BHVF should be written in Solidity and be able to verify block header, the correctness of the chain for N blocks and check signatures. Since default BAS implementation is BSC-based then default chain implementation is supported by BAS validator hub, we define such function to allow other developers that don’t want to rely on EVM or parlia consensus engine to write their own verification function. Of course developers might publish malicious functions or a function with a vulnerability that allows them to skip block verification and they should provide an audition of this code to be trusted by a community.

```solidity
interface IBlockHeaderVerificationFunction {
    function verifyBlockHeader(bytes rawBlockHeader, address[] existingValidatorSet) external pure returns (address[] newValidatorSet);
    
    function verifyCrossChainPacket(bytes rawBlockHeader, bytes receiptsProof, bytes rawReceipt) external pure returns (
        uint256 transferredAmount,
        address recipientAddress
    );
}

interface BASValidatorHub {
  function registerBAS(string projectName, address[] initialValidatorSet, IBlockHeaderVerificationFunction bhvf) external;
}
```

P.S: This interface is a draft and might change in the future.

System contract address of BAS Validator Hub smart contract is: `0x0000000000000000000000000000000000007000`

### 5.2 Fast-finality and BLS cryptography

Parlia is a BFT-like consensus where only one validator produces a block and to be sure in the correctness of this operations we must wait for the confirmation time, usually its 2/3*N+1, where N is active validators (15 blocks for the current configuration). It means that to prove one block we must upload at least 15 blocks to the blockchain. BLS cryptography with parlia’s fast-finally can solve this problem because in this case we can collect one aggregated signature and send only this aggregated signature in the BSC, but here we must know BLS public keys of each validator. Currently BLS cryptography is merged to the official geth repository, but it's not supported by BSC yet, so it might take some time to apply these changes to the parlia consensus engine. Also it’s not strictly required to use BLS, geth has support of the BN256 curve since Byzantium fork can be used as a replacement for BLS aggregated signatures.

Here we have next solutions:
+ Break compatibility with Parlia and implement fast-finality in the BAS version of Parlia using BN256 curve.
+ Wait for the BSC’s version of fast-finality and for BLS support.

For now let's assume that fast-finality is an optional solution for the BAS since we have a lot of moving parts here.

### 5.3 System Smart Contracts

Each BAS sidechain is technology-agnostic, meaning that it’s able to include or modify any module inside BAS and bring any consensus or runtime execution environment. By default BAS provides an EVM execution environment with a predefined set of system smart contracts for platform operation. If BAS developers want to bring more functionality to their sidechain then they should implement it on their own or contribute it to the BAS official template to extend the default module set with additional extensions that can be used by other developers in the future.

Predefined BSC-compatible system smart contracts:
+ Staking (`0x0000000000000000000000000000000000001000`) - for managing validator delegations and active validator set
+ SlashingIndicator (`0x0000000000000000000000000000000000001001`) - for slashing not active validators
+ SystemReward (`0x0000000000000000000000000000000000001002`) - treasury for the system rewards to cover relay fees and others

BAS defined smart contracts:
+ Governance (`0x0000000000000000000000000000000000007002`) - default on-chain implementation by Compund’s Alpha governance
+ ChainConfig (`0x0000000000000000000000000000000000007003`) - configuration for the consensus that is managed by on-chain governance
+ NativeAssetBridge (`0x0000000000000000000000000000000000007004`) - cross chain bridge for native assets between BAS and BSC

Using the parlia consensus engine motivates users to stake their funds and vote for the honest validators that makes BAS sidechain much more decentralized and trustless. It also helps stakers to gain rewards from their stakes by receiving fees from block producers.

To be able to interact with Parlia consensus engine, BAS must support staking contract that is compatible with next interface:

```solidity
interface IValidatorSet {
  function init() external;
  function getValidators() external view returns (address[] memory);
  function deposit(address validator) external payable;   
  receive() external payable;
}

interface ISlashingIndicator {
  function init() external;
  function slash(address validator) external;
}

interface ISystemReward {
  function init() external;
  receive() external payable;
}
```

BAS provides default implementation and financial model for staking and it is embedded to the genesis block as system smart contract, but BAS developers can choose another model based on their business requirements.

#### Staking Smart Contract

Default implementation of BAS will contain staking smart contracts on Solidity for EVM execution environment. This smart contract is an extension over `IValdiatorSet` and allows users to manage active validators based on the total delegated amount and distribute rewards between stakeholders. It's not strictly required to have EVM implementation of such smart contracts but for default BAS solutions it might be very useful. I’m not going to specify here ABI methods for such smart contracts because it's implementation-defined and each BAS developer can implement their own version based on their requirements.

#### Governance

Each BAS should have on-cain governance to let users vote for the new proposal on the chain. The governance is based on the compound’s alpha governance and validator owners in the chain are able to create new proposals and vote for them. Voting power is distributed based on the total delegated amount to the validator. Once ⅔ of the quorum is reached and >51% of votes are for the proposal then it can be executed by anyone on the chain. Governance is able to manage staking parameters like felony threshold or jail period.

## 6. License

The content is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
## 7. Cooperation

BAS starts its development based on [Ankr](https://github.com/Ankr-network/bas-template-bsc), special thanks to [Ankr](https://www.ankr.com/) team and [Nodereal](https://nodereal.io/) team for their contribution.


