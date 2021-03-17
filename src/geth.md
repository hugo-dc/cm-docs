# Geth changes

A Geth fork was created and added some features to merkleize contracts when
performing a full sync.

Changes are described below.

## New modules

Two new modules are introduced to the Geth code base:

- `ssz`: contains the [schema](./schema.md) defined for the code tree.
- `codetrie`: contains the [logic](./codetrie.md) needed to manipulate the tree.

## codemerkleization flag

changes: 

* `cmd/geth/chaincmd.go`
* `cmd/geth/usage.go`
* `cmd/geth/main.go`
* `cmd/utils/flags.go`
* `eth/gen_config.go`
* `eth/backend.go`
* `eth/config.go`

A new flag was added in order to indicate geth we want to *analyze* how the
contracts in a block would be merkleized and get an idea of the sizes of the
proofs required for the contracts in each block.

## state_processor

path: `core/state_processor.go`

One of the main [changes](#changed-code) in Geth is in the `state_processor`.
In function `Process` a [new "Contract Bag"](./codetrie.md#newcontractbag) is
created. A `ContractBag` is a map where the key is the **code hash** and the
value is the **contract code**, this avoids duplicating identical contracts
with the same bytecode. 

Then, each transaction in the block is applied. When the [EVM
Interpreter](#run-evm-interpreter) is *executing* the opcodes, collects
information about touched opcodes and which chunks the opcodes belong.

After contracts bytecode and touched opcodes/chunks are collected, the stats are 
calculated by calling the [`bag.Stats()`](./codetrie.md#stats-contractbag-method) method,
this method merkleizes the contract, and generates the proof needed for that
contract in that specific block. The proof consists in indices, hashes, zero
levels and leaves. And the sum of those proofs for all the is considered the
"proof size".

>, also the indices, hashes, zero levels and leaves
> are encoded as RLP as another way to measure the proof size.
> ... pending to include RLP

The results are stored in a `csv` (`cm-result.csv`) file with the following
fields:

- Block number.
- Proof size: The sum of all indices, zero levels, hashes, and leaves.
- Code size: The sum of all contract's bytecode size in the block.
- Total indices
- Total zero levels
- Total hashes
- Total Leaves

## Run (EVM Interpreter)

path: `core/vm/interpreter.go`

Checks if the `-codemerkleization` flag is set, and `ContractBag` was
initialized correctly.

[Retrieve](./codetrie.md#get-contractbag-method) the `Contract` from the
contract bag, otherwise if the contract does not exist in the contract bag yet
a new `Contract` object is created.

[Marks the chunk](./codetrie.md#touchpc-contract-method) corresponding to the current opcode at the current Program Counter (pc) as "touched".

If the current opcode is `CODECOPY` it will also [touch the
range](./codetrie.md#touchrange-contract-method) of opcodes being copied.

When the current opcode is `EXTCODECOPY`, it will also retrieve the
code for the "external" contract code we want to copy 
and that code will be [marked as touched](./codetrie.md#touchrange-contract-method).



## Added code

This is the directory tree for the new code added to Geth

```
├── codetrie
│   ├── ssz
│   │   ├── types_encoding.go
│   │   └── types.go
│   ├── bin_hex_test.go
│   ├── codetrie.go
│   ├── codetrie_test.go
│   ├── contract.go
│   ├── op.go
│   ├── ssz_test.go
│   └── transition.go
```

## Changed code

```
├── cmd
│   ├── geth
│   │   ├── chaincmd.go
│   │   ├── main.go
│   │   └── usage.go
│   └── utils
│       ├── flags.go
├── core
│   ├── ...
│   ├── state_processor.go
│   ├── ...
├── eth
│   ├── backend.go
│   ├── config.go
│   ├── gen_config.go
```

## Code
