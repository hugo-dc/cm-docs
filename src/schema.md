# Schema

The SSZ schema is defined as a series of [types](#types) needed to represent
the Code trie. After defining these [types](#types) it is processed by
[FastSSZ](#fastssz) in order to generate all the data needed to create the
tree, generate and verify proofs, this process generates Go source code in the
[`types_encoding.go`](#geth-code) file.

## Types

### CodeTrie

`CodeTrie` is the type created to represent the code tree:

```go
type CodeTrie struct {
	Metadata *Metadata
	Chunks   []*Chunk `ssz-max:"1024"`
}
```

The code trie consists in some [`Metadata`](#metadata) and the code
[`Chunk`](#chunk)s.

### Metadata

The metadata will store the `Version`, `CodeHash` and total `CodeLength`,
according to the [EIP](https://eips.ethereum.org/EIPS/eip-2926#metadata-fields)

```golang
type Metadata struct {
	Version    uint8
	CodeHash   Hash `ssz-size:"32"`
	CodeLength uint16
}
```

### Hash

The Hash used in the Matadata's `CodeHash` is defined as a byte array.

```golang
type Hash []byte
```

### Chunk

Each Chunk contains a FIO, which corresponds to the First Instruction Offset
(i.e. not part of a `PUSH*` opcode's input).

The Chunk type also contains an array of byte code up to 32 bytes each.


```golang
type Chunk struct {
	FIO  uint8
	Code []byte `ssz-size:"32"` // Last chunk is right-padded with zeros
}

```


