# eth_getAddressAppearances

An Ethereum JSON-RPC endpoint specification for getting all EVM appearances of an address

See the WIP draft specification here [EIP-XXXX.md](EIP-XXXX.md). This is not a formal proposal,
if a proposal is made (as an EIP, a note will be left here including a link to the actual
proposal).

## Table of Contents
- [eth_getAddressAppearances](#eth_getaddressappearances)
  - [Table of Contents](#table-of-contents)
  - [TL;DR](#tldr)
  - [Specification](#specification)
  - [Endpoint vs custom javascript tracer](#endpoint-vs-custom-javascript-tracer)
  - [TrueBlocks ecosystem](#trueblocks-ecosystem)
  - [`chifra list <address>` vs `eth_getAddressAppearances`](#chifra-list-address-vs-eth_getaddressappearances)
  - [The role of the JSON-RPC](#the-role-of-the-json-rpc)
  - [Future of trueblocks-core](#future-of-trueblocks-core)
  - [Intermediate measures](#intermediate-measures)
  - [Just use custom javascript tracers / external data sources](#just-use-custom-javascript-tracers--external-data-sources)
  - [Sharded archive node](#sharded-archive-node)


## TL;DR

New archive node endpoint

eth_getAddressAppearances(my_wallet) that returns
- transaction x in block p
- transaction y in block q
- transaction z in block r

Then those transactions can be traced to get everything that transaction did to/from your wallet,
not just the subset of things that appear in transfers/events.

The endpoint could usher in a renaissance of apps that don't need infrastructure providers,
just either:
- Your archive node
- Your portal node (a sharded archive node, in R&D/WIP/alpha)

## Specification

See the [eth_getAddressAppearances JSON-RPC method](EIP-XXXX.md) specification.

## Endpoint vs custom javascript tracer

TrueBlocks pioneered the implementation of a real usable index that gets all the address
appearances. It's appreciation by a range of users highlights that this information is
general purpose and critical for a broad range of applications.

This supports that index is worth creating a dedicated endpoint for (`eth_getAddressAppearances`).

## TrueBlocks ecosystem

Some notes on the components of the TrueBlocks ecosystem which is centered on the provision and
use of the address appearances.

- TrueBlocks (organisation)
- trueblocks-core (software)
    - CLI based software (`chifra <command>`)
    - Responds to various user queries (What transactions did account x appear in `chifra list <address>`. Many other queries, but many use `chifra list` under the hood.
    - Speaks to a local archive node
    - Uses the UnchainedIndex to respond to queries
    - Creates the UnchainedIndex and pins to IPFS. Achieves this via JSON-RPC methods to archive node.
    Parses entire blockchain to detect addresses as they appear. (`chifra scrape`)
- UnchainedIndex
    - Essentially a mapping of `address -> list(block_number, transaction_index)`
    - Distributed data structured addressable over IPFS
    - Updates are coordinated with smart contract
    - Partial shards obtained by bloom filters
    - Contains enough data to be able to serve `eth_getAddressAppearances` responses
- trueblocks-explorer (software)
    - Uses trueblocks-core to display useful historical chain data

So there is
- software that creates the index
- software that distributes the index
- the distributed index data
- software that uses the index via CLI (`chifra <command>`)
- software for graphical exploration

## `chifra list <address>` vs `eth_getAddressAppearances`
Some notes on the components of the TrueBlocks ecosystem and how they relate to this `eth_getAddressAppearances` endpoint

Compare the following hypothetical queries (fabricated data, looking for transactions involving the
deposit contract):
```command
chifra list 0x00000000219ab540356cBB839Cbe05303d7705Fa
```
Returns
```
address    blockNumber    transactionIndex
0x00000000219ab540356cBB839Cbe05303d7705Fa    17000000    10
0x00000000219ab540356cBB839Cbe05303d7705Fa    17000000    15
0x00000000219ab540356cBB839Cbe05303d7705Fa    17000009    3
```

Is equivalent to the proposed:
```command
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc": "2.0", "method": "eth_getAddressAppearances", "params": ["address": "0x00000000219ab540356cBB839Cbe05303d7705Fa"], "id":1}' http://127.0.0.1:8545 | jq
```
Returns
```
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "appearances": [
        {"blockNumber": "0x1036640", "transactionIndices": ["0xa", "0xf"]},
        {"blockNumber": "0x1036649", "transactionIndices": ["0x3"]},
    ],
  }
}
```

The `eth_getAddressAppearances` is designed to produce the same data. Another
mental model is that the UnchainedIndex was
invented because the `eth_getAddressAppearances` didn't exist.

## The role of the JSON-RPC

When getting information from a local node, one makes all queries to the same place:

```
http://127.0.0.1:8545
```
This includes `eth_getBlockByNumber`, `debug_traceTransaction` and others.

The methods are a coordination mechanism between client implementations and node users.

If `eth_getAddressAppearances` was an endpoint one could find appearances, and then call the
same endpoint again, diving into the transactions returned:

1. Start with `my_address` (personal wallet address)
2. Call `eth_getAddressAppearances(my_address)`, receive `(block_id, transaction_id)`
3. Call `eth_getBlockByNumber(block_id)`, received block
4. Call `debug_traceTransaction(transaction_id)`, receive re-execution of transaction (trace)
5. Start inspecting the trace (or multiple traces), learning what the transaction did.
6. Accrue a history of changes across multiple transactions (e.g., token/ether balances over time)

Without the `eth_getAddressAppearances` endpoint, one cannot use the node without first referring
to other data sources.

## Future of trueblocks-core

If nodes provided `eth_getAddressAppearances`, trueblocks could call that endpoint when needed,
and `chifra <command>` would under the hood use that endpoint rather than look up
the local/IPFS UnchainedIndex
```sh
chifra list <address>
```
Importantly, other local decentralised applications could also call `eth_getAddressAppearances`.

## Intermediate measures

As nodes do not support `eth_getAddressAppearances`, one could implement a phased approach as
follows:

1. Modify trueblocks-core too respond to requests to `eth_getAddressAppearances`
2. Modify an execution client too respond to `eth_getAddressAppearances` requests, but
just have them redirect to trueblocks-core (and hence use UnchainedIndex)
3. Experiment and evaluate. Do users find the endpoint useful?
4. Modify the `eth_getAddressAppearances` as desired
5. Implement `eth_getAddressAppearances` natively in the client (no reliance on UnchainedIndex)
6. Implement in other clients

## Just use custom javascript tracers / external data sources

Hand-crafted solutions and external data sources have so far not resulted in truly decentralised
applications.

For local applications to thrive, easy and standardised methods are required.

It seems a little suspicious that one can have someone set up an archive node, but still
not be able to review their own account history without going through extra hoops.

## Sharded archive node

The [Portal Network](https://github.com/ethereum/portal-network-specs). Seeks to shard the
content of an Ethereum node across many users with limited resources. This may even include
full archive node capacity.

The network supports the addition of sub-protocols. For example, history and state are in
different sub-protocols.

To serve `eth_getAddressAppearances`, an index that maps `address -> list(block_number, transaction_index)` is required. This is static data (as the UnchainedIndex highlights). An archive node
can also create this index locally.

Therefore, the portal network could also support this index. As the portal network has
a specific data model, the nature of the index would need to be considered. In the interim,
Portal Network node users can simply run trueblocks-core and use it to access the UnchainedIndex,
which shards the data (via a manifest and bloom filters).

If archive nodes start implementing `eth_getAddressAppearances` natively, then a subprotocol can
be added to the Portal Network to distribute the index data natively.

The result would be a quick-to-start, low resource node that could show the complete history
of the transactions important for an end user. This can even include trustless sharded archive
node that does not require tracing, as shown in the prototype [archors](https://github.com/perama-v/archors), which creates a state proof bundle for every historical block.