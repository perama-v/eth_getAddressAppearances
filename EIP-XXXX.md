```
title: eth_getAddressAppearances JSON-RPC method
description: An Ethereum JSON-RPC endpoint for getting all EVM appearances of an address
author: Perama (@perama-v)
discussions-to: -
status: Draft
type: Standards
category: Interface
created: 2023-07-22
requires: -
```
## Table of Contents
  - [Abstract](#abstract)
  - [Motivation](#motivation)
  - [Specification](#specification)
    - [Type aliases](#type-aliases)
    - [Design parameters](#design-parameters)
    - [Derived parameters](#derived-parameters)
    - [Address](#address)
    - [Appearance definition](#appearance-definition)
    - [`appearance` type](#appearance-type)
    - [eth_getAddressAppearances method](#eth_getaddressappearances-method)
    - [eth_getAddressAppearances response](#eth_getaddressappearances-response)
  - [Rationale](#rationale)
  - [Backwards Compatibility](#backwards-compatibility)
  - [Test Cases](#test-cases)
    - [Positive detection cases](#positive-detection-cases)
    - [Negative detection cases](#negative-detection-cases)
  - [Reference Implementation](#reference-implementation)
  - [Security Considerations](#security-considerations)
    - [False positives: Appearances return for addresses that do not exist](#false-positives-appearances-return-for-addresses-that-do-not-exist)
    - [False positives: Addresses that do not affect state](#false-positives-addresses-that-do-not-affect-state)
    - [Multiple appearances](#multiple-appearances)
    - [Precompiles](#precompiles)
  - [Copyright](#copyright)


## Abstract

An endpoint that returns a list of transactions that involve a specific account address.
These transactions can be re-executed for accounting purposes. Without such
an endpoint, a user must re-execute all transactions or use a third party provider for basic
accounting.

## Motivation

The Ethereum Virtual Machine (EVM) can alter the state for an account in arbitrary ways.
To perform accounting for an address the EVM can be replayed to identify the changes that have
been made. This results in a history of that account, equivalent to bookkeeping.

Many EVM actions do not result in readily-trackable events (via LOG opcode).
Two illustrative examples are:
- A CALL to an account with no code sends Ether to to that account without. Used to batch send
ether to many accounts efficiently.
- ERC-20 transfers. The ERC-20 standard allows a token to be sent without an associated
event. Although such practice is discouraged, it occurs and is technically valid.

Re-executing the transaction (`eth_debugTraceTransaction`) allows these changes to be
identified. Re-executing all transactions relevant to an address is required for basic
accounting such as a chart tracking ether/ERC-20 balances over time.

The `eth_getAddressAppearances` endpoint returns a list of which transactions are relevant
to an account. This alleviates the burden of a re-executing the entire chain history
for every new account address of interest.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Type aliases

|Name|Example|Description|
|-|-|-|
|Hex-string|`0x0abc`|Hex encoded, 0x-prefixed, leading 0's permitted|
|Hex-number|`0xabcd`|Hex encoded, 0x-prefixed, leading 0's omitted|

### Design parameters
|Name|Value|Description|
|-|-|-|
|`MAX_VANITY_ZERO_CHARS`|`8`|Leading or trailing '0's permitted in an address|

### Derived parameters
|Name|Definition|Value|Description|
|-|-|-|-|
|`MIN_NONZERO_BYTES`|`20 - MAX_VANITY_ZERO_CHARS // 2`|`16`| Smallest possible nonzero component of an address|

### Address

An address is informally defined as 40 characters that
- may include some leading or trailing zeros
(for vanity addresses)
- appears in the EVM 32-byte environment with left padding.

An address has the following formal definition:
- MUST be 20 bytes
- When detected within > 20 bytes, 32 bytes MUST appear with left padding (12 leading zero-bytes)
- MUST begin or end with `MIN_NONZERO_BYTES` non-zero bytes.
This allows for vanity address inclusion of up to `MAX_VANITY_ZERO_CHARS`.
- MUST NOT be a known precompile

### Appearance definition

An appearance is informally defined as the transaction identifier for a transaction
that contains at least one a given address.

An appearance MUST be return by the endpoint if it meets any one of the definitional criteria.

An appearance has the following formal definition, divided into transaction/non-transaction
scenarios:
- Intra-transaction appearance
    - MAY be the transaction sender (transaction "from" field)
    - MAY be the transaction target (transaction "to" field)
    - MAY be in the transaction calldata (transaction "input" field).
    - MAY appear in the context of logs (LOG0, LOG1, LOG2, LOG3, LOG4), including address, topics or data of the log.
    - MAY be the address used in opcodes that have an address field
    (including but not limited to CALL, CALLCODE, STATICCALL, DELEGATECALL, SELFDESTRUCT, CREATE, CREATE2)
    - MAY be in the call-family (CALL, CALLCODE, STATICCALL or DELGATECALL) opcode argument data (such data defined by CALL opcode "argsOffset" and "argsSize" fields)
    - MAY be in the call-family (CALL, CALLCODE, STATICCALL or DELGATECALL) opcode return data (such data defined by CALL opcode "retOffset" and "retSize" fields)
    - MAY be in the create-family (CREATE or CREATE2) data (such as that defined by the CREATE2 opcode "offset" and "size" fields)
- Extra-transaction appearance
    - MAY be the recipient of an uncle reward, or block reward
    - MAY be a consensus layer withdrawl ("address" field in a transaction "withdrawls" object list)

An address appearance is defined as having the following components:
- Block number (hex number e.g., "0x5", "0x53f)
    - MUST be included for any appearance
- Transaction indices
    - MUST be included for any intra-transaction appearance.
    - MUST be omitted (empty list) if an extra-transaction appearance occurs but an intra-transaction does not occur.

### `appearance` type
The `appearance` type is:
|Name|Required|Type|Example|Explanation|
|-|-|-|-|-|
|`blockNumber`|true|Hex-number|"0x2", "0x123abc" |The block number. An unsigned 64-bit integer as a Hex-number|
|`transactionIndices`|true|List of Hex-number|["0x2","2e"]|A list transactions indices for the given block, ordered from earliest to latest|

### eth_getAddressAppearances method


|Parameter Name|Required|Type|Example|Explanation|
|-|-|-|-|-|
|`address`|true|Hex-string|"0x00000000219ab540356cBB839Cbe05303d7705Fa"|The address to get appearances for. Must be 20 bytes|
|`range`|false|List of Hex-number|["0x2", "0x123abc"]|A list of two block numbers [start_inclusive, end_inclusive], representing inclusive range|

```json
{
    "jsonrpc": "2.0",
    "method": "eth_getAddressAppearances",
    "params": ["address": "0x00000000219ab540356cBB839Cbe05303d7705Fa"],
    "id": 0
}
```
With optional range:
```json
{
    "jsonrpc": "2.0",
    "method": "eth_getAddressAppearances",
    "params": [
        "address": "0x00000000219ab540356cBB839Cbe05303d7705Fa",
        "range": ["0x2", "0x123abc"]
        ],
    "id": 0
}
```
### eth_getAddressAppearances response

|Name|Required|Type|Example|Explanation|
|-|-|-|-|-|
|`appearances`|true|array of [`appearance`](#appearance-type) objects|see below|A list of transaction ids in which the address appears, ordered from earliest to latest|

Consider the following hypothetical response where the address appears in three blocks.
- In block "0x22", as two intra-transaction appearances.
- In block "0x2ef" as one extra-transaction appearance (e.g., a block reward)
- In block "0x1234", as one intra-transaction appearance, and one extra-transaction appearance (e.g., a withdrawal).
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "appearances": [
        {"blockNumber": "0x22", "transactionIndices": ["0x0", "0x2c"]},
        {"blockNumber": "0x2ef", "transactionIndices": []},
        {"blockNumber": "0x1234", "transactionIndices": ["0x11"]},
    ],
  }
}
```

No appearances found:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "appearances": [],
  }
}
```

## Rationale

Ethereum allows censorship resistant economic participation.
Anyone can personally create an entry in the shared ledger.

The `eth_getAddressAppearances` method allows censorship resistant accounting.
Anyone can perform personal bookkeeping of the shared ledger.

Clients that implement this method may be more appealing to users than clients that do not.

Users can include:
- Developers who create fast, local interfaces for protocols. The
`eth_getAddressAppearances` can be used for application addresses. The intersection of transactions
that a user address and protocol address allows for efficient starting point for displaying
the history of application (previous swaps/mints/interactions)
- Users with EOAs or smart wallets, including those designed for account abstraction. The endpoint
allows local personal account history exploration.

Clients do not have to implement this as part of the protocol. Eventually all clients may
end up with an implementation in their archive nodes, and the endpoint may be widely used.
At this point, the endpoint might be a candidate for inclusion in to the definition of the
Ethereum protocol.

## Backwards Compatibility

No backward compatibility issues found.

## Test Cases

### Positive detection cases
Deposit contract address in a 32 byte hex string with left padding:
```
"0x00000000000000000000000000000000219ab540356cBB839Cbe05303d7705Fa"
> (true, "0x00000000219ab540356cBB839Cbe05303d7705Fa")
```

Deposit contract address bytes, but shifted left. A valid address is detected,
but is not the deposit contract address:
```
"0x00000000000000000000219ab540356cBB839Cbe05303d7705Fa000000000000"
> (true, "0x219ab540356cBB839Cbe05303d7705Fa00000000")
```

### Negative detection cases

Deposit contract address in a 32 byte hex string with right padding:
```
"0x00000000219ab540356cBB839Cbe05303d7705Fa000000000000000000000000"
> false
```

## Reference Implementation

A 32 byte sequence may be inspected to determine if it meets criteria for an address as follows:

```go
// Source: UnchainedIndex Specification, trueblocks-core@v0.51.0, Go implementation
func potentialAddress(addr string) bool {
 // Any 32-byte value smaller than this number (including precompiles)
 // are assumed to be baddresses. While there are technically a very
 // large number of addresses in this range, we choose to eliminate them
 // in an effort to keep the index small.
 //
 // While this may seem drastic—that a lot of addresses are being excluded,
 // the number is actually a quite small number--less than two out of
 // every 10000000000000000000000000000000000000000000000 20-bytes strings
 // are excluded, and almost every one of these are actually numbers such
 // account balance or number of tokens transferred. It’s worth it.
 small := "00000000000000000000000000000000000000ffffffffffffffffffffffffff"
 // -------+-------+-------+-------+-------+-------+-------+-------+
 if addr <= small {
 return false
 }
 // Any 32-byte value with less than this many leading zeros assumed to be
 // a baddress. (Most addresses are 20-bytes long and left-padded with zeros
 // Note: we’re processing these as strings, so 24 characters is 12 bytes.
 largePrefix := "000000000000000000000000"
 // -------+-------+-------+
 if !strings.HasPrefix(address, largePrefix) {
 return false
 }
 // A large number of what would normally be considered valid addresses
 // happen to end with eight zeros. We’re not sure why, but we identify
 // these as badresses as well in a final effort to lower the size of
 // the index. We’ve seen no obvious ill-effects from this choice.
 if strings.HasSuffix(address, "00000000") {
 return false
 }
 return true
}
```

## Security Considerations

The EVM allows for use of data that has the same structure as an 20 byte address.
The algorithm used to find address appearances ideally minimises missing appearances
(false negatives) and so may include false positives.

### False positives: Appearances return for addresses that do not exist

Some non-address data with a particular encoding may be used in an EVM opcode and
be identified by an implementation serving `eth_getAddressAppearances`.
Algorithms must be flexible enough to detect vanity addresses created by
generating many private/public key pairs to find addresses with long sequence of 0's
(for example the deposit contract `0x00000000219ab540356cBB839Cbe05303d7705Fa`).

Hence, a query for encoded data will return a list of transaction where that data exists.
The data is not an address, so this represents a bloat to the data.

### False positives: Addresses that do not affect state

An EVM opcode may involve an address that does not meaningfully affect state.
For example, an address may be pushed on and off the EVM stack without any other
action for that address.
An implementation serving `eth_getAddressAppearances` may return that transaction,
even though there is no meaningful information from an accounting perspective.

Analysis of the transaction during re-execution will allow such
false positives to be identified.

### Multiple appearances

An address may appear multiple times within a block, as any combination of intra-transaction
and extra-transaction appearances. The method response includes transaction indices
for all intra-transaction indices. It is the responsibility of the caller to check
if the block also containes extra-transaction appearances.

### Precompiles

Existing precompiles are located at addresses `0x01` to `0x09`. They do
not meet definitional criteria and SHOULD NOT be included.

Future precompiles may be deployed at addresses that meet criteria however, SHOULD NOT be included
in response to calls to `eth_getAddressAppearances`.

### Node size

The provision of the endpoint will likely result in an increase in node size. The magnitude of
the index data as a standalone data structure is approximately 80GB at block 17 million.
An archive node might therefore experience a ~4% increase in size (from roughly 2.00TB to 2.08TB).

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).