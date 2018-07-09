# IPLD and hash-based data structures

## Table of Contents

-   [Background Information](#background-information)
    - [Bitcoin Block Structure](#bitcoin-block-structure)
    - [Bitcoin Block Header](#bitcoin-block-header)
    - [Possible Implementation](#possible-implementation)
- [btc.go](#btc.go)
- [Planning](#planning)
- [Meetings](#meetings)

## Background Information

### Bitcoin Block Structure

| Field               | Description                                   | Size                      |
|---------------------|-----------------------------------------------|---------------------------|
| Magic no            | value always 0xD9B4BEF9                       | 4 bytes                   |
| Blocksize           | number of bytes following up to end of block  | 4 bytes                   |
| Blockheader         | [Bitcoin Block Header](#bitcoin-block-header) | 80 bytes                  |
| Transaction counter | positive integer                              | 1 - 9 bytes               |
| transactions        | list of transactions                          | ::<Transaction counter>:: |

## Bitcoin Block Header

| Field          | Description                                                | Updated when...                             | Size     |
|----------------|------------------------------------------------------------|---------------------------------------------|----------|
| Version        | Block version number                                       | a software upgrade that has a newer version | 4 bytes  |
| hashPrevBlock  | 256-bit hash of the previous block header                  | A new block is received                     | 32 bytes |
| hashMerkleRoot | 256-bit hash based on all of the transactions in the block | A transaction is accepted                   | 32 bytes |
| Time           | Current timestamp as seconds since 1970-01-01T00:00 UTC    | Every few seconds                           | 4 bytes  |
| Bits           | Current target in compact format                           | The difficulty is adjusted                  | 4 bytes  |
| Nonce          | 32-bit number (starts at 0)                                | A hash is tried (increments)                | 4 bytes  |

### Possible Implementation
The following is a possible block implementation. It's not exactly Bitcoin's but it should paint a clear picture.
```go
type Block struct {
	Timestamp         int64
	Transactions      []*Transaction
	PreviousBlockHash []byte
	Hash              []byte
	Nonce             int
	Difficulty        int
	Number            int
}
```
## IPLD Bitcoin

### IPLD Bitcoin - Decoding a Block 
1. Read the Binary file as `byte slice`. 
```go
marshalledDataByteSlice, _ := ioutil.ReadFile("block.bin")
```
2. use `hex.DecodeString` to turn the loaded  `byte slice` into hexadecimal byte slice.

```go
	marshalledHexDataByteSlice, _ := hex.DecodeString(string(marshalledDataByteSlice[:len(marshalledDataByteSlice)-1]))
```
3. use `DecodeBlockMessage()` to unmarshall the `byte slice` to `[]node.Node`.the function can be found in  [parsing.go](https://github.com/ipfs/go-ipld-btc/blob/47a103a37d46eb45f23097716a1224ce7b28e379/parsing.go)
Decode block message rebuilds the `block` and everything that it stores as it is defined in [`IPLD Bitcoin Block Struct`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)
```go
	nodes, _ := DecodeBlockMessage(data)
```
- it rebuilds the block struct and put it in `nodes[0]`
- it rebuild the `transaction` struct and put the, in `nodes[1]` 
!-- Confirm --!
- it rebuild a merkle tree and store each of its values into the array items that come after `nodes[1]`

### IPLD Bitcoin - accessing primitive datatypes
after turning the raw data byte slice into `[]node.Node`, primitive types can be accessed directly after explicit casting to the struct it originated from.
1. We know that `nodes[0]` stores the `block`
```go
blockUnmarshalled := nodes[0].(*Block)
```
2. now, one can access all the filds with primitive data types directly such as
```go
blockNonce := blockUnmarshalled.Nonce
```
### IPLD Bitcoin - Using methods on primitive datatypes
#### [IPLD Bitcoin Block Struct](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)

- IPLD uses JSON For serialization
- You can see it resembles Bitcoin's Block fields. 
```go
type Block struct {
	rawdata []byte
	Version    uint32   `json:"version"`
	Parent     *cid.Cid `json:"parent"`
	MerkleRoot *cid.Cid `json:"tx"`
	Timestamp  uint32   `json:"timestamp"`
	Difficulty uint32   `json:"difficulty"`
	Nonce      uint32   `json:"nonce"`
	cid *cid.Cid
}
```
after turning the raw data byte slice into `[]node.Node`, primitive types can be accessed directly after explicit casting to the struct it originated from and functions can be called on them.
1. We know that `nodes[0]` stores the `block`
```go
blockUnmarshalled := nodes[0].(*Block)
```
2. now, one can call any function that accepts that data type such as [`header() `](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go) 
that returns `block header` as `[]byte`
```go
header() 
header := blockUnmarshalled.header()
```
### IPLD Bitcoin - using `cid` 
#### [IPLD Bitcoin Tx Struct](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)
`cid` are used to traverse between hash-based data structures. for instance, `cid` can be used to access a `transaction` stored in the block.
keep in mind that `transaction` has has a separate schema that `block`
```go
type Tx struct {
	Version   uint32     `json:"version"`
	Inputs    []*TxIn    `json:"inputs"`
	Outputs   []*TxOut   `json:"outputs"`
	LockTime  uint32     `json:"locktime"`
	Witnesses []*Witness `json:"witnesses"`
}
```
to access any defined data-structure through `cid`, [`ResolveLink()`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go) is used. 
[`ResolveLink()`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go) function takes a `<path>` as `[]string` .

## [IPLD Bitcoin Block Resolve Function](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)
The main function that allows traversing a path through a block
```go
// Resolve attempts to traverse a path through this block.
func (b *Block) Resolve(path []string) (interface{}, []string, error) {
	if len(path) == 0 {
		return nil, nil, fmt.Errorf("zero length path")
	}
	switch path[0] {
	case "version":
		return b.Version, path[1:], nil
	case "timestamp":
		return b.Timestamp, path[1:], nil
	case "difficulty":
		return b.Difficulty, path[1:], nil
	case "nonce":
		return b.Nonce, path[1:], nil
	case "parent":
		return &node.Link{Cid: b.Parent}, path[1:], nil
	case "tx":
		return &node.Link{Cid: b.MerkleRoot}, path[1:], nil
	default:
		return nil, nil, fmt.Errorf("no such link")
	}
}
```

## [IPLD Node Interface](https://github.com/ipfs/go-ipld-format/blob/master/format.go)
```go
package format
import (
	"context"
	"fmt"

	blocks "github.com/ipfs/go-block-format"

	cid "github.com/ipfs/go-cid"
)

type Node interface {
	blocks.Block
	Resolver

	// ResolveLink is a helper function that calls resolve and asserts the
	// output is a link
	ResolveLink(path []string) (*Link, []string, error)

	// Copy returns a deep copy of this node
	Copy() Node

	// Links is a helper function that returns all links within this object
	Links() []*Link

	// TODO: not sure if stat deserves to stay
	Stat() (*NodeStat, error)

	// Size returns the size in bytes of the serialized object
	Size() (uint64, error)
}
```
## [IPLD Block Interface](https://github.com/ipfs/go-block-format/blob/master/blocks.go)
```go
package blocks
type Block interface {
	RawData() []byte
	Cid() *cid.Cid
	String() string
	Loggable() map[string]interface{}
}
```
## [IPLD Resolver Interface](https://github.com/ipfs/go-ipld-format/blob/master/format.go)
```go
package format
type Resolver interface {
	// Resolve resolves a path through this node, stopping at any link boundary
	// and returning the object found as well as the remaining path to traverse
	Resolve(path []string) (interface{}, []string, error)

	// Tree lists all paths within the object under 'path', and up to the given depth.
	// To list the entire object (similar to `find .`) pass "" and -1
	Tree(path string, depth int) []string
}
```
## [IPLD Link Struct](https://github.com/ipfs/go-ipld-format/blob/master/format.go)
```go
package format
// Link represents an IPFS Merkle DAG Link between Nodes.
type Link struct {
	// utf string name. should be unique per object
	Name string // utf8

	// cumulative size of target object
	Size uint64

	// multihash of the target object
	Cid *cid.Cid
}
```
## [IPLD NodeStat Struct](https://github.com/ipfs/go-ipld-format/blob/master/format.go)
```go
package format
// NodeStat is a statistics object for a Node. Mostly sizes.
type NodeStat struct {
	Hash           string
	NumLinks       int // number of links in link table
	BlockSize      int // size of the raw, encoded data
	LinksSize      int // size of the links segment
	DataSize       int // size of the data segment
	CumulativeSize int // cumulative size of object and its references
}

```


## [IPLD Cid Struct](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)
```go
package cid
type Cid struct {
	version uint64
	codec   uint64
	hash    mh.Multihash
}
```



