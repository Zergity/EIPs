---
title: Transaction Event Manifest
description: New transaction type carrying a signed, atomically-enforced declaration of which log events the transaction must emit.
author: Zergity (@Zergity)
discussions-to: https://ethereum-magicians.org/t/transaction-event-manifest
status: Draft
type: Standards Track
category: Core
created: 2026-05-16
requires: 1559, 2718, 2930, 7702, 7981
---

## Abstract

Introduces a new transaction type carrying an event manifest: a signed, enforced declaration of which log events the transaction must emit and — for the contracts it covers — the only events those contracts may emit. The EVM checks the manifest at the end of execution; mismatch reverts the transaction.

Reuses and extends the EIP-2930 `access_list` field. Storage slots keep their advisory warming semantics; entries gain an event constraint that declares required, permitted, or forbidden emissions.

## Motivation

Today, a user signing a transaction commits to calldata, not to outcomes. Their wallet simulates the call and shows what it expects to happen, but nothing in the protocol binds execution to that expectation. If the contract behaves differently — due to a malicious frontend, a proxy upgrade, state drift between sign and inclusion, or a routing change — the user has no recourse.

Existing mitigations are partial: protocol-specific slippage checks (`amountOutMin`), wallet-side simulation, off-chain transaction screening. None generalize and none are enforced by the protocol.

This EIP introduces a primitive for the signer to declare, atomically with the transaction, what observable outcomes must hold. Outcomes are expressed as predicates over emitted events — the EVM's existing, ABI-typed, layout-independent signal for "something happened." The manifest is enforced at the end of execution; if any declared event is missing or any predicate fails, the transaction reverts.

The same primitive composes with ERC-7683 cross-chain intents (the manifest expresses the destination-chain outcome) and ERC-7730 clear signing (the manifest is what the wallet renders to the user).

### Design Goals

The manifest is designed to satisfy three properties simultaneously. Several of the spec's choices look over-engineered or under-engineered in isolation but are forced by the need to satisfy all three at once.

#### 1. Anti-phishing: no unwanted or unexpected emission

A signed manifest is binding on the entire transaction's emissions. A phishing site cannot reroute the user's signed call through an unexpected helper contract, smuggle an extra `Approval` past the intended `Swap`, substitute a different recipient, or have the same event fire twice when the user authorized one. Every observable side effect of the transaction is enumerated in the manifest at sign time and re-checked by the EL before state is committed.

The mechanisms that deliver this property:

- **Global emitter whitelist.** Any contract that emits a log must appear in the access list with event items. An undeclared address emitting anything reverts the transaction. This closes the "reroute through an unexpected helper" bypass.
- **Bijective per-address matching.** For each declared address, every emitted log must pair 1:1 with a declared item. This closes the "smuggle an extra event" bypass and the "fire the same event twice for one signed item" bypass.
- **Predicate-pinned indexed and data parameters.** Each item constrains topics and data via inclusive ranges. This closes the "substitute a different recipient or amount" bypass.

#### 2. Cardinality flexibility: events may fire more than once

Real interactions emit the same event multiple times. A batched transfer emits `Transfer` per recipient; a multi-hop swap emits `Swap` per pool; an `ERC721` mint may emit one `Transfer(0, to, tokenId)` per minted token. The manifest expresses an `N`-fold occurrence by declaring the corresponding event item `N` times; each declaration consumes exactly one emission under the bijection.

This keeps the grammar minimal — no separate count or multiplicity field, no count predicate language — at the cost of larger manifests for large `N`. The tradeoff is deliberate: every authorized emission costs one signed entry, so a user auditing the manifest can answer "how many `Transfer`s am I authorizing?" by counting items, not by interpreting a count expression. A future EIP MAY add explicit multiplicities (e.g. `[topic_predicates, data_predicates, [min, max]]`) if the size cost of repeated items proves limiting in practice.

#### 3. Order independence: emissions may occur in any order (when unambiguous)

For a manifest whose items have pairwise non-overlapping predicates — the typical case, where each declared event matches a distinct log — emission order is irrelevant. A swap that emits `Approval` before `Swap` matches the same manifest as one that emits them in reverse, because each log can only be consumed by one declaration. Decoupling the manifest from execution-path ordering is necessary because sub-call order is a routing-level detail the signing user has no reason to fix; a routing change between sign-time and inclusion (a different pool, a different solver path) should not invalidate the signature when every emission is still individually authorized.

Declaration order matters in two cases, both narrow:

- **Multi-fire** (the same predicate declared `N` times): the duplicate declarations are interchangeable under the match, so the user can list them in any order they like; the outcome is the same.
- **Multi-matched** (two or more items with overlapping predicates — e.g. a specific item and a wildcard at the same signature): the verifier consumes items greedily in declaration order, so a log that satisfies both items is consumed by the one declared first. Wallets SHOULD declare specific items before broader ones; otherwise an emission order that delivers the broad log before the specific one will fail (the specific item gets consumed first and the broad item then has no log to claim, or vice versa).

In the multi-matched case, emission order can co-determine the outcome together with declaration order — strict bipartite matching could find a valid pairing where greedy does not. The spec uses greedy because it is implementable as a single pass per `LOG` opcode and matches how users reason about "the next event consumes the next applicable declaration." Wallets that want emission-order independence MUST keep predicates pairwise non-overlapping.

Order independence (where it holds) does not weaken anti-phishing: the bijection requirement is per-item exact. A reordering that introduces a new event or duplicates an existing one cannot find a matching and reverts.

#### How the goals interact

These three pull against each other:

- Strict anti-phishing wants to constrain everything, including emission order and cardinality.
- Multi-fire wants the user to authorize "this event, multiple times" without enumerating each occurrence.
- Order independence wants the spec to ignore sequencing entirely.

The chosen primitives — per-address global coverage, per-item bijection, value-range predicates over topics and data, ordered tuples for items but unordered set-matching for items-to-logs — form a minimal set that delivers all three. Granting any further freedom (e.g. allowing the same item to discharge multiple logs, or allowing an undeclared address to emit anything) would erode anti-phishing; requiring any further constraint (e.g. order-preserving matching, or one signed entry covering N occurrences) would erode flexibility.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### New Transaction Type

A new EIP-2718 transaction type `TX_TYPE` with payload:

```
TransactionPayload = rlp([
  chain_id,
  nonce,
  max_priority_fee_per_gas,
  max_fee_per_gas,
  gas_limit,
  to,
  value,
  data,
  access_list,
  authorization_list,
  y_parity, r, s
])
```

All fields except `access_list` have the same meaning as in EIP-7702. `TX_TYPE` is a single byte to be assigned by EIP editors at inclusion; throughout this document it appears as a placeholder symbol, not as a fixed value. Implementations and tests SHOULD parameterize on it and not hard-code a numeric assignment.

### Access List Grammar

```
access_list = [entry, entry, ...]

entry = [address, storage_slots, event_constraint]   # always length 3

storage_slots    = [bytes32, bytes32, ...]            # possibly empty (no slots warmed)

event_constraint = []                                  # empty list   → any-emit (no constraint)
                 | [item, item, ...]                   # non-empty list → strict bijection
                 | 0x                                  # empty string → forbid: MUST NOT emit

item              = [topic_predicates, data_predicates]
topic_predicates  = [param_predicate, ...]            # 0..4 entries
data_predicates   = [param_predicate, ...]            # 0..N entries

param_predicate = bytes32                              # shorthand for a single-member union of one exact value
                | [member, member, ...]                # union; [] = wildcard
member          = bytes32                              # exact match X (shorthand for [X, X])
                | [bytes32, bytes32]                   # inclusive range [min, max]
```

Each entry is exactly three positional elements: an address, a list of storage slots to warm, and an event constraint declaring what (if anything) the address may emit. Storage warming and emission constraint are independent: any combination of (zero or more slots) and (any-emit / forbid / bijection) is valid.

The three forms of `event_constraint` are RLP-distinguishable by type and length:

| Encoding | RLP head byte | Reads as | Meaning |
|---|---|---|---|
| `[]` (empty list) | `0xc0` | "no constraint" | Any-emit: this address MAY emit any logs; emissions are not bijection-matched. |
| `[item, item, ...]` (non-empty list) | `0xc1+` | "exactly these" | Strict bijection: this address MUST emit exactly these logs, 1:1 with items, under §"Matching". |
| `0x` (empty byte string) | `0x80` | "forbid" | This address MUST NOT emit any log; any emission triggers the same revert as an undeclared emitter. |

A non-empty byte string at the `event_constraint` position is invalid; implementations MUST reject any entry whose third element is a string other than `0x`. The empty-string sentinel is the natural opposite of the empty-list wildcard at the same position: both encode in a single RLP byte, differ only in RLP's type-tag bit, and read together as "list = constrain by enumeration; string = constrain by prohibition." No new sentinel bytes, no magic values.

The term `entry` always refers to a top-level 3-tuple. The term `item` is reserved for an element of `event_constraint`'s list form. The term `member` is reserved for elements of a union inside a `param_predicate`.

#### Storage slots

The `storage_slots` list is advisory, byte-for-byte identical in semantics to the EIP-2930 access list. Each slot is added to `accessed_storage_keys` for warming at transaction start; subsequent `SLOAD`/`SSTORE` against the slot pay warm-access prices. There is no post-execution check on storage and storage warming grants no emission rights.

#### Event items

A log matches an event item if all of:

1. `topic_predicates.length == log.topics.length` (i.e. the item targets the same `LOGn` opcode that emitted the log).
2. For each `topic_predicates[i]`, `log.topics[i]` satisfies the predicate.
3. `log.data.length` is a multiple of 32 and `data_predicates.length == log.data.length / 32`.
4. For each `data_predicates[i]`, the i-th 32-byte word of `log.data` satisfies the predicate.

The equality in rule 3 is intentional and parallels the equality in rule 1: it forces every data word the contract emits to be covered by a predicate (wildcard or otherwise), closing the trailing-data-smuggling vector that a `<=` rule would leave open. A contract emitting fewer or more words than the item declares causes the item to fail to match the log, and the verifier falls through to the next item (or reverts if no item matches).

A `LOG` whose `log.data.length` is not a multiple of 32 — possible from hand-written assembly or non-Solidity contracts — fails rule 3 against every item with non-zero `data_predicates` and matches no event item; if the emitting address is declared with at least one event item, the transaction reverts as for any other unmatched log. Wallets SHOULD NOT author items intended to match unaligned data.

`data_predicates` has no explicit upper bound on length; the per-byte intrinsic cost (`ACCESS_LIST_DATA_COST = 64`, see §"Gas Costs") makes degenerate manifests economically infeasible long before the verifier's per-emission scan becomes a concern. Wallets are free to declare items targeting events with hundreds of data words (e.g. large `bytes` blobs) as long as the user is willing to pay the intrinsic charge.

A predicate is satisfied as follows:

- Bare `bytes32` value `X` — satisfied iff the value equals `X`. Canonical encoding for a single exact match; equivalent to `[X]` and to `[[X, X]]`, ~35 bytes cheaper than the latter.
- List form `[member, member, ...]` — satisfied iff the value, interpreted as `uint256`, is admitted by at least one member, where each member is either:
    - `X` (bytes32) — admits `value == X`; equivalent to `[X, X]`.
    - `[min, max]` (2-list of bytes32) — admits `min <= value <= max`.
- Empty list `[]` — wildcard (always satisfied).

RLP disambiguates at every level. At the predicate position: a string is the bare-exact shorthand, a list is a union. Within a union: each member is a string (exact) or a 2-list (range). Mixed unions are natural — `[X, [Y, Z]]` admits "exactly `X`, or anywhere in `[Y, Z]`".

**Authoring caveat — union vs. range.** At the predicate position, a 2-list of bare `bytes32` values is a *union of two exacts*, not a range. To declare a range `[min, max]`, the 2-list must be wrapped in an outer union list:

| Encoding | Meaning | Admits |
|---|---|---|
| `[A, B]`     | union of two exacts                | `value == A` or `value == B` |
| `[[A, B]]`   | union of one range                 | `A <= value <= B` |
| `A`          | bare-exact shorthand (canonical)   | `value == A` |
| `[A]`        | union of one exact (equivalent to bare) | `value == A` |

The two forms `[A, B]` and `[[A, B]]` are visually one bracket apart but match disjoint sets when `A != B` and the open interval `(A, B)` is non-empty. Wallets and tooling MUST surface the chosen form to the user; manifest authors SHOULD prefer the bare form for single exacts, the flat list form for small enumerations, and the wrapped form only when a range is intended. Test Case 11 exercises this distinction.

The wallet SHOULD use the most compact form available:

- Single exact: bare `bytes32` `X` rather than `[X]` or `[[X, X]]`.
- Multiple exacts: `[X, Y, Z]` rather than `[[X, X], [Y, Y], [Z, Z]]`.
- A single range stays `[[min, max]]`.

Exact matches dominate (signatures, addresses, token IDs); the compact encodings save ~37 bytes per pinned value, which compounds across a manifest — at `ACCESS_LIST_DATA_COST = 64` gas/byte, a fully-pinned LOG4 item (4 topics) drops by ~148 bytes ≈ 9500 gas versus the all-range encoding.

The length of `topic_predicates` implicitly encodes the LOG opcode the item targets:

| `topic_predicates.length` | Matches | Typical use |
|---|---|---|
| 0 | `LOG0` only | Anonymous events with no topics |
| 1 | `LOG1` only | Anonymous events with 1 topic, or a named event with no indexed parameters (signature pinned via `topic_predicates[0]`) |
| 2 | `LOG2` only | Named event with 1 indexed parameter |
| 3 | `LOG3` only | Named event with 2 indexed parameters |
| 4 | `LOG4` only | Named event with 3 indexed parameters |

`topic_predicates` MUST have at most 4 entries. For named events, the signature occupies `log.topics[0]` and is pinned by `topic_predicates[0]` — typically the bare-`bytes32` shorthand `sig` (equivalent to `[[sig, sig]]`). For anonymous events, `topic_predicates[0]` is whatever predicate the user wants on the first user-defined topic (wildcard if unconstrained).

To match the same event signature across multiple LOG levels (rare; Solidity events have a fixed indexed-parameter count fixed by their ABI), declare one item per level.

Anonymous events warrant elevated authoring care. An anonymous `LOG0` carries no topics at all (`topic_predicates.length == 0`) — there is no signature topic to discriminate which anonymous event was emitted, so two unrelated anonymous events with the same data shape are indistinguishable to the matcher. The data predicates are the only handle. Wallets and dapps SHOULD pin every data word that carries semantic meaning when authoring items against anonymous `LOG0`/`LOG1` emitters, and SHOULD surface to the user that the discrimination is data-only.

#### Dynamic ABI types

Predicates always operate on raw 32-byte words, not on decoded ABI values. For events containing dynamic-length parameters (`string`, `bytes`, `T[]`, `tuple` with dynamic fields), this has three consequences the wallet MUST handle:

- **Indexed dynamic parameters.** Solidity hashes the value: `log.topics[i] == keccak256(value)`. To pin a known string `s`, the wallet sets the corresponding topic predicate to `keccak256(s)`; ranges over dynamic values are meaningless.
- **Non-indexed dynamic parameters in data.** Each occupies one head word (an offset, in bytes, into `log.data`) followed by tail words elsewhere in the data (a length word plus content words, padded to 32 bytes). Offsets shift whenever any earlier dynamic tail changes size, so they are routing-level artifacts rather than user intent — wildcard them.
- **Total data length.** Rule 3 above requires `data_predicates.length` to equal `log.data.length / 32`. For events with dynamic parameters, the total word count depends on the dynamic-value sizes at emission time. The wallet MUST derive this from a full simulation trace and declare a predicate for every word, wildcarding the offset, length, and content words it does not pin. If state drift between sign-time and inclusion changes any dynamic-value length, the word count changes, the match fails, and the transaction reverts — the same drift-induced revert mode as for any other field.

A future EIP MAY add a "tail wildcard" predicate to remove the exact-word-count coupling for dynamic events. The current spec deliberately omits it to keep the matching rule single-pass and uniform.

#### Matching

For each address with one or more event items declared, the manifest requires a **bijection** between the address's event items and the logs emitted by that address during the transaction: a 1:1 pairing in which each item is matched by exactly one log and each log is consumed by exactly one item. The pairing is constructed greedily in declaration order — see §"Enforcement" for the operational procedure.

Consequences of the bijection:

- The number of logs emitted by `address` MUST equal the number of event items declared for `address`.
- Each event item is satisfied by exactly one log (inclusion).
- Each log is consumed by exactly one event item (exclusivity, with no duplicate-matching).
- To permit `N` logs satisfying the same predicate, the manifest MUST declare the item `N` times. Identical items are interchangeable under the greedy match.
- When two items in the same address group could match the same log (overlapping predicates), the one declared earlier wins. Wallets SHOULD declare specific items before broader ones to avoid a specific log being consumed by a broader item and leaving the specific item unmatched.

**The manifest is the complete declaration of every emission in the transaction.** Every emission must be authorized by the entry covering its emitter:

- An address absent from the access list MUST emit zero logs; any emission causes the transaction to revert (undeclared-emitter rule).
- An address present with `event_constraint = 0x` MUST emit zero logs; any emission causes the transaction to revert (forbid rule).
- An address present with `event_constraint = []` MAY emit any logs; emissions are authorized but not bijection-matched.
- An address present with `event_constraint = [item, ...]` MUST emit exactly the declared logs in 1:1 correspondence with items, under the bijection above.

The forbid form (`0x`) and address-omission are semantically equivalent at runtime; the forbid form exists to let a wallet warm storage slots on an address while still prohibiting all emissions from it.

### Enforcement

Enforcement is performed inline with execution, not as a post-execution pass:

1. At transaction start, for each entry whose `event_constraint` is a non-empty list, initialize an ordered queue of unconsumed items in declaration order.
2. Whenever a `LOG0..LOG4` opcode emits a log, classify the emitting address by its access-list entry and act:
   a. **No entry**, or **entry with `event_constraint = 0x` (forbid)**: record a *manifest-violation marker* against the current call frame and continue execution (see §"Frame journaling" below).
   b. **Entry with `event_constraint = []` (any-emit)**: no action; the emission is authorized and not bijection-matched.
   c. **Entry with `event_constraint = [item, ...]` (bijection)**: scan the queue in declaration order and consume the first item the log satisfies. Remove that item from the queue and journal the consumption against the current frame. If no item in the queue matches, record a manifest-violation marker against the current frame and continue execution.
3. At the end of transaction execution, immediately before committing state changes, both of the following MUST hold; if either fails, the transaction reverts as a top-level exceptional halt:
   - Every declared address's queue is empty.
   - No manifest-violation marker survives in the outermost frame's journal.

**Frame journaling.** Item consumption and manifest-violation markers are journaled per call frame, in the same way storage writes and emitted logs are journaled. When a frame reverts (via the `REVERT` opcode, exhausted gas, or any other in-frame exceptional halt) — and the EVM correspondingly discards the logs emitted within that frame — the verifier MUST also discard every manifest-violation marker recorded within that frame and restore each item consumed within the frame to its original position at the head of the relevant address's queue, in the original declaration order. When a frame commits, its violation markers and queue mutations fold into the parent frame's journal.

This makes the manifest a deterministic function of the *surviving* log set only: an emission inside a frame that subsequently reverts is invisible to the EVM and equally invisible to the verifier. Implementations MAY perform the matching eagerly at each `LOG` opcode (recommended; the typical no-revert path detects malformed manifests on the first violating emission and avoids wasting further compute) or lazily over the surviving log set at end-of-execution. Both modes MUST produce identical accept/reject outcomes for every transaction; a violation discovered eagerly inside a frame that later reverts MUST be discarded along with that frame's other journaled state.

**Top-level exceptional halt.** A manifest violation that survives to the outermost frame's commit — any surviving violation marker, or any non-empty queue at end of execution — is a transaction-level exceptional halt analogous to out-of-gas: the entire transaction's state changes are discarded, the sender pays gas up to the point of detection, and no returndata is produced. The halt is NOT a frame-level `REVERT` and cannot be caught by a parent `CALL`/`STATICCALL`/`try` — once a marker is folded into the outermost frame's journal it can no longer be discarded by any subsequent revert. Storage warming by itself triggers no halts and no journaled markers.

### Gas Costs

Pricing follows the [EIP-2930](./eip-2930.md) per-item model plus [EIP-7981](./eip-7981.md)'s flat per-byte surcharge on access-list data. One new per-item constant, `ACCESS_LIST_EVENT_COST`, is introduced for event items; the per-byte rate is unchanged from EIP-7981.

```
intrinsic_cost +=
    ACCESS_LIST_ADDRESS_COST      * num_addresses
  + ACCESS_LIST_STORAGE_KEY_COST  * num_storage_items
  + ACCESS_LIST_EVENT_COST        * num_event_items
  + ACCESS_LIST_DATA_COST         * access_list_data_bytes
```

Under the 3-tuple entry grammar:

- `num_addresses` is the number of entries.
- `num_storage_items` is the total length of every entry's `storage_slots` list.
- `num_event_items` is the total length of every entry's `event_constraint` when in its non-empty-list form (any-emit `[]` and forbid `0x` contribute zero).
- `access_list_data_bytes` is computed per entry and summed:

  ```
  access_list_data_bytes = Σ_entries (20
                                    + 32 * len(storage_slots)
                                    + rlp_size(event_constraint))
  ```

  Addresses and storage keys retain EIP-7981's payload-only count (20 bytes per address, 32 bytes per storage key, framing not charged). The `event_constraint` field is charged its **full RLP-encoded byte length**, including all framing (outer list header, per-item list headers, predicate-union list headers, range 2-list headers, and member string headers). This is intentional and differs from the addresses/storage-keys rule: predicate trees can nest several levels deep, so framing is a meaningful fraction of the on-wire footprint and must be priced to prevent gaming via deep wrappers.

  Concretely:
  - Any-emit `[]` contributes 1 byte (`0xc0`).
  - Forbid `0x` contributes 1 byte (`0x80`).
  - A non-empty event-constraint list contributes its full RLP encoding. A bare `bytes32` predicate value contributes 33 bytes (`0xa0` prefix + 32 payload); a range `[[A, B]]` at a predicate position contributes 70 bytes (the figure exercised by Test Case 11). A fully-pinned `LOG4` event item — four bare-bytes32 topic predicates plus `N` bare data-predicate words — contributes roughly `1 + (1 + 4 * 33) + (1 + N * 33)` bytes of framing-plus-payload.

  This split keeps EIP-7981's address/slot accounting bit-for-bit identical for the legacy fields while giving predicates a regime that scales with their actual wire size.

| Constant | Value | Source |
|---|---|---|
| `ACCESS_LIST_ADDRESS_COST`     | 2400 | EIP-2930 |
| `ACCESS_LIST_STORAGE_KEY_COST` | 1900 | EIP-2930 |
| `ACCESS_LIST_EVENT_COST`       | 1900 | this EIP |
| `ACCESS_LIST_DATA_COST`        | 64   | EIP-7981 |

`ACCESS_LIST_EVENT_COST` is set equal to `ACCESS_LIST_STORAGE_KEY_COST` rather than higher, despite event items imposing runtime work (per-`LOG` queue scan + end-of-tx sweep) that storage items do not. The justification is that the runtime work is bounded by the declared item count, which is itself paid for at intrinsic time: each declared item costs `ACCESS_LIST_EVENT_COST` plus its per-byte data cost, so a manifest large enough to make the verifier work non-trivial has already been charged proportionally. The per-`LOG` scan is also linear in the per-address queue depth, not in the global item count — and per-byte costs (`ACCESS_LIST_DATA_COST = 64`) dominate the encoding of any non-trivial predicate (a fully-pinned `LOG4` item runs ~140 data bytes ≈ 9000 gas, dwarfing the 1900-gas item base). Implementations whose benchmarks find the per-emission scan dominates SHOULD propose a separate per-emit surcharge in a follow-up; the current pricing reflects the expectation that intrinsic charging plus per-byte costs already cover verifier work.

## Backwards Compatibility

Existing transaction types (`0x01`, `0x02`, `0x04`, and any other type pre-dating this EIP) keep the EIP-2930 access list grammar (entries are 2-tuples `[address, [slot, ...]]`). Their semantics are unchanged.

The new transaction type `TX_TYPE` (single byte, assigned by EIP editors at inclusion) uses the 3-tuple grammar above. The 3-tuple shape is **not** RLP-compatible with the EIP-2930 2-tuple shape: a `TX_TYPE` transaction MUST encode every entry with three positional elements, and a legacy access list cannot be reused byte-for-byte in a `TX_TYPE` transaction.

A type-`TX_TYPE` transaction whose entries all carry `event_constraint = []` imposes no constraint on emissions and is observationally equivalent to a type-`0x02` transaction with the same storage slots, modulo the new type byte and the intrinsic gas line item. A `TX_TYPE` transaction with an empty `access_list` is well-formed and forbids all emissions for the entire transaction: any `LOG` opcode from any address causes an immediate revert under the undeclared-emitter rule. This is the maximally restrictive manifest and is the natural mode for transactions whose authorized observable effect is "nothing emits."

### Legacy access list promotion

Wallets that promote a legacy access list into a `TX_TYPE` transaction MUST widen each entry `[ADDR, [K, ...]]` to the any-emit form `[ADDR, [K, ...], []]`. Silently widening to the forbid form `[ADDR, [K, ...], 0x]` would convert a permissive legacy entry into a restrictive manifest constraint and almost always cause unintended reverts; the forbid form MUST be surfaced only when the user (or dapp) explicitly authors it.

The two 3-tuple shapes look almost identical but mean opposite things:

| Form | Tx type | Manifest applied? | Means |
|---|---|---|---|
| `[ADDR, [K]]`        | legacy (0x01 / 0x02 / 0x04) | no                  | warm `K`; emissions unrestricted (no manifest exists) |
| `[ADDR, [K], []]`    | `TX_TYPE`                   | yes                 | warm `K`; any-emit (manifest declares no constraint on `ADDR`) |
| `[ADDR, [K], 0x]`    | `TX_TYPE`                   | yes                 | warm `K`; **forbid** all emissions from `ADDR` |

The legacy 2-tuple maps semantically onto the **3-tuple any-emit form**, not onto the forbid form.

### Wallet rendering

Wallets MUST render any-emit entries (those with `event_constraint = []`) as a distinct, prominent class — not as fine-print bullets — because an any-emit entry authorizes the address to emit any logs without further constraint. Suggested copy: "⚠ Contract `ADDR` is authorized to emit any events." This is a load-bearing UX rule: the on-chain shape is meaningless if a phishing dapp can slip an any-emit entry past a user who expected a bijection. Forbid entries SHOULD also be rendered explicitly, as a positive declaration that no emissions are permitted from the address.

## Rationale

### Why events, not storage

Events are protocol authors' intentional declarations of what happened. They are ABI-typed, named, and stable across implementation changes (proxy upgrades, internal refactors, storage layout shifts). Storage slots are implementation details that vary between contract versions and require per-contract metadata to interpret meaningfully.

For a manifest expressing user intent — the layer this EIP targets — events are the right unit. Storage-level constraints would couple the manifest to contract internals the user has no reason to know. Three specific obstacles to a storage-based manifest:

- **Pre-image opacity.** Mapping and dynamic-array slots are addressed by `keccak256(key . base_slot)` (and nested hashes for mappings of mappings). To target "the balance of `0xAlice`" the manifest would have to carry the pre-image (`key`, `base_slot`) and require the EL to compute the hash, or carry the already-hashed slot and rely on off-chain tools to derive it. Either way the human-readable thing the user signs (an address, a token ID) is one keccak removed from the thing the EVM actually checks, and any layout-version skew between sign-time and inclusion silently retargets the slot.
- **Language- and compiler-specific layouts.** Solidity, Vyper, Huff, and hand-written assembly each lay out state differently — Solidity packs from slot 0 upward with right-aligned values, Vyper uses different defaults for dynamic types, and many production contracts (proxies, diamonds, upgradeable storage) deliberately namespace storage at arbitrary base slots. Encoding a manifest against any of these requires the wallet to know not just the contract's ABI but its compiler, version, and storage namespacing convention. Events have none of this coupling: a `Transfer(address,address,uint256)` signature means the same thing across every language and layout.
- **Sub-slot packing.** A single 32-byte slot frequently encodes multiple values — a `uint128` balance and a `uint128` lastUpdate packed together, or boolean flags bit-packed across a `uint256`. Predicating on "did the balance exceed X" requires the manifest to specify a bit offset, width, and signedness in addition to the slot index, and to mirror the contract's chosen packing exactly. A compiler upgrade or struct reordering that preserves the contract's external behavior silently invalidates every storage manifest written against the old packing. Events expose decoded, individually-typed parameters and avoid this entirely.

These are not theoretical: each one shifts the manifest's burden from "what the user wanted to happen" to "how this specific contract was compiled," which is exactly the coupling this EIP exists to avoid.

### Why reuse `access_list`

The field already carries the right shape (address-grouped, signed under the transaction, propagated by all clients) and the right pricing model (per-entry plus per-byte). Adding a parallel field would duplicate the structure.

The name "access list" stops being descriptive — events are not accesses — but the structural alignment outweighs the cosmetic loss.

### Why one signature

The manifest is part of the transaction payload, covered by the existing transaction signature. The user signs once; the manifest is bound to the calldata and the rest of the transaction parameters. No second signature, no separate verification step. The manifest signer is, by construction, the transaction signer.

Three use cases want the two roles separated:

- **Sponsored gas with outcome authority.** Alice authorizes outcomes (signs a manifest); Bob constructs the calldata and pays gas to submit.
- **Intent / solver flows.** A user signs an off-chain intent (including a manifest expressing the desired outcome); a solver constructs the calldata that produces that outcome.
- **Multi-party authorization.** A treasury operator signs the calldata for routine operations; a security officer signs the manifest entries authorizing which counterparties and amounts are acceptable.

The first two are already handled at the **application layer** without any EL change. ERC-4337 bundlers separate the user's `UserOperation` signature (which covers the manifest the user authored) from the bundler's EOA-tx signature, with the validator enforcing inside `validateUserOp` that the access list of the submitted transaction equals the manifest the user signed. ERC-7683 orders work the same way: the user signs an order containing the manifest, the destination-chain filler signs the fill transaction, and the settler verifies the order's manifest equals the on-chain access list. Permit-style relayers can apply the same pattern with a custom verifier contract.

A direct EL-level extension is sketchable but deliberately out of scope for this EIP. The shape would be an additional signature attached to the transaction, analogous to EIP-7702's `authorization_list`:

```
TransactionPayload = rlp([
  ..., access_list,
  manifest_authority = [chain_id, manifest_nonce, y_parity, r, s],
  authorization_list, y_parity, r, s
])
```

The signature covers `keccak256(rlp(chain_id, manifest_nonce, access_list))`. The EL recovers an authority address `A`, advances `A`'s manifest-specific nonce (its own replay namespace), and exposes `A` to contracts via a new opcode (e.g. `MANIFESTAUTHORITY`) so downstream logic can gate on outcome authorization. A more flexible variant — per-entry signatures, with different parties authorizing different addresses' constraints — is also possible at the cost of larger grammar surface.

This is deferred because the application-layer patterns already cover the dominant use cases; the EL-level extension introduces a new replay-protection surface (per-authority nonce, per-tx signature verification cost, MEV implications around who pays for failed manifest authority recovery); and it benefits from deployment experience with the base manifest primitive before being added to consensus. A follow-up EIP MAY add it if application-layer patterns prove insufficient in practice.

### Manifest authorship

The manifest can be authored by either the wallet or the dapp; the protocol does not distinguish, and both paths are expected in practice:

- **Wallet-generated.** The wallet runs a full simulation of the proposed transaction, enumerates every emission across every emitter, and writes one event item per observed log. The wallet is the user's agent, so this path has the strongest trust properties, but it requires the wallet to run a tracing simulator against current state. Wallet-generated manifests tend to be loose: a generic simulator does not know which events express protocol intent versus which are incidental.

- **Dapp-generated.** The dapp supplies a manifest alongside the calldata, drawing on its protocol-level knowledge of expected outcomes (e.g. exactly one `Swap` from the router, exactly two `Transfer`s from the token contract, one `LP` mint, no `Approval`s). The wallet's role becomes verification: re-simulate, confirm the supplied manifest is consistent with the simulation, render it via ERC-7730, and sign. Dapp-authored manifests can be tighter and more semantically meaningful — they encode the protocol's intent, not just the simulator's trace.

In both cases the wallet's signature binds the manifest: the EVM enforces what was signed regardless of who drafted it. A dapp cannot smuggle in a manifest the wallet did not check, because the wallet signs over the full transaction including the access list. Conversely, a wallet that wants to refuse a dapp-supplied manifest (because it is too loose, too tight, or inconsistent with simulation) simply does not sign. Tooling and ERC-7730 metadata are the bridge that lets the wallet present a dapp-authored manifest to the user in human-readable form, the same way it presents a wallet-authored one.

### Why the LOG opcode is encoded by `topic_predicates.length`

An earlier draft allowed items with fewer topic predicates than the log being matched ("predicates beyond the log's topic count cause non-match" but no rule on the reverse). That permitted a LOG1 item to absorb a LOG3 emission as long as `topics[0]` matched — the extra two indexed parameters were unchecked, opening a path for an attacker to smuggle indexed payload past a partially-specified item. Tightening the match to require `topic_predicates.length == log.topics.length` closes this without adding any new field: the item's predicate count is itself the LOG-opcode declaration (`LOG0` through `LOG4`).

An explicit opcode metadata byte was considered and rejected. It would have been redundant with the predicate count, introduced a second encoding of the same information (with potential mismatch), and added one byte per item. The implicit encoding makes it impossible to declare a topic-predicate count that disagrees with the targeted opcode, and anonymous events fall out for free: anonymous `LOG0` → 0 topic predicates; anonymous `LOG1` → 1 predicate at `topics[0]` (wildcard or pinned), etc.

The corner case is "this signature regardless of indexed-parameter count" — which never arises in Solidity (the indexed-parameter count is fixed by the event's ABI) and can in any case be expressed by declaring one item per LOG level. The grammar surface to support it natively is not justified.

### Why a bijection, enforced greedily

A weaker rule — "each item matched by some log, each log matched by some item" — is insufficient against duplicate emissions. A contract could emit the same event twice when the user signed off on one, and both copies would trivially satisfy the same item; the user is robbed twice while the manifest reads valid. Requiring a one-to-one pairing forces the wallet to enumerate every distinct emission and forces the verifier to reject any log it cannot consume against a unique item. To permit `N` copies of an event, the user must explicitly declare the item `N` times — duplication becomes an opt-in, signed decision rather than an exploitable ambiguity.

The bijection is built greedily — emissions iterate in emission order, each consumes the first remaining item it satisfies — rather than via end-of-execution bipartite matching. Greedy matching is strictly weaker (a wallet that interleaves a specific item after a wildcard can revert even when a bijection theoretically exists), but the tradeoff is justified:

- Implementation is a single pass per `LOG` opcode; no global maximum-matching algorithm in consensus code.
- Failures revert eagerly, conserving gas and isolating the offending emission.
- For non-overlapping items (the typical case) the greedy outcome coincides with the unique bipartite matching.
- The remaining edge case — overlapping predicates — admits a clear authoring rule ("declare specific before broad"), well within wallet capability.

Coverage is global. Every contract that emits during the transaction — including helper contracts, routers, oracle callbacks, and internal-call emitters — must appear in the access list with event items, and the wallet builds the manifest from a complete simulation trace. The alternative (allowing undeclared addresses to emit freely) leaves a trivial bypass: a malicious entry point reroutes the substantive emission through an undeclared helper, and the manifest signed by the user becomes decorative. The tradeoff is enumeration cost — larger manifests, higher intrinsic gas — in exchange for the user signing off on the complete set of observable effects.

A wildcard event item (`[[sig], []]` — signature pinned via the bare-bytes32 shorthand, all other topics and data unconstrained) admits any single event with that signature from the address; declare it once per expected occurrence, and declare it AFTER any more specific items for the same signature.

### Storage slots kept advisory

Storage post-value predicates were considered and removed. They require sub-slot bit-level addressing (packed Solidity slots), a separate predicate language for storage post-values, and protocol-level understanding of storage layouts that varies between contracts. Event predicates achieve nearly the same user-facing goals without the encoding complexity or VM coupling.

Storage slots remain in the grammar in their EIP-2930 form because they are still useful for warming and for EIP-7928 parallelization. They live at the `storage_slots` position of each entry, peer to `event_constraint`, so an address can declare any combination of (slot warming, emission constraint) independently:

| Entry | Slot warming | Emission constraint |
|---|---|---|
| `[ADDR, [], []]`           | none      | any-emit |
| `[ADDR, [K], []]`          | `K` warm  | any-emit |
| `[ADDR, [K], 0x]`          | `K` warm  | forbid (no emissions allowed) |
| `[ADDR, [K], [item, ...]]` | `K` warm  | bijection over the listed items |

Storage slots carry no post-execution check, no emission rights, and no participation in the bijection. Encoding and intrinsic gas charging match EIP-2930 / EIP-7981 exactly, including the constant name `ACCESS_LIST_STORAGE_KEY_COST`.

A type-`TX_TYPE` transaction whose `event_constraint` is `[]` everywhere is therefore observationally identical to a type-`0x02` transaction with the same storage slots, modulo the new type byte and the intrinsic gas line item — confirming the no-regression backwards-compatibility claim in §"Backwards Compatibility".

### Why `0x` for the forbid sentinel

The third position of the entry is a position where the grammar already distinguishes "list" from "string" naturally. Picking the empty string `0x` as the forbid sentinel exploits this existing axis without introducing any new sentinel value or magic byte:

- **Type-disjoint from `[]` at the RLP layer.** `[]` encodes as `0xc0`; `0x` encodes as `0x80`. They differ by exactly RLP's type-tag bit. A decoder distinguishes them before any payload is read, so the choice is unambiguous and unforgeable at the wire level.
- **Canonical and minimal.** Both `0x` and `[]` have exactly one valid RLP encoding (a single byte each). There is no shorter or alternative encoding either can be silently mutated into, which closes a class of round-trip / re-encoding bugs that magic-byte sentinels (e.g., `0xff`, `0x00`, ASCII `F`) would invite.
- **Semantically symmetric.** `[]` reads as "empty list of constraints to enumerate" (declarative absence of a bijection); `0x` reads as "empty payload, prohibition only" (declarative absence of any permitted emission). The pair is mnemonic at a glance: list = enumerate; string = prohibit.

Alternatives considered and rejected:

| Alternative | Why rejected |
|---|---|
| Drop the case; use 2-tuple `[ADDR, [K]]` for forbid. | Same wire shape carries opposite meaning across tx types: warm-only in 0x02, forbid in `TX_TYPE`. Copy-paste between tx types silently flips semantics. |
| Reserved magic byte (e.g. `0xff`, `0x00`, `0x46` = ASCII `F`). | Requires a normative "exactly this byte, no other" rule. Tooling can silently mutate to a different byte that round-trips through their internal representations. Adds 1 byte of payload (RLP `0x81 ff`) over the empty-string form. |
| Sentinel item inside the list (e.g. `[ADDR, [K], [forbid_item]]`). | Conflates the forbid signal with the bijection items, requiring a mixing rule (forbid sentinel may only appear alone). The forbid signal also gets visually buried at a deeper nesting level. |
| Fourth tuple element (e.g. `[ADDR, [K], [], flags]`). | Breaks the always-3-tuple rule. Adds payload (1+ bytes for the flags item plus its RLP prefix). Conflates a binary choice with a future extension surface. |

### Why an explicit any-emit form

Without an any-emit form the grammar conflates two distinct intents: "this address is unconstrained" and "I forgot to declare this address." Both look identical (the address is simply absent). With `event_constraint = []` the relaxation is signed, rendered, and surfaced — the wallet shows the user "this address may emit anything" instead of silently treating an omission as deliberate. This matters because real flows often touch contracts whose emissions a wallet cannot tightly audit (third-party routers, oracle callbacks, proxy admins); forcing the user to enumerate every emission from such a contract — or to fail to sign — is a common failure mode. The any-emit form gives the user a legible escape that is still part of the signed manifest, so a phishing dapp cannot silently widen a bijection into an any-emit without changing the bytes the user signs over.

### Composition with Other Standards

**EIP-7702.** The manifest is enforced regardless of whether the transaction uses delegated authority. Combined with 7702, a delegated executor's actions are bounded by the manifest the user signed.

A 7702-delegated EOA executes code at the EOA's own address: when that code emits a log, the `LOG` opcode's emitting address is the EOA itself, not the delegated implementation contract. The signer's own EOA address therefore MUST appear in the access list with an appropriate `event_constraint` (bijection, any-emit, or forbid) whenever the delegated code is expected to emit. Omitting the EOA from the manifest while its delegated code emits triggers the undeclared-emitter revert under §"Matching". Tooling that drafts manifests from a simulation trace MUST attribute logs emitted under a 7702 delegation to the EOA, not to the underlying implementation contract.

**ERC-4337.** A 4337 validator MAY check the manifest as part of `validateUserOp`. The on-chain transaction submitted by the bundler carries the manifest in its access list, and the EL enforces it independently of the bundler.

**ERC-7683.** A cross-chain order can carry the manifest in its `orderData` as a sub-type. The destination-chain fill transaction includes the manifest in its access list, giving the user atomic enforcement of the destination outcome.

**ERC-7730.** Clear-signing metadata describes how to render event predicates as human-readable manifests. The user sees "Swap 100 USDC for at least 0.04 WETH"; the wallet encodes this as event predicates over `Transfer` and `Swap` signatures.

Global whitelist semantics raise the bar on 7730 coverage. Because every emission in the transaction must be enumerated by a manifest item — across every emitter, not just the contract carrying the user's intent — the wallet must render, and the user must be able to recognize, the full set of permitted events from the full set of declared contracts. Incomplete metadata risks either (a) spurious reverts, if the wallet omits an incidental emission from a helper or routed-through contract, or (b) opaque approvals, if the user is shown a headline outcome while the manifest tacitly permits unrendered emissions from contracts they do not understand are participating. 7730 publishers SHOULD provide labels for every event a contract may emit during the intended interaction; wallets SHOULD warn when a simulated emitter or event lacks 7730 metadata and surface those entries explicitly rather than burying them in wildcards.

## Test Cases

The following test vectors illustrate the matching and enforcement rules. They are not exhaustive; conforming implementations MUST also reproduce the gas accounting in §"Gas Costs".

Throughout, let:

- `TOKEN_A`, `TOKEN_B`, `POOL`, `HELPER` denote distinct 20-byte addresses.
- `T = keccak256("Transfer(address,address,uint256)")`.
- `S = keccak256("Swap(address,uint256,uint256,address)")`.
- `A = keccak256("Approval(address,address,uint256)")`.
- `addr32(X)` denote the address `X` left-padded to 32 bytes.
- `u(N)` denote `N` encoded as a left-padded 32-byte unsigned integer.

### Case 1 — Exact bijection (accept)

Access list (each entry is `[address, storage_slots, event_constraint]`):

```
[
  [TOKEN_A, [], [ [[T, addr32(alice), addr32(POOL)], [u(100)]] ]],
  [POOL,    [], [ [[S, addr32(alice)],               [u(100), [[u(40), u(50)]], addr32(alice)]] ]],
  [TOKEN_B, [], [ [[T, addr32(POOL),  addr32(alice)], [[[u(40), u(50)]]]] ]]
]
```

Emitted, in order: `TOKEN_A` emits `Transfer(alice, POOL, 100)`; `POOL` emits `Swap(alice, 100, 45, alice)`; `TOKEN_B` emits `Transfer(POOL, alice, 45)`. Each queue is consumed exactly once. **Outcome:** accept.

### Case 2 — Duplicate emission against a single item (revert)

Same access list as Case 1, but `TOKEN_A` emits `Transfer(alice, POOL, 100)` twice. The first emission consumes the lone `TOKEN_A` item; the second has no item left in the queue. **Outcome:** revert at the second `LOG3`.

### Case 3 — Duplicate emission authorized by duplicate item (accept)

Access list contains `TOKEN_A`'s item twice (identical predicates). `TOKEN_A` emits the same `Transfer(alice, POOL, 100)` twice. The two emissions consume the two duplicate items in declaration order. **Outcome:** accept.

### Case 4 — Greedy match: specific-before-broad ordering matters

Access list contains a single entry `[TOKEN_A, [], [item_specific, item_wildcard]]`, with items in order:

```
[
  [[T, addr32(alice), addr32(POOL)], [u(100)]],   # specific
  [[T],                              []]           # wildcard at signature T
]
```

`TOKEN_A` emits `Transfer(alice, POOL, 100)` then `Transfer(POOL, bob, 5)`. The first emission satisfies both items; greedy match consumes the specific one. The second emission satisfies only the wildcard. **Outcome:** accept.

Reverse the declaration order (wildcard first, specific second), keep emissions identical. The first emission satisfies the wildcard first; the wildcard is consumed by `Transfer(alice, POOL, 100)`. The second emission `Transfer(POOL, bob, 5)` does not satisfy the specific item (recipient mismatch). **Outcome:** revert.

### Case 5 — Undeclared emitter (revert)

Access list as in Case 1. During execution, `HELPER` (absent from the access list) emits any log. **Outcome:** revert at the `LOG` opcode that emitted from `HELPER`.

### Case 6 — Forbid sentinel emits (revert)

Access list contains `[HELPER, [0x00..01], 0x]` (slot warmed, `event_constraint = 0x`). `HELPER` emits any log. **Outcome:** revert at the `LOG` opcode; the forbid sentinel is a positive prohibition. Storage warming grants no emission rights.

### Case 6b — Any-emit form (accept)

Access list contains `[HELPER, [], []]` (no slots warmed, `event_constraint = []`). `HELPER` emits arbitrary logs during the transaction. **Outcome:** accept; the any-emit form authorizes the emissions and they are not bijection-matched. Wallets MUST render this entry prominently as "may emit any events" (see §"Backwards Compatibility" / Wallet rendering).

### Case 7 — End-of-tx queue non-empty (revert)

Access list as in Case 1, but `TOKEN_B` never emits. At end of execution, `TOKEN_B`'s queue still holds one unmatched item. **Outcome:** revert before state commit.

### Case 8 — Shorthand equivalence

For `topic_predicates[0]` (the signature topic), each of the following encodings matches an emitted `log.topics[0] == T`:

```
T                       # bare bytes32
[T]                     # single-member union
[[T, T]]                # single-member range
```

All three MUST yield identical accept/reject outcomes on every log. RLP-encoded byte counts: `T` = 33 bytes (one 32-byte string with its `0xa0` prefix); `[T]` = 34 bytes (33-byte payload + 1-byte short-list prefix); `[[T, T]]` = 70 bytes (66-byte inner-list payload + 2-byte long-list prefix wrapped again with a 2-byte long-list prefix).

### Case 9 — `LOG`-opcode discrimination by `topic_predicates.length`

Access list contains `[TOKEN_A, [], [ [[T, addr32(alice)], []] ]]` (one item, `topic_predicates.length == 2`, matches `LOG2` only):

```
[ [[T, addr32(alice)], []] ]
```

If `TOKEN_A` emits an event with `log.topics.length == 3` (a `LOG3`, e.g. the standard `Transfer(from,to,value)` whose `from` and `to` are both indexed), the item does not match (rule 1), no other item is queued, and the transaction reverts. The wallet must declare the item with 3 topic predicates to target a `LOG3`.

### Case 10 — Data-length equality

Access list contains `[POOL, [], [ [[S], [u(100), u(45)]] ]]` (`data_predicates.length == 2`):

```
[ [[S], [u(100), u(45)]] ]
```

If `POOL` emits a `Swap` whose data is 3 words (e.g. an additional non-indexed `address recipient`), `log.data.length / 32 == 3 ≠ 2`. Rule 3 fails; the transaction reverts.

### Case 11 — Union vs. range distinction

Two entries for `POOL` (`[POOL, [], <event_constraint>]`) differ only in one outer pair of brackets around a 2-list at the data-predicate position:

```
# (a) union of two exacts:  admits value == 40 or value == 50
[ [[S, addr32(alice)], [u(100), [u(40), u(50)], addr32(alice)]] ]

# (b) single range:          admits 40 <= value <= 50
[ [[S, addr32(alice)], [u(100), [[u(40), u(50)]], addr32(alice)]] ]
```

`POOL` emits `Swap(alice, 100, 45, alice)` (data middle word == 45):

- Against (a): 45 is neither 40 nor 50 → predicate not satisfied → no item matches → revert.
- Against (b): `40 <= 45 <= 50` → predicate satisfied → bijection holds → accept.

Conversely, if `POOL` emits `Swap(alice, 100, 50, alice)` (middle word == 50), both (a) and (b) accept (50 is an exact member of the union and the upper bound of the range). The two encodings are interchangeable only on the boundary points; everywhere in the open interior they diverge. Wallets MUST NOT silently elide the distinction.

## Security Considerations

### Manifest tightness

The manifest is the complete declaration of every emission in the transaction. Each declared address must satisfy a bijection between its event items and its emitted logs, built greedily as logs fire; each undeclared address must emit zero logs. The verifier reverts on (a) any unaccounted-for emission from a declared address, (b) duplicate emissions over the same predicate when only one item is declared, (c) cardinality mismatches in either direction (one extra `Approval` flips the manifest from valid to revert; one missing `Transfer` does the same), (d) any emission at all from a contract the user did not enumerate, and (e) a greedy match that "steals" a specific log into a broader item, leaving a specific item with no log to claim.

Manifest authors (wallet or dapp) SHOULD work from full simulation traces, declare one event item per observed emission across every emitter the trace touches (including helper, router, oracle, and proxy-internal contracts), widen predicate ranges — not cardinalities — to absorb expected state drift, and order items from most-specific to least-specific within each address group so the greedy match cannot mis-consume. Omitting any emitter, miscounting any emission, or mis-ordering overlapping items causes the transaction to revert. Wallets receiving a dapp-supplied manifest MUST re-simulate and verify the manifest matches the trace before signing; an unverified manifest from an untrusted dapp gives the user the same exposure as the pre-manifest world. Tooling SHOULD warn the user when the simulated trace contains many low-information emissions (e.g. logs from libraries) since each must be signed off on as a discrete entry.

### State drift

A manifest is generated at sign time — by the wallet from a simulation trace, or by the dapp from its protocol knowledge, with the wallet re-simulating to verify. Between signing and inclusion, on-chain state may change such that the same calldata no longer produces the declared events. The transaction will revert. This is the intended failure mode but represents a UX consideration: tight manifests may have high revert rates in fast-moving markets. Manifest authors SHOULD widen predicate ranges proportionally to expected drift.

### Verifier cost

Manifest checking is inline with execution: each `LOG` opcode does an O(items_remaining_in_queue) scan of unconsumed items in declaration order and consumes the first match (or reverts). Because the queue only shrinks over the transaction, the per-emission scan is bounded by the *initial* per-address queue length and amortizes downward as items are consumed: total per-transaction event-matching work is at most `Σ_addr (n_addr * (n_addr + 1) / 2)` comparisons, where `n_addr` is the number of event items declared for that address — i.e., quadratic in per-address declarations but with no cross-address coupling and no global maximum-matching pass. The end-of-execution check is a single O(declared_addresses) sweep that confirms every queue is empty. Gas pricing on event items (per-item intrinsic plus per-byte data, see §"Gas Costs") is calibrated so the intrinsic charge dominates the verifier's per-emission cost; storage items remain free of post-execution work.

### Replay and nonce

The manifest is bound to the transaction by the transaction's signature. Replay protection is provided by the transaction nonce in the standard way. No separate manifest nonce is needed.

### Mempool intent exposure

A signed manifest is visible to anyone who sees the transaction in the public mempool. Compared to a calldata-only transaction, the manifest makes the user's intended outcomes legible at the protocol level — searchers no longer need to simulate to predict expected emissions. This raises no new class of MEV (the calldata already encodes the operation and the simulator already extracts outcomes), but it lowers the cost of doing so. Users sensitive to this exposure SHOULD route through private mempools, intent/solver flows, or ERC-4337 bundlers that hold the transaction until inclusion, as they would for any latency-sensitive transaction today.

### Contracts deployed mid-transaction

A contract created during the transaction (via `CREATE` or `CREATE2`) may emit a log from its constructor. The deployed address is deterministic given the transaction and pre-transaction state; the wallet's simulation trace surfaces it directly:

- **CREATE2**: `keccak256(0xff || deployer || salt || keccak256(init_code))[12:]`. Depends only on the deployer, salt, and init code; drift-immune across mempool reorderings.
- **CREATE**: `keccak256(rlp([deployer, nonce]))[12:]`, where `nonce` is the deployer's nonce at the instant the `CREATE` opcode fires. Within the transaction this is fully determined by the simulation; there is no intra-transaction race.

Wallets MUST enumerate every constructor-emitting deployment surfaced by the trace as an ordinary access-list entry with the predicted address and event constraint.

**CREATE-specific drift.** The deployer's nonce is shared mutable state. Any other transaction landing before this one that causes the same deployer to execute `CREATE` shifts the predicted address. The shifted deployment emits from an address the manifest does not enumerate, triggering the undeclared-emitter revert. This is the same class of state-drift failure the manifest produces for any other shifted observation (e.g., a swap output landing outside a predicate range); it is the intended failure mode, not a special case. Dapps whose flows depend on a deployer contract that frequently CREATEs SHOULD prefer `CREATE2` for emitter deployments, or wrap the constructor's emission with a re-emit from a stable parent address.

### Composability with reverts

The verifier journals violations per call frame (see §"Frame journaling"): an emission that would violate the manifest inside a frame that subsequently reverts is discarded along with the frame's other state, exactly as the EVM discards the log itself. This is the semantics that makes lazy and eager enforcement equivalent, and it is also what permits "trial" patterns (e.g., a router that probes a path inside `try`/`catch` and reverts if it observes an unwanted emission) to keep working — only emissions that actually survive into the outermost frame's commit count.

The flip side: a violation that *does* survive into the outermost commit is a transaction-level exceptional halt analogous to out-of-gas, not a frame-level `REVERT` opcode. A parent contract's `try`/`catch` or `STATICCALL`-with-success-check CANNOT swallow such a violation — once the marker is folded into the outermost frame's journal, no subsequent frame revert can discard it; the entire transaction's state changes are discarded and the sender pays gas up to the point of detection. A router that wraps the user's intended call in a `try`/`catch` and continues on violation does not defeat the anti-phishing property: any unauthorized emission whose effects the router preserves (i.e., the router itself commits) propagates to the outermost frame and triggers the halt. Implementations MUST surface the failure as a top-level transaction revert, not as a returndata-bearing frame revert.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
