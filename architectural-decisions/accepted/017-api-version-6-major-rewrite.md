---
Number: 17
Title: API Version 6's major rewrite
Category: Server
Status: accepted
---

<!-- ADR template adapted from Michael Nygard's -->

## Context

<!-- What is the issue that we're seeing that is motivating this decision or change? -->

Ogmios was created circa 2018, prior to the Shelley era, as a way to bridge Cardano to developers. One of Ogmios' main goal has been to maximise the developer experience and ease of interaction with the Cardano blockchain.

True to its core goal, its API hasn't changed much in the course of the parse years. However, having crossed all the Cardano eras, it has accumulated few technical debts. Ogmios often integrate eras and features ahead of time and often, before they properly settle in the ledger and the ecosystem. Hence, some names chosen initially now sound a little awkward (e.g. `extra data hash` instead of `script integrity hash` or, `metadata` vs `auxiliary data`).

In addition, the Byron era which was integrated first and well before the Shelley era came out still present some oddities. While most eras that followed Shelley were built upon Shelley and shared an extensive part of its source code, Byron has been seggregated out and is different. At the time, it wasn't clear that all eras following Shelley would be similar so we did not bother making Byron and Shelley too similar. Now however, Byron feels very much disconnected from other eras.

More, there were also regrettable decisions made from the start regarding choices of serialization in the API. In particular, the use of singleton object instead of discriminated unions made it hard down the line for client to parse data. This particularly the case for recursive structures such as timelocks scripts from the Allegra/Mary eras.

Finally, over time, Ogmios has tried to abstract away the complexity of the hard-fork combinator. While all mini-protocols in Cardano are era-dependent, Ogmios tries as much as possible to be era-independant. Yet, Ogmios would still return different responses based on era sometimes which create a strange duality between the input interface and the server's output. This era separation on the client side also adds complexity as it forces client to consider era differences even when it isn't needed (e.g. accessing a block header hash that is present in ALL eras shouldn't require a complex dance).

## Decision

<!-- What is the change that we're proposing and/or doing? -->

The outer API of Ogmios will be rewritten, aiming to solve the various quirks that have been identified since it was created. In particular, we will follow a few principles while doing so:

1. **The API should maximise consistency.**<br/>
   If two fields have same names, they should refer to the same entity. And vice-versa, two entities that are semantically equivalent should have the same name. Similarly, if a particular approach is
   used for structuring a certain object, it is reasonable to expect any object that are structurally equivalent to also be serialised in the same way.

2. **The API should be as unambiguous and as self-explanatory as possible.**</br>
   This means, for example, avoiding acronyms such as `tx`, `vk`, `blk` and favor their complete alternatives `transaction`, `verificationKey`, `block`. The only two exceptions in the API to this are:

   - id (→ identifier);
   - vrf (→ verifiable random function);

  because they are more often seen in these forms than in their full form. The new API shall also try to avoid ambiguity arising from context. If something is called `hash` or `id`, it should be nested in a parent object that removes any ambiguity. For example: `{ "transaction": { "id": "..." } }`, `{ "header": { "hash": "..." } } `.

3. **The API should favor composability.**<br/>
  By composability, we mean composability within the API itself. Thus, if a method of the API expects a an object `X`, and some other method returns an object `X`, it should be straightforward to lift the result as an input. On a concrete example, this means avoiding coumpound keys such as `transactionId` or `headerHash` and prefer them the form of a partial object (e.g. `{ "transaction": { "id": "..." } }`).
  This also means that whenever possible, the API should try to align the representation of objects that share similar fields. This is particularly the case for blocks and transactions which are now represented in one common format, with optional fields.

## Consequences

<!-- What becomes easier or more difficult to do because of this change? -->

- The API is easier to parse and comprehend overall.

- The JSON-schema definition has been split into two. One file still lives in this repository, while the other has been moved to [CardanoSolutions/cardanonical](https://github.com/CardanoSolutions/cardanonical). The latter contains definitions of ALL data-types present in the Cardano blockchain for reuse across multiple services of the ecosystem.

- Every single client application is now broken :) ... but here is a (hopefully exhaustive) migration guide:

### Migration guide

> **Note**
> There are still many [test vectors](https://github.com/CardanoSolutions/ogmios/tree/master/server/test/vectors) available for every element of the Ogmios API. Use them!

#### <strike>JSON-WSP</strike> → JSON-RPC 2.0

JSON-WSP has been ditched and replaced by [JSON-RPC 2.0](https://www.jsonrpc.org/specification) with which Ogmios is now fully compatible. In particular, this means that request and response payloads are a bit more lightweight.

<table>
<tr>
<th>Old</th>
<th>New</th>
</tr>
<tr>
<td>

```json
{
  "type": "jsonwsp/request",
  "version": "1.0",
  "servicename": "ogmios",
  "methodname": "query",
  "args": { "query": "genesisConfig" },
  "mirror": { "id": "foo" }
}
```

</td>
<td>

```json
{
  "jsonrpc": "2.0",
  "method": "queryNetwork/config",
  "params": { "era": "shelley" },
  "id": "foo"
}
```

</td>
</tr>
</table>

> **Note**
>
> Ogmios' implementation of JSON-RPC 2.0 is _slightly_ more flexible as the specification w.r.t to the `id` field. While the specification indicates that this field should be a `string`, Ogmios will still accept _anything_ as an `id` field (`string`, `number`, `object`, etc..). So it is essentially a drop-in replacement for `mirror`.

##### Requests

All methods names have been adjusted as well. Here's a translation table (<u>beware the casing</u>, methods are now `lowerCamelCase`):

| Old               | New                                         |
| ---               | ---                                         |
| `RequestNext`     | `nextBlock`                                 |
| `FindIntersect`   | `findIntersection`                          |
| ---               | ---                                         |
| `SubmitTx`        | `submitTransaction`                         |
| `EvaluateTx`      | `evaluateTransaction`                       |
| ---               | ---                                         |
| `Acquire`         | `acquireLedgerState`                        |
| `Query`           | `queryLedgerState/*` <br/> `queryNetwork/*` |
| `Release`         | `releaseLedgerState`                        |
| ---               | ---                                         |
| `AwaitAcquire`    | `acquireMempool`                            |
| `NextTx`          | `nextTransaction`                           |
| `HasTx`           | `hasTransaction`                            |
| `SizeAndCapacity` | `sizeOfMempool`                             |
| `ReleaseMempool`  | `releaseMempool`                            |

4. Similarly, many ledger state queries have been renamed. Here's a recap table:

| Old                          | New                          |
| ---                          | ---                          |
| `currentEpoch`               | `epoch`                      |
| `currentProtocolParameters`  | `protocolParameters`         |
| `delegationsAndRewards`      | `rewardAccountSummaries`     |
| `eraStart`                   | `eraStart`                   |
| `eraSummaries`               | `eraSummaries`               |
| `ledgerTip`                  | `tip`                        |
| `nonMyopicMemberRewards`     | `projectedRewards`           |
| `proposedProtocolParameters` | `proposedProtocolParameters` |
| `rewardsProvenance`          | N/A                          |
| `rewardsProvenance'`         | `rewardsProvenance`          |
| `stakeDistribution`          | `liveStakeDistribution`      |
| `utxo`                       | `utxo`                       |

Also, some queries have been moved under `queryNetwork` and are always available, in any era:

| Old             | New                    |
| ---             | ---                    |
| `blockHeight`   | `blockHeight`          |
| `chainTip`      | `tip`                  |
| `genesisConfig` | `genesisConfiguration` |
| `systemStart`   | `startTime`            |

> **Warning**
>
> The `queryNetwork/genesis` local-state-query now expects one era as argument (either 'byron', 'shelley' or 'alonzo') to retrieve the corresponding genesis configuration.

##### Responses

Query responses from the local-state-query protocol are now wrapped under the query name. So for example, querying the ledger tip as:

<table>
<tr>
<th>Request</th>
<th>Response</th>
</tr>
<tr>
<td>

```json
{
  "jsonrpc": "2.0",
  "method": "queryLedgerState/tip",
}
```

</td>
<td>

```json
{
  "jsonrpc": "2.0",
  "result": {
    "tip": {
      "slot": 1234,
      "header": { "hash": "1234567890abcdef" }
    }
  }
}
```

</td>
</tr>
</table>

##### Errors

Errors in the protocol are now returned as JSON-RPC 2.0 errors, with unique error codes. This includes unsucessful operations such as an intersection not found on `findIntersection`, a point not acquired on `acquireLedgerState` or a failed transaction submission / evaluation. Some errors contain details specific to the error. This approach should simplify both parsing and documentation, as each error is now identified by a specific code.

In particular, every single phase-1 or phase-2 validation errors now have a unique error code and description attached to them.

#### Data models

##### Transaction

The transaction model has been greatly reworked. The main changes are:

- The fields previously nested under `body` and `witnesses` have been flattened out and are now part of the top level object (e.g. `inputs` are no longer nested under `body`).

- Few fields have been renamed

  | Old    | New    |
  | ---    | ---    |
  | `TODO` | `TODO` |

- Metadata now only contains user-defined labelled metadata instead of also containing extra scripts. Extra scripts have been moved to the `scripts` field and merged with witness scripts.
  The naming is now also a bit less awkward as `body → blob` for accessing user-defined metadata is now simply `labels`.

##### Block

The block model has also been reworked and merged together into one comprehensive block type. Fields have been renamed, some have been nested and other unnested. Here's a recap:

| Old                   | New                             |
| ---                   | ---                             |
| `hash`                | N/A (removed)                   |
| `header.blockSize`    | `size`                          |
| `header.blockHeight`  | `height`                        |
| `header.slot`         | `slot`                          |
| `header.blockHash`    | N/A (removed)                   |
| `headerHash`          | `header.hash`                   |
| `header.previousHash` | `ancestor`                      |
| `header.opCert`       | `issuer.operationalCertificate` |
| `header.issuerVk`     | `issuer.verificationKey`        |
| `header.issuerVrf`    | `issuer.vrfVerificationKey`     |
| `body`                | `transactions`                  |
| `header.signature`    | N/A (removed)                   |

Byron blocks are also now less "weird" than the rest of the blocks. So few changes concern only (old) Byron blocks to align them with the new model:

| Old                                          | New                        |
| ---                                          | ---                        |
| `header.genesisKey`                          | `issuer.verificationKey`   |
| `header.prevHash`                            | `ancestor`                 |
| `header.signature.signature`                 | N/A (removed)              |
| `header.signature.dlgCertificate.delegateVk` | `delegate.verificationKey` |
| `header.protocolVersion`                     | `protocol.version`         |
| `header.protocolMagicId`                     | `protocol.magic`           |
| `header.softwareVersion`                     | `protocol.software`        |
| `header.proof`                               | N/A (removed)              |
| `body.txPayload`                             | `transactions`             |
| `body.dlgPayload`                            | `operationalCertificates`  |
| `body.updatePayload`                         | `governanceAction`         |

##### Protocol Parameters

TODO

##### Phase-1 (native) scripts

TODO

##### Transaction phase-1 and phase-2 errors

TODO