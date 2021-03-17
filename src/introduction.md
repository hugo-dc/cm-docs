# Introduction

This document describes experiments in Geth using chunk based merkleization
according to [EIP 2926](https://eips.ethereum.org/EIPS/eip-2926), analyzing the
Merkleization of contracts touched in blocks execution during a full sync.

We are using SSZ Merkleization which was implemented in the
[fastssz](./fastssz.md) library and included: 

- New structures/scheme were defined in order to represent a binary tree.
- Proof Generation Proof verification

