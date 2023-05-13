## Node based StateMachine Manager (in development)

## Goals

* decoupling state machine input processing from a given **state’s current enumerations**
* state signaling that all feeds into the same sink (the manager’s `signal_queue`) ; this allows lifts and transits to be processed **homogeneously** thus avoiding type opacity through `Box<dyn Signal>`


## In practice the design should give at most three message streams connected to a particular state machine down:
I/O

* One `Input` (handles both external and internal events)
```rust
Signal {
    id: StateId<K>,
    input: I,
}
```
Two outputs:
* `Signal` output (events meant to be processed as inputs for other state machines)
* `Notification` output (events meant to be processed by _anything_ that is not a state machine fed by a given `signal_queue`)


### The new `StateMachineManager` owns:
* The state storage layer (using `NodeStorage`)
* the input event stream
* The state machine processors

### The `StateMachineManager` is responsible for:
* routing `Signal`s to the appropriate state machines
* Injecting `ProcessorContext`s into the state machines: this action is what allows state machines to cycle concurrently

https://github.com/knox-networks/core/blob/67f7dc6ac57f5c6650d82ce0019e65a31278ae93/common/src/state_machine/node_state_machine.rs#L65-L74

### `NodeStore` is responsible for:
* inserting & updating various state hirearchies
* operations are done concurrently by holding all node trees in `Arc<Mutex<_>>` containers.

### This allows `NodeStore` storage to:
* Create multiple indices (Through fresh `DashMap` key insertions) pointing to the same tree by incrementing the `Arc` count and inserting a new entry per child node
* Allows independent interior mutability per state tree, decoupling unrelated states from resource contention

https://github.com/knox-networks/core/blob/67f7dc6ac57f5c6650d82ce0019e65a31278ae93/common/src/state_machine/storage.rs#L10-L16


## Node based `TimeoutManager` (in development)

Considerations:
  - only one timer is active per `StateId<K>`, State machines should not have
    to keep track of `Operation::Set(Instant::now())`
    emitted to notifications.
    Thus, all timers should be indexable by `StateId<K>`.
  - A newer `Operation::Set` for the same `StateId<K>` should override an old timer.
  - A timeout should emit a `Signal` that is pertinent to the related state machine.

Approach:
* `TimeoutManager` implements a per tick polling approach to resolving
  timeouts:
  https://github.com/knox-networks/core/blob/83d57647e38a55c5cfecacca8c89ebe98d45ab68/common/src/state_machine/timeout.rs#L170-L193
* TimeoutManager accepts two types of inputs, set and cancel (timeout):
  https://github.com/knox-networks/core/blob/83d57647e38a55c5cfecacca8c89ebe98d45ab68/common/src/state_machine/timeout.rs#L85-L89
* Timeouts are stored within the `TimeoutLedger`:
  https://github.com/knox-networks/core/blob/83d57647e38a55c5cfecacca8c89ebe98d45ab68/common/src/state_machine/timeout.rs#L25-L32
* `TimeoutLedger` contains a `BTreeMap` that indexes IDs by `Instant` and a
  `HashMap` that indexes `Instant`s by ID This double indexing allows timeout
  cancellations to go through without providing the `Instant` that they were
  meant to remove

