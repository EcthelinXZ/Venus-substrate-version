# Availability Recovery

This subsystem is the inverse of the [Availability Distribution](availability-distribution.md) subsystem: validators will serve the availability chunks kept in the availability store to nodes who connect to them. And the subsystem will also implement the other side: the logic for nodes to connect to validators, request availability pieces, and reconstruct the `AvailableData`.

This version of the availability recovery subsystem is based off of direct connections to validators. In order to recover any given `AvailableData`, we must recover at least `f + 1` pieces from validators of the session. Thus, we will connect to and query randomly chosen validators until we have received `f + 1` pieces.

## Protocol

`PeerSet`: `Validation`

Input:

- NetworkBridgeUpdateV1(update)
- AvailabilityRecoveryMessage::RecoverAvailableData(candidate, session, backing_group, response)

Output:

- NetworkBridge::SendValidationMessage
- NetworkBridge::ReportPeer
- AvailabilityStore::QueryChunk

## Functionality

We hold a state which tracks the current recovery interactions we have live, as well as which request IDs correspond to which interactions. An interaction is a structure encapsulating all interaction with the network necessary to recover the available data.

```rust
struct State {
    /// Each interaction is implemented as its own async task, and these handles are for communicating with them.
    interactions: Map<CandidateHash, InteractionHandle>,
    /// A recent block hash for which state should be available.
    live_block_hash: Hash,

    /// interaction communication. This is cloned and given to interactions that are spun up.
    from_interaction_tx: Sender<FromInteraction>,
    /// receiver for messages from interactions.
    from_interaction_rx: Receiver<FromInteraction>,

    // An LRU cache of recently recovered data.
    availability_lru: LruCache<CandidateHash, Result<AvailableData, RecoveryError>>,
}

struct InteractionHandle {
    awaiting: Vec<ResponseChannel<Result<AvailableData, RecoveryError>>>,
}

struct Unavailable;
enum FromInteraction {
    // An interaction concluded.
    Concluded(CandidateHash, Result<AvailableData, RecoveryError>),
    // Send a request on the network.
    NetworkRequest(Requests),
}

struct InteractionParams {
    validator_authority_keys: Vec<AuthorityId>,
    validators: Vec<ValidatorId>,
    // The number of pieces needed.
    threshold: usize, 
    candidate_hash: Hash,
    erasure_root: Hash,
}

enum InteractionPhase {
    RequestFromBackers {
        // a random shuffling of the validators from the backing group which indicates the order
        // in which we connect to them and request the chunk.
        shuffled_backers: Vec<ValidatorIndex>,
    }
    RequestChunks {
        // a random shuffling of the validators which indicates the order in which we connect to the validators and
        // request the chunk from them.
        shuffling: Vec<ValidatorIndex>, 
        received_chunks: Map<ValidatorIndex, ErasureChunk>,
        requesting_chunks: FuturesUnordered<Receiver<DataResponse<ErasureChunk>>>,
    }
}

struct Interaction {
    to_state: Sender<FromInteraction>,
    params: InteractionParams,
    phase: InteractionPhase,
}
```

### Signal Handling

On `ActiveLeavesUpdate`, if `activated` is non-empty, set `state.live_block_hash` to the first block in `Activated`.

Ignore `BlockFinalized` signals.

On `Conclude`, shut down the subsystem.

#### `AvailabilityRecoveryMessage::RecoverAvailableData(receipt, session, Option<backing_group_index>, response)`

1. Check the `availability_lru` for the candidate and return the data if so.
1. Check if there is already an interaction handle for the request. If so, add the response handle to it.
1. Otherwise, load the session info for the given session under the state of `live_block_hash`, and initiate an interaction with *launch_interaction*. Add an interaction handle to the state and add the response channel to it.
1. If the session info is not available, return `RecoveryError::Unavailable` on the response channel.

### From-interaction logic

#### `FromInteraction::Concluded`

1. Load the entry from the `interactions` map. It should always exist, if not for logic errors. Send the result to each member of `awaiting`.
1. Add the entry to the availability_lru.

#### `FromInteraction::NetworkRequest(requests)`

1. Forward with `NetworkBridgeMessage::SendRequests`.

### Interaction logic

#### `launch_interaction(session_index, session_info, candidate_receipt, candidate_hash, Option<backing_group_index>)`

1. Compute the threshold from the session info. It should be `f + 1`, where `n = 3f + k`, where `k in {1, 2, 3}`, and `n` is the number of validators.
1. Set the various fields of `InteractionParams` based on the validator lists in `session_info` and information about the candidate.
1. If the `backing_group_index` is `Some`, start in the `RequestFromBackers` phase with a shuffling of the backing group validator indices and a `None` requesting value.
1. Otherwise, start in the `RequestChunks` phase with `received_chunks` and `requesting_chunks` both empty.
1. Set the `to_state` sender to be equal to a clone of `state.from_interaction_tx`.
1. Initialize `received_chunks` to an empty set, as well as `requesting_chunks`.

Launch the interaction as a background task running `interaction_loop(interaction)`.

#### `interaction_loop(interaction)`

```rust
// How many parallel requests to have going at once.
const N_PARALLEL: usize = 50;
```

Loop:
  * If the phase is `InteractionPhase::RequestFromBackers`
    * If the `requesting_pov` is `Some`, poll for updates on it. If it concludes, set `requesting_pov` to `None`. 
    * If the `requesting_pov` is `None`, take the next backer off the `shuffled_backers`.
        * If the backer is `Some`, issue a `FromInteraction::NetworkRequest` with a network request for the `AvailableData` and wait for the response.
        * If it concludes with a `None` result, return to beginning. 
        * If it concludes with available data, attempt a re-encoding. 
            * If it has the correct erasure-root, break and issue a `Concluded(Ok(available_data))`. 
            * If it has an incorrect erasure-root, issue a `FromInteraction::ReportPeer` message and return to beginning.
        * If the backer is `None`, set the phase to `InteractionPhase::RequestChunks` with a random shuffling of validators and empty `received_chunks` and `requesting_chunks`.

  * If the phase is `InteractionPhase::RequestChunks`:
    * Poll for new updates from `requesting_chunks`. Check merkle proofs of any received chunks, and any failures should lead to issuance of a `FromInteraction::ReportPeer` message.
    * If `received_chunks` has more than `threshold` entries, attempt to recover the data. If that fails, or a re-encoding produces an incorrect erasure-root, break and issue a `Concluded(RecoveryError::Invalid)`. If correct, break and issue `Concluded(Ok(available_data))`.
    * While there are fewer than `N_PARALLEL` entries in `requesting_chunks`,
        * Pop the next item from `shuffling`. If it's empty and `requesting_chunks` is empty, break and set the phase to `Concluded(None)`.
        * Issue a `FromInteraction::NetworkRequest` and wait for the response in `requesting_chunks`.
