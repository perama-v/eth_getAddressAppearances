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
    - [Inventory of components](#inventory-of-components)
    - [Comparison of `chifra list <address>` vs `eth_getAddressAppearances`](#comparison-of-chifra-list-address-vs-eth_getaddressappearances)
    - [Future of trueblocks-core](#future-of-trueblocks-core)
    - [Intermediate measures](#intermediate-measures)
  - [The role of the JSON-RPC](#the-role-of-the-json-rpc)
  - [Just use custom javascript tracers / external data sources](#just-use-custom-javascript-tracers--external-data-sources)
  - [Sharded archive node](#sharded-archive-node)
  - [Example applications](#example-applications)
    - [Deposit contract display](#deposit-contract-display)
    - [Wallet receives from exchange batch-send](#wallet-receives-from-exchange-batch-send)
  - [Alternative or symbiotic designs](#alternative-or-symbiotic-designs)
    - [Note on implementation details](#note-on-implementation-details)
    - [Why not eth_addressesPerBlock?](#why-not-eth_addressesperblock)

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

TrueBlocks is the inspiration for this concept. One might view this `eth_getAddressAppearances`
method as an attempt to fold some of the functionality found in trueblocks-core/UnchainedIndex ([https://github.com/TrueBlocks/trueblocks-core](https://github.com/TrueBlocks/trueblocks-core))
into the node where it seems to really belong.

### Inventory of components
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

### Comparison of `chifra list <address>` vs `eth_getAddressAppearances`
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


### Future of trueblocks-core

If nodes provided `eth_getAddressAppearances`, trueblocks could call that endpoint when needed,
and `chifra <command>` would under the hood use that endpoint rather than look up
the local/IPFS UnchainedIndex
```sh
chifra list <address>
```
Importantly, other local decentralised applications could also call `eth_getAddressAppearances`.

### Intermediate measures

As nodes do not support `eth_getAddressAppearances`, one could implement a phased approach as
follows:

1. Modify trueblocks-core too respond to requests to `eth_getAddressAppearances`
2. Modify an execution client too respond to `eth_getAddressAppearances` requests, but
just have them redirect to trueblocks-core (and hence use UnchainedIndex)
3. Experiment and evaluate. Do users find the endpoint useful?
4. Modify the `eth_getAddressAppearances` as desired
5. Implement `eth_getAddressAppearances` natively in the client (no reliance on UnchainedIndex)
6. Implement in other clients


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

## Example applications

### Deposit contract display

A local frontend for a deployed contract. The goal is a local frontend that shows live updates,
as well as historical updates.

This could be any deployed protocol, but for now we will use the Ethereum deposit, contract at an address: `0x00000000219ab540356cBB839Cbe05303d7705Fa`.

Someone has written a program that acts as an interface for a user.
The user has a node accessible over port 8545.
The program can call methods on that node, including `eth_getBlockByNumber` and, in this
scenario, the new `eth_getAddressAppearances`.

The program periodically asks for the latest block to know where the chain head is up to.
Suppose the last query was for block `0x1036000`, and the chain has progressed 3 blocks `0x1036003`.

As new blocks are produced, if the contract appears in a transaction, the interface would like
to know the details of what happened.

The deployed contract emits events for successful deposits, these are easily tracked in transaction
logs. However other interactions with the contract are possible, including deposits that fail
due to running out of gas. To see these, the transaction must be re-executed and examined.

The program calls

```json
{
    "jsonrpc": "2.0",
    "method": "eth_getAddressAppearances",
    "params": [
        "address": "0x00000000219ab540356cBB839Cbe05303d7705Fa",
        "range": ["0x1036001", "0x1036003"]
        ],
    "id": 0
}
```
The response is (made up for this example):
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "appearances": [
        {"blockNumber": "0x1036001", "transactionIndices": ["0x0", "0x2c"]},
        {"blockNumber": "0x1036002", "transactionIndices": ["0x3"]},
        {"blockNumber": "0x1036003", "transactionIndices": ["0x11"]},
    ],
  }
}
```
The program knows that 4 transactions involved the deposit contract somehow.
First the program fetches the transaction receipts, and if they appear normal, no
further work is required, the events can be parsed and the user interface updated.

Suppose transaction "0x3" in block "0x1036002" has no events logged. Now the application
calls to re-execute and examine the transaction using its hash "0xcdef...8765".

```
{
    "jsonrpc": "2.0",
    "method": "eth_debugTraceTransaction",
    "params": ["0xcdef...8765"],
    "id": 0
}
```
The trace returned is parsed and it is discovered that the transaction was reverted.

The display can now be updated, showing the user the 3 successful deposits and the 1 failed
deposit.

The user now clicks on a tab in the application called "my transactions". The program
finds the address from the connected wallet "0xbcde...5555". Then it calls:

```json
{
    "jsonrpc": "2.0",
    "method": "eth_getAddressAppearances",
    "params": [
        "address": "0xbcde...5555",
        "range": []
        ],
    "id": 0
}
```
This returns the list of all transactions relevant to the user. The program finds the
intersection of the two lists (user address and deposit contract). Those transactions
can now be fetched and examined closely.

This pattern applies to all applications: the intersection of a
- eth_getAddressAppearances for protocol address
- eth_getAddressAppearances for user address

Represents interesting transactions that can be fetched, re-executed and displayed.

### Wallet receives from exchange batch-send

A user has a local node and a wallet interface to explore their history. As above,
the program running the display calls a local node. It can query historical transactions,
as well as periodically calling for pertinent transactions at the chain tip.

The program calls the node and finds a list of tranactions. They are fetched and examined for logged
events. One is found that does not have logged events.

It is re-executed using debug_traceTransaction which reveals a CALL from an account to the
users address. As the user account has no code, the result is a transfer of ether to the user.

The frontend displays this transfer, which otherwise is opaque.

This scenario applies to a common pattern, which is a cost saving measure to send ether to
multiple addresses at once. E.g,. from a centralized exchange to multiple users withdrawing
around the same time.

Without `eth_getAddressAppearances`, the local program is unable to show why the users balance
is suddenly higher.

## Alternative or symbiotic designs

### Note on implementation details

A node may implement `eth_getAddressAppearances` differently according to its architecture.
For example, for each address encountered when running the execution phase of the chain
sync, it may store some compressed map for every appearance
(pair of block number and transaction index).

It may be tempting to provide halfway endpoints that help implementations roll out in phases,
this may be counterproductive. An intermediate phase endpoint may imply different representations
in different clients. It seems wiser to decide: is `eth_getAddressAppearances` worth
implementing, or not?

### Why not eth_addressesPerBlock?
It returns too much extraneous data and places too much emphasis on the block, rather than
the user/application address.

Consider a method `eth_addressesPerBlock` that accepts a block number and returns
the addresses that appear in that block.

Consider an application with a contract deployed (let's use the deposit contract application example
again `0x0000...05Fa`). A program talks to the node to find tranasctions that involve the contract.
```
query: eth_addressesPerBlock(block_number)
returns: [address_1, address_2, 0x0000...05Fa, address_3, ...]
```

This is a binary result (the application was invovled somewhere in the block) and now every
transaction must be interrogated.

Alternatively, the transaction index could also be returned:

```
query: eth_addressesPerBlock(block_number)
returns: [(address_1, index_a), (address_2, index_b), (0x0000...05Fa, index_c), (address_3, index_d), ...]
```
Now it is clear that transaction with index_c must be examined to get useful information to display.

This is a lot of information that is very clearly the is not useful to the program creating the
interface for the user. Many addresses are completely irrelevant (there will often be many
addresses appear per transaction, and can be ~50-200 transactions per block).

As transaction with index_c can be re-executed, this re-execution will reveal addresses that
have interacted with the contract (e.g., making a deposit). Some applications have multiple
addresses all in one (such as multi-sends to different participants). The transaction trace
contains all this information.

So even though the method returns the addresses of users who interacted with the contract in
transaction index_c, their discovery actually comes from inspecting that contract. There
will be opcodes (e.g., CALL) whose subjects are the addresses that are the users of that protocol.

Queries for user-centric applications start with the premise: we are interested in activity related
to the address of the user/contract.

It is therefore not fruitful to start by looking for all addresses that appear in a block.
It is mostly extraneous or redundant work. For a full explorer that seeks to display everything
that happened in a block, one must re-execute the block and parse the opcodes to find meaning.
For example, batch ether transfers using the CALL opcode to dozens of EOAs in a single transaction.
Tracing the transaction is likely the only way to correctly gain meaning from such a transaction,
and so the eth_addressesPerBlock method is redundant (the addresses are present on the stack
in the trace for each CALL).

The low utility for a user is also made clear when considering past blocks.
A user likely does not know which block they should be interested in, but eth_addressesPerBlock
either requires this knowledge, and so implies it should be called for all blocks.

In other words, applications can be described as having one of three patterns:
- General user wallet display
    - One call: Here is my address, show me where to go next (which tx to trace)
- General application display
    - One call: Here is the contract address, show me where to go next
- Single user, single application display
    - Two calls
        - Here is my address
        - Here is the application contract address (or addresses)
    - The intersection of the two results is where to go next

`eth_getAddressAppearances` satisfies all these patterns efficiently, and includes the provision
to limit queries to a range of blocks.
