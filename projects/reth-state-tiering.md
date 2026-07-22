# State Tiering using EIP-8188 and EIP-8295 for Reth 

Implementing Ethereum state tiering through `last_written_block` and `last_written_period` signal to allow for increased gas costs for write-inactive states.

## Motivation

Ethereum's state is ever evolving. Being one of the most used cryptocurrencies worldwide, there is a constant influx of account data, storage slot data and multiple other things. According to the article "[Not All State is Equal](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)" which contains a detailed analysis of the Ethereum state from **genesis** block to block **22,431,083**, the Ethereum state is very uneven: most contracts go inactive right after creation, a majority of storage slots are never touched again after their first write, and a small number of contracts and factories account for most of the state. Out of the full node state around **80%** has been unaccessed for over a year ([EIP-8188 ACD Slides](https://weiihann.github.io/20260521-eip8188-acd/))

This proposal intends to form the baseline for splitting the Ethereum state into a "hot" and "cold" one based on a write-age signal. In other words, "cold" state would be states which have not had a write operation in a long time and "hot" state would be those which have had a recent write operation. This allows individual clients to optimize the commit-critical mutable path: trie updates, state-root computation, compaction, and other work associated with repeatedly rewriting state.

## Project description

This project contains a complete implmentation of the signals described in [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) and [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) for Reth following the [Geth Implementation](https://github.com/weiihann/go-ethereum/tree/feat/eip-8188/inject) by the original author. The following diagram contains a simplistic description of the hot-cold state separation.

```
hot state                           cold state
───────────────────────────         ─────────────────────────────────────────
standard MPT nodes (RLP)
+ stubs pointing to           ──→   blobs where each blob = one originally-stubbed subtree
```

The "hot" state is stored in a database where write access is easier and cheaper (RocksDB) and "cold" state is basically a simple append-only flat file. Commands that allow us to backfill the necessary data into the state and tier the states accordingly will be added to the `reth db` suite:

1. `reth db inject-periods` : This command is going to allow us to enter the necessary `period` data, as described in [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) into the state of the chain. 

2. `reth db convert-inactive` : This command is going to identify the inactive states, convert them into a format writable in the append-only flat file and connect the necessary stubs in the hot state to the blobs in the cold state. This hot-cold separation will ensure the tiering happens appropriately.

Some other support commands may be added as necessary.

## Specification

### Data Collection

To collect the necessary data, we will be using data collected through [Xatu](https://github.com/ethpandaops/xatu) and stored in ClickHouse. The ClickHouse instance allows us to extract necessary data using simple SQL queries:

**Account Data**
    
**SQL Query** 

```
SELECT lower(address) AS addr, max(block_number) AS block
    FROM (
        SELECT address, block_number FROM canonical_execution_balance_diffs
            WHERE block_number >= ? AND block_number <= ?
        UNION ALL
        SELECT address, block_number FROM canonical_execution_nonce_diffs
            WHERE block_number >= ? AND block_number <= ?
        UNION ALL
        SELECT address, block_number FROM canonical_execution_storage_diffs
            WHERE block_number >= ? AND block_number <= ?
        UNION ALL
        SELECT contract_address AS address, block_number FROM canonical_execution_contracts
            WHERE block_number >= ? AND block_number <= ?
    )
    GROUP BY addr
```

**Explanation**

The query aggregates all account write events within the specified block range by combining balance changes (`canonical_execution_balance_diffs`), nonce updates (`canonical_execution_nonce_diffs`), storage modifications (`canonical_execution_storage_diffs`), and contract creations (`canonical_execution_contracts`). It then groups the results by account address and selects the maximum block_number for each account. This value corresponds to the account's `last_written_block`, following the definition of "account modified" as defined under the "Account Rules" section in [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188), since it represents the most recent block in which the account was modified. 

**Storage Slot Data**

**SQL Query** 
```
SELECT lower(address) AS addr, lower(slot) AS slot_key, max(block_number) AS block
    FROM canonical_execution_storage_diffs
    WHERE block_number >= ? AND block_number <= ?
        AND to_value != '0x0' AND to_value != '0x00' AND to_value != '0'
    GROUP BY addr, slot_key
```

**Explanation**

The query scans `canonical_execution_storage_diffs` to identify all storage slot modifications within the specified block range. It filters out writes where the new value is zero (`0x0`, `0x00`, or `0`), since EIP-8188 tracks only storage slots that remain allocated after the write. The results are grouped by account address and storage slot, and the maximum `block_number` is selected for each pair. This value represents the storage slot's `last_written_block`, i.e., the most recent block in which that non-zero storage slot was modified.

From this `last_written_block` data, `last_written_period` is calculated as follows:

```
last_written_period = max(0, (last_written_block - PERIOD_START_BLOCK) // PERIOD_LENGTH)
```

This definition comes from [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295). The `inject-periods` command describes earlier uses these SQL queries to inject the period data into the state.

### Inactive Subtree Identification

The inactive-subtree identification routine identifies **maximal inactive subtrees** in the Ethereum state trie according to the inactivity criteria defined by EIP-8188. A subtree is considered **inactive** if every account (or storage slot, in the case of a storage trie) contained within the subtree has an age greater than or equal to a user-specified threshold (`inactive_min_age`). The age of an object is computed from the `last_written_period` stored in the EIP-8188 snapshot metadata.

The identification algorithm performs a single sequential DFS traversal of the account trie while simultaneously advancing a snapshot iterator over the corresponding metadata. Since both iterators enumerate leaves in the same deterministic order, every account or storage slot is processed exactly once without requiring random lookups. This makes the algorithm suitable for execution on mainnet-sized state and for integration into clients such as Reth.

During traversal, a stack of internal-node frames is maintained. Each frame aggregates the inactivity status of its children in a bottom-up manner. Child subtrees that are completely inactive are temporarily stored as candidates. When an internal node has been fully processed, the algorithm determines whether all of its descendants are inactive. If so, the node itself becomes the representative inactive subtree and its candidate children are discarded. Otherwise, the node is mixed, and all previously collected inactive child candidates are emitted as maximal inactive subtree roots. This guarantees that the output contains the largest possible inactive subtrees without redundancy.

Before beginning traversal, the snapshot is synchronized with the latest committed state by flushing any pending in-memory or layered state into the persistent snapshot view. This ensures that recently modified accounts and storage slots are visible to both the snapshot iterator and the trie traversal, preventing stale metadata from affecting subtree identification.

#### Pseudocode

```rust
fn identify_inactive_subtrees(trie: Trie, snapshot: Snapshot, min_age: u64) {
    let mut iterator = trie.node_iterator();
    let mut snapshot_iter = snapshot.leaf_iterator();
    let mut stack = Vec::new();

    while let Some(node) = iterator.next() {
        match node {
            Node::Internal => {
                stack.push(Frame::new(node.hash()));
            }

            Node::Leaf(leaf) => {
                let metadata = snapshot_iter.next().unwrap();

                assert_eq!(leaf.key(), metadata.key());

                let inactive =
                    current_period - metadata.last_written_period >= min_age;

                if let Some(parent) = stack.last_mut() {
                    parent.record_leaf(inactive);
                }
            }

            Node::EndInternal => {
                let frame = stack.pop().unwrap();

                if frame.all_children_inactive() {
                    if !frame.is_embedded() {
                        if let Some(parent) = stack.last_mut() {
                            parent.add_candidate(frame.root_hash());
                        } else {
                            emit(frame.root_hash());
                        }
                    }
                } else {
                    for subtree in frame.candidates() {
                        emit(subtree);
                    }

                    if let Some(parent) = stack.last_mut() {
                        parent.record_child_active();
                    }
                }
            }
        }
    }

    for contract in trie.contract_accounts() {
        if contract.storage_root() != EMPTY_ROOT {
            identify_inactive_subtrees(
                contract.storage_trie(),
                snapshot.storage_snapshot(contract.address()),
                min_age,
            );
        }
    }
}
```

After the inactive subtree has been identified, it gets stored in the cold state and a reference for it is stored in the hot state.

### Cold State Format
When a maximal inactive subtree is identified and moved to the cold file, it is replaced in the hot database by a compact 17-byte "primary stub". This stub acts as a filesystem pointer, allowing the client to resolve the data only when needed.

The format of the Primary Stub is as follows:

* Marker (1 Byte): 0x00 — identifies the database entry as a cold state stub.

* Blob Offset (8 Bytes): The absolute byte position in the append-only file where the specific subtree blob begins.

* Root in Blob (4 Bytes): The relative offset within that blob entry to the root node of the subtree.

* Root Size (4 Bytes): The byte length of the root entry.

The actual Cold State file is a collection of concatenated **blobs**, where each blob has the following format:

* **Header (16 bytes)**

  * Stores metadata describing the blob.
  * Contains the blob format version, reserved bytes, the offset of the root node, and the size of the root node.
  * Enables the reader to locate the root of the serialized trie.

* **Serialized Trie**

  * Contains the complete inactive subtree serialized in **post-order depth-first traversal (DFS)**, where all children are written before their parent.
  * The root node is therefore the last node stored in the blob.

* **Trie Node**

  * Every node begins with a one-byte **node type** identifier.
  * A node can be one of two types:

    * **Full Node**

      * Represents a branch node in the Merkle Patricia Trie.
      * Contains 17 child slots: one for each hexadecimal nibble (`0–15`) and one value slot.
    * **Short Node**

      * Represents either an extension node or a leaf node.
      * Contains the compressed key segment followed by a single child slot.

* **Child Slot**

  * Each child slot begins with a one-byte **kind** field indicating how the child is represented.
  * The supported child slot types are:

    * **Empty** – indicates that no child exists.
    * **Hashed Reference** – stores the child's Keccak-256 hash together with its offset and size within the blob.
    * **Embedded Reference** – stores only the child's offset and size, since embedded nodes do not have an independent hash.
    * **Inline Value** – stores the serialized value directly within the node (typically the account or storage value associated with a leaf).

This layout preserves the complete Merkle Patricia Trie structure while allowing inactive subtrees to be stored as self-contained blobs that can later be reconstructed or materialized without requiring access to the active state database.

## Roadmap

| **Phase**                                                 | **Weeks** | **Deliverables**                                                                                                                                                                                                                      |
| --------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1: Data Collection & Period Generation**                | **6**   | Implement the data collection pipeline using Xatu and ClickHouse, derive `last_written_block` for accounts and storage slots, compute `last_written_period` according to EIP-8295, and validate the generated metadata.               |
| **2: Period Injection (`reth db inject-periods`)**        | **7-8**   | Extend the state schema to store `last_written_period`, implement the `reth db inject-periods` command, inject account and storage metadata into the state, and ensure compatibility with the existing RocksDB-backed state.          |
| **3: Inactive Subtree Identification**                    | **9-11**  | Implement the DFS-based inactive subtree identification algorithm, traverse account and storage tries using snapshot metadata, identify maximal inactive subtrees, and validate correctness on representative state snapshots.        |
| **4: Cold State Conversion (`reth db convert-inactive`)** | **12-14** | Implement serialization of inactive subtrees into the append-only cold state file, generate the blob format and primary stubs, replace inactive subtrees in the hot state with stubs, and implement lazy retrieval from cold storage. |
| **5: Testing, Benchmarking & Optimization**               | **15-18** | Perform functional and integration testing, benchmark hot/cold state performance, verify state root correctness and data integrity, fix implementation issues, and optimize storage layout and traversal performance.                 |
| **(Stretch)**                                             | **18+**   | Extend the implementation for Ethrex as well if time remains.                                                          |


## Possible challenges

* **Uncertainty about Parameters**: [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) and [EIP-8295](https://eips.ethereum.org/EIPS/eip-8295) are very new EIPs and plenty of constant parameters like `blocks_per_period` and `inactive_min_age` are still TBD. Without those parameters, a lot of sample experimentation is necessary to figure out which are the optimal values for those parameters.

## Goal of the project

Success looks like:

## Goal of the Project

Success looks like:

* A complete implementation of the state tiering primitives described in **EIP-8188** and **EIP-8295** within Reth.
* A working `reth db inject-periods` command that computes and injects `last_written_period` metadata for all accounts and storage slots using data collected from Xatu.
* A working `reth db convert-inactive` command that correctly identifies maximal inactive subtrees, serializes them into the cold state file, and replaces them in the hot state with primary stubs.
* A cold state format that faithfully preserves the original Merkle Patricia Trie structure and allows inactive subtrees to be materialized on demand.
* Correctness of the implementation verified through functional and integration testing, ensuring that trie integrity, state roots, and account/storage lookups remain unchanged after state tiering.
* Performance evaluation demonstrating that inactive state can be successfully separated from the hot database while maintaining correctness and providing the foundation for future state expiry and tiered storage optimizations.

## Collaborators

### Fellows 

[Astrion](https://github.com/astrion-coder)

### Mentors

[Ng Wei Han](https://github.com/weiihann)

## Resources

* [Not All State is Equal](https://ethereum-magicians.org/t/not-all-state-is-equal/25508)
* [EIP-8188: Last-Written Block for Accounts and Slots](https://eips.ethereum.org/EIPS/eip-8188)
* [Geth implementation of State Tiering](https://github.com/weiihann/go-ethereum/tree/feat/eip-8188/inject)
* [Xatu](https://github.com/ethpandaops/xatu)
* [EIP-8188 ACD Slides](https://weiihann.github.io/20260521-eip8188-acd/)