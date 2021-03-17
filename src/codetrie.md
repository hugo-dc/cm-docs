# CodeTrie

A new go module `codetrie` was added, this module contains functions that
allows to create contract objects, and the corresponding trees for each
contract.

#### NewContractBag

path: `codetrie/contract.go`.

Returns a new empty [`ContractBag`](#contractbag-type) object.

```go
func NewContractBag() *ContractBag {
	return &ContractBag{
		contracts: make(map[common.Hash]*Contract),
	}
}
```

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

A contract object contains a code, and a list of [Chunk](./schema.md#chunk)s
corresponding to the code, it can also store which chunks were touched during
the execution of the contract. When a chunk is touched, the value of the
element with key `index` will be set to `true`.

```go
type Contract struct {
	code          []byte
	chunks        []*Chunk
	touchedChunks map[int]bool
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

#### CMStats (type)

path: `codetrie/contract.go`

```golang
type CMStats struct {
	NumContracts  int
	ProofSize     int
	CodeSize      int
	ProofStats    *ProofStats
	TouchedChunks []int
}
```



#### Stats (ContractBag Method)

path: `codetrie/contract.go`

For each contract in the contract bag, get the code size and its
"[ProofStats](#proofstats-contract-method)" (RLPSize, Indices, ZeroLevels,
Hashes, TouchedChunks). These results will be accumulated in the [code
merkleization block stats](#cmstats-type).

[code](./code.md#stats-contractbag-method)


#### Get (ContractBag Method)

path: `codetrie/contract.go`

Checks if contract is already in the contracts bag. To do that checks if the
hash for the input code already exists in the `contracts` map. If it exists it
will return the pointer to the `Contract` corresponding to that code hash.

If the contract was not previously added to the contract bag it will create
a [new contract](#newcontract) corresponding to the provided bytecode.


#### CodeSize (Contract Method)

path: `codetrie/contract.go`

Loops through all contracts in the bag. Sums the length of each contract code
size.

Returns the result.

#### ProofStats (Contract Method)

[`Prove`](#prove-contract-method) the contract. Get the length of Indices,
ZeroLevels, Hashes, TouchedChunks and Leaves in the resulting proof.

> It also serializes the proof as RLP in order to know the serialized length.

#### Prove (Contract Method)

path: `codetrie/contract.go`

[Creates a new SSZ tree](#getssztree) using the contracts code and specifying
the chunk size of 32.

Create new array of **metadata** indices (`mdIndices`), it is initialized with the indexes 7,
8, 9,  and 10.

`mdIndices := []int{7, 8, 9, 10}`


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

Creates a new array of chunk indices (`chunkIndices`), with a capacity equal to 
the double of the total touched chunks:

for each of the touchedChunks, append the chunk index to the `chunkIndices`,
the chunk index is `6144` + the "index" of the `touchedChunks` array. What is
appended is `chunkIdx*2` and `chunkIdx*2+1`.

Creates a [Multiproof](./fastssz.md#provemulti) based on the array of
`mdIndices` and `chunkIndices`.

The multiproof is [compressed](./fastssz.md#compress-multiproof) and
[returned](./fastssz.md#compressedmultiproof).

#### NewContract

path: `codetrie/contract.go`

[Splits](#chunkify) the code into chunks of 32 bits

Creates and returns a new `Contract` object containing the `code`, the `chunks` 
and an empty map for `touchedChunks`.


#### TouchPC (Contract Method)

path: `codetrie/contract.go`

Calculates the corresponding chunks number for the `pc` (`chunk_number = pc / 32`).

Adds the chunk as touched by creating an element in the `touchedChunks` map,
where the key is the `chunk number` and the value is `true`.

#### TouchRange (Contract Method)

path: `codetrie/contract.go`

Calculate the chunks for the given range of opcodes and marks them as touched.

#### GetSSZTree

path: `codetrie/codetrie.go`

It [gets](#preparessz) a new [CodeTrie](./schema.md#codetrie). Then calls the
fastssz generated code to get a SSZ tree and this tree is returned.

#### prepareSSZ

[Chunkify](#chunkify)s the provided code, if the requested chunk size is
different than 32, an error `MerkleizeSSZ only supports chunk size of 32`
occurs.

Calculates the First Instruction Offsets (FIO) for each one of the
[chunk](./schema.md#chunk)

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


