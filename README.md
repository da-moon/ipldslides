# IPLD and hash-based data structures

# Table of Contents

- [Overveiw](#overview)

# Overview

IPLD can be used to link different hash-based datastructures with each other. For instance, one can store a Bitcoin block inside a git object and use that Git object to read transaction inside the Bitcoin block. 

A simple use case example is to read a bitcoin binary file and traverse through it and see the transactions inside of it. In Bitcoin network, there are two main separate hash based data structures: `<Blocks>` and `<Transaction>`. IPLD can be used to build a wrapper to traverse and access a `<Block>` or a `<Transaction>`.

# IPLD Building Blocks
IPLD uses [`<Node>`](https://github.com/ipfs/go-ipld-format/blob/master/format.go) and   [`<Block>`](https://github.com/ipfs/go-block-format/blob/master/blocks.go) and [`<CID>`](https://github.com/ipfs/go-cid/blob/master/cid.go) as the corner stone of it's functionalities.

## [`<Node> Interface`](https://github.com/ipfs/go-ipld-format/blob/master/format.go)
`<Node>` is the base interface all IPLD nodes must implement. Essentially, Content that were stored in an arbitary data-structures would be stored as a node, content such as a Bitcoin Block, A Bitcoin Transaction and so on. 
`<Node>` interface forces implementation of the following methods
| Function Name              | Return Data type      | Purpose                                            |
|----------------------------|-----------------------|----------------------------------------------------|
| ResolveLink(path []string) | *Link, []string,error | call `<resolve()>` and assert the output is a link |
| Copy()                     | Node                  | return a deep copy of this node                    |
| Links()                    | []*Link               | return all links within this object                |
| Stat()                     | *NodeStat, error      | Not important                                      |
| Size()                     | uint64, error         | return the size of the serialized object           |
| Resolver                   |                       | An interface that implements other methods         |
| blocks.Block               |                       | An interface that implements other methods         |


## [`<Link> Struct` ](https://github.com/ipfs/go-ipld-format/blob/master/format.go)

Information needed to traverse from one node to another node is stored in `<Link>`.
| Name   | Data type | Purpose                                                     |
|--------|-----------|-------------------------------------------------------------|
| Name   | string    | unique UTF string name of the link. Seems not so important. |
| Size   | uint64    | the cumulative size of target object                        |
| Cid    | *cid.Cid  | Metadata(such as multihash) of the target object            |

## [`<Resolver> Interface` ](https://github.com/ipfs/go-ipld-format/blob/master/format.go)
`<Resolver>` interfaces forces implementation of the following methods
| Function Name                 | Return Data type           | Purpose                                                                                                                                  |
|-------------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Resolve(path []string)        | interface{},[]string,error | resolve a path through this node, stopping at any link boundary and returning the object found as well as the remaining path to traverse |
| Tree(path string, depth int)  | []string                   | lists all paths within the object under `path`, and up to the given `depth`. passing `path=""` and `depth=-1`lists the entire object     |

## [`<Block> Interface` ](https://github.com/ipfs/go-block-format/blob/master/blocks.go)

`<Block>` contains the lowest level of IPLD data structures.A block is raw data accompanied by a `<CID>`. 
`<Block>` interface forces implementation of the following methods
| Function Name |      Return Data type     |
|:-------------:|:-------------------------:|
| RawData()     |           []byte          |
| Cid()         |         *cid.Cid          |
| String()      |           string          |
| Loggable()    |  map[string]interface{}   |

## [`<CID> Struct`](https://github.com/ipfs/go-cid/blob/master/cid.go)

`<CIDs>` are self-describing content identifiers. Essentially, They represent a content's ID. They are used in `<Block>` as a way to uniquely name it and in `<Link>` for accessing the content.
| Name     | Data type    | Description                                                                 |
|----------|--------------|-----------------------------------------------------------------------------|
| version  | uint64       | `<CID>` version                                                             |
| codec    | uint64       | Type of content `<CID>` points to. eg. Bitcoin Block , Ethereum Transaction |
| hash     | mh.Multihash | The hashing algorithm used.                                                 |
> List of all codecs can be found here [`<https://github.com/multiformats/multicodec/blob/master/table.csv>`](https://github.com/multiformats/multicodec/blob/master/table.csv)



# IPLD Bitcoin
## struct types
Bitcoin has many different data structures. Each of those structures needs to get implemented in the code to access and read the data stored in a `Block` binary file.
### Bitcoin Block : [`<Block>` ](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go) 
|    Name    | Data type |
|:----------:|:---------:|
| Version    |   uint32  |
| Parent     |  *cid.Cid |
| MerkleRoot |  *cid.Cid |
| Timestamp  |   uint32  |
| Difficulty |   uint32  |
| Nonce      |   uint32  |
> There is a `cid <*cid.Cid>` field in the code which seems redundant. 

### Transaction :  [`<Tx>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)

|   Name   | Data type |
|:--------:|:---------:|
| Version  |   uint32  |
| Inputs   |  []*TxIn  |
| Outputs  |  []*TxOut |
| LockTime |   uint32  |
> There is a `Witnesses < []*Witness>` field in the code that is used to decode `Segwit` Transactions. That won't be covered in theis document for the sake of simplicity. 

### Input Transaction :  [`<TxIn>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)

|     Name    | Data type |
|:-----------:|:---------:|
| PrevTx      |  *cid.Cid |
| PrevTxIndex |   uint32  |
| Script      |   []byte  |
| SeqNo       |   uint32  |

### Output Transaction :  [`<TxOut>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)

|  Name  | Data type |
|:------:|:---------:|
| Value  |   uint64  |
| Script |   []byte  |

### Transaction Tree :  [`<TxTree>`](https://github.com/ipfs/go-ipld-btc/blob/master/tx_tree.go)

|  Name |  Data type |
|:-----:|:----------:|
| Left  | *node.Link |
| Right | *node.Link |

## Decoding Block Binary into `<[]Node>`
1. read the Binary file  
```go
hexBinaryData, _ := ioutil.ReadFile("fixtures/block.bin")
```
2. Decode the hexadecimal byte slice `<hexBinaryData>` and store it in `<data>`.
```go
data, _ := hex.DecodeString(string(hexBinaryData[:len(hexBinaryData)-1]))
```
3. 	Turn `<data>` byte slice into IPlD `<[]Node>` by calling [`<DecodeBlockMessage(data)>`](https://github.com/ipfs/go-ipld-btc/blob/master/parsing.go)
```go
nodes, _ := DecodeBlockMessage(data)
```
## Direct Primitive Data-Type Access

`<Node>` can be used to run functions on and access fields in Bitcoin blocks and read Transaction values. 
an example is 
```go
timestamp := nodes[0].(*Block).Timestamp
```
Functions can also get called. For instance, you can see the block header by calling [`<HexHash()>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)
```go
header := nodes[0].(*Block).Timestamp
```
## Accessing Fields IPLD Way

there are  two functions for accessing fields IPLD way [`<Resolve>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go) , [`<ResolveLink>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)

### [`<Resolve>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)
This function can be used to traverse through <`Links`> and also access primitive types since it returns an `<interface{}>`. 
<!-- Test -->

```go
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
### [`<ResolveLink>`](https://github.com/ipfs/go-ipld-btc/blob/master/btc.go)

this function is used when you strictly want to access and traverse through links which means Fields with data type `<*cid.Cid>` can **only** get accessed.
```go
func (b *Block) ResolveLink(path []string) (*node.Link, []string, error) {
	out, rest, err := b.Resolve(path)
	if err != nil {
		return nil, nil, err
	}

	lnk, ok := out.(*node.Link)
	if !ok {
		return nil, nil, fmt.Errorf("object at path was not a link")
	}

	return lnk, rest, nil
}
```
> As you can see, it uses `<Resolve()>` to handle the task it is given.



