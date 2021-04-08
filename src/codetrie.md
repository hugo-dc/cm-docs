# CodeTrie

A new go module `codetrie` was added, this module contains functions that
allows to create contract objects, and the corresponding trees for each
contract.

#### ContractBag (Type)

path: `codetrie/contract.go`.

A `ContractBag` contains a `contracts` map where the key is a `Hash` (the
contract code hash) and the value is a [`Contract`](#contract) object pointer.

```go
type ContractBag struct {
	contracts map[common.Hash]*Contract
}
```

#### Contract (Type)

path: `codetrie/contract.go`.

A contract object contains `code`, and a list of chunks that were touched
during the execution of the contract. When a chunk is touched, the value of the
element with key `index` will be set to `true`.

```go
type Contract struct {
	code          []byte
	touchedChunks map[int]bool
}
```

#### CMStats (type)

path: `codetrie/contract.go`

```golang
type CMStats struct {
	NumContracts int
	ProofSize    int
	CodeSize     int
	ProofStats   *ssz.ProofStats
	RLPStats     *ssz.RLPStats
}
```

#### Chunk (Type)

path: `codetrie/codetrie.go`.

A chunk object containes two fields: 

- `fio`: is the first instrucction offset in the chunk.
- `code`: the bytecode in the chunk, it can be up to 32 bytes. 

```go
type Chunk struct {
	fio  uint8 // firstInstructionOffset
	code []byte
}
```

#### RLPStats (type)

```go
type RLPStats struct {
	RLPSize    int
	UnRLPSize  int
	SnappySize int
}
```

#### NewContractBag

path: `codetrie/contract.go`

Returns a new empty [`ContractBag`](#contractbag-type) object.

```go
func NewContractBag() *ContractBag {
	return &ContractBag{
		contracts: make(map[common.Hash]*Contract),
	}
}
```

#### Stats (ContractBag Method)

path: `codetrie/contract.go`

For each contract in the [ContractBag](#contractbag), gets the code size,
[Prove](#prove-contract-method)s the contract, obtaining
a [Multiproof](./fastssz.md#multiproof) and a [Compressed
Multiproof](./fastssz.md#compressedmultiproof). 

Stats for the [Multiproof](./fastssz.md#multiproof)s,
[CompressedMultiproof](./fastssz.md#compressedmultiproof)s, and for the [RLP
encoded](#newrlpstats) proof are collected in the main [Stats](#cmstats-type)
object.

```go
func (b *ContractBag) Stats() (*CMStats, error) {
	stats := NewCMStats()
	stats.NumContracts = len(b.contracts)
	for _, c := range b.contracts {
		stats.CodeSize += c.CodeSize()
		rawProof, err := c.Prove()
		if err != nil {
			return nil, err
		}
		p := ssz.NewMultiproof(rawProof)
		cp := ssz.NewCompressedMultiproof(rawProof.Compress())

		ps := cp.ProofStats()
		stats.ProofStats.Add(ps)

		rs, err := ssz.NewRLPStats(p, cp)
		if err != nil {
			return nil, err
		}
		stats.RLPStats.Add(rs)
	}
	stats.ProofSize = stats.ProofStats.Sum()
	return stats, nil
}
```

#### Get (ContractBag Method)

path: `codetrie/contract.go`

Checks if contract is already in the contracts bag. To do that checks if the
hash for the input code already exists in the `contracts` map. If it exists it
will return the pointer to the `Contract` corresponding to that code hash.

If the contract was not previously added to the contract bag it will create
a [new contract](#newcontract) corresponding to the provided bytecode.


#### CodeSize (Contract Method)

path: `codetrie/contract.go`

Returns the current contracts code size.

#### Prove (Contract Method)

path: `codetrie/contract.go`

[Creates a new SSZ tree](#getssztree) using the contracts code and specifying
the chunk size of 32.

Create new array of **metadata** indices, it is initialized with the indexes 7,
8, 9,  and 10.

    mdIndices := []int{7, 8, 9, 10}
    => [7, 8, 9, 10]

Another array is created containing the **touched chunks**, each one of this
chunks obtains an index which will correspond to the place they belong in the
tree.

Creates a [Multiproof](./fastssz.md#provemulti) based on the array of
metadata indices and chunk indices.

The multiproof is [compressed](./fastssz.md#compress-multiproof) and
[returned](./fastssz.md#compressedmultiproof).

This Multiproof will have the following Tree structure:

```
          1
         / \
        /   \
       /     \
      /       \
     /         \
    2           3
   / \        /   \ 
  4   5      6     7
 / \ / \    / \  
8  9 10 11 ... ... 

```

- `6` - Root of the tree of depth 10 (Contains the touched chunks)
- `7` - Number of chunks
- `8` - Version
- `9` - CodeHash
- `10` - Code length
- `11` - Empty field

#### NewContract

path: `codetrie/contract.go`

Returns a [Contract](./codetrie.md#contract-type) object containing the `code`
and an empty map of `touchedChunks`.

#### TouchPC (Contract Method)

path: `codetrie/contract.go`

Calculates the corresponding chunk number for the `pc` (`chunk_number = pc / 32`).

Adds the chunk as touched by creating an element (if not exists) in the
`touchedChunks` map, where the key is the `chunk number` and the value is
`true`.

#### TouchRange (Contract Method)

path: `codetrie/contract.go`

Calculate the chunks for the given range of opcodes and marks them as touched.

#### GetSSZTree

path: `codetrie/codetrie.go`

It [gets](#preparessz) a new [CodeTrie](./schema.md#codetrie). Then calls the
[fastssz](./fastssz) generated code to get the SSZ Tree which will get
returned.

#### prepareSSZ

path: `codetrie/codetrie.go`

[Chunkify](#chunkify)s the provided code, if the requested chunk size is
different than 32, an error `MerkleizeSSZ only supports chunk size of 32`
occurs.

Calculates the First Instruction Offsets (FIO) for each one of the
[chunk](./schema.md#chunk)s

Creates the `metadata`, which consists in version (`0`), the code hash and code
length.

Returns a new [CodeTrie](./schema.md#codetrie) containing the `metadata`and
chunks.

#### Chunkify

path: `codetrie/codetrie.go`

Calculates the total number of chunks are required for the given bytecode.

Splits the code into 32 bits chunks and also finds the first instruction offset
([FIO](#setfio)), this is to consider that the initical bytes might be part of a previous
PUSH* input and to not consider those as opcodes.

#### setFIO

Loops the bytecode contained in the chunk's, when a `PUSH*` opcode is found, it
will calculate if the input data for `PUSH*` is exceeding the current chunk, if
that's the case it will calculate which offset is the Firs Instruction in the
next chunk (avoid considering `PUSH*` data as instructions).


#### NewRLPStats

path: `codetrie/ssz/proof.go`

Serializes the [CompressedMultiproof](./fastssz.md#compressedmultiproof) as RLP
and gets the RLP size (`RLPSize`).

Serializes the uncompressed [Multiproof](./fastssz.md#multiproof) as RLP and
gets the RLP size (`UnRLPSize`).

The uncompressed [Multiproof](./fastssz.md#multiproof)'s RLP is compresssed
using [Snappy](https://github.com/golang/snappy) compression (`SnappySize`).

The [RLPStats](#rlpstats-type) object is returned containing the previous
fields `RLPSize`, `UnRLPSize`, and `SnappySize`.

