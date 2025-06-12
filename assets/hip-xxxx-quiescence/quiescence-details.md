# Quiescence details

The high level description of quiescence can be found in the [Quiescence HIP](../../HIP/hip-xxxx-quiescence.md).


### Side effects of quiescence

Various parts of the system assume that events are constantly being created and consensus is always advancing. With
quiescence, this is not the case. This means that various parts of the system need to be modified to account for this.

- The `SignedStateSentinel` uses wall-clock time to determine if a signed state is old. This will need to be modified to
  use the take quiescence into account.
- `PcesConfig.minimumRetentionPeriod` uses wall-clock time to determine how long to keep events.
- The platform status `ACTIVE` currently moves to `CHECKING` based on wall-clock time. We will need to add a
  `QUIESCED` status.
- Metrics can produce misleading information due to the pause in event creation. Example: `secC2C` tracks the amount of
  time that passes from an event being created to it reaching consensus. Ordinarily, this is a few seconds. If this
  value spikes, it is usually an indicator of a performance issue in the network. If quiescence is not taken into
  account, this value will spike to the amount of time the network was quiesced, which would look like an issue, but is
  expected behavior.
- NOTE FOR REVIEWERS: I probably haven't thought of all the side effects yet, please add any you can think of.

---

## Changes

DUMP:

Rule 2:

- The consensus module will need to be able to distinguish between signature transactions and user transactions.
  An additional API is needed for this.

Rule 3:

- In order for the consensus module to know when a block is fully signed, the
  execution module would need to notify it. An additional API is needed for this as well.

Rule 4:

- After each consensus round is handled, the consensus module can ask execution
  for the next TCT.

### Architecture and/or Components

- Each transaction we store in an event or the transaction pool needs to have an additional boolean that indicates if it
  needs to reach consensus or not.
- We will need functionality to detect non-ancient transactions that need to reach consensus. This should be part of the
  event creation module.
- The event creator module should be updated with information about consensus transactions that do not have a boundary
  round after them. This will be used to determine if the network can quiesce or not. This information needs to be sent
  from the transaction handler to the event creator module.
- The event creator should stop creating events if there are no transactions that need to reach consensus, unless there
  are pending transactions, or there is less than 1 minute before the freeze time according to the wall-clock.
- The event creator should update the platform status to `QUIESCED` when appropriate.
- The event creator should create a QB if there are transactions that need to reach consensus, and there are
  restrictions on creating events that advance consensus.

### Public API

An additional API is needed for the consensus module to determine if a transaction needs to reach consensus or not.

- When we receive transactions as part of events through gossip, we need to check if they need to reach consensus. This
  should be done with a new method in the `SwirldMain` interface with a definition like:
  `boolean needsToReachConsensus(Bytes transaction)`. This method should become part of `ApplicationCallbacks`.
- When transactions are being submitted to the platform before being put into an event, we also need this same
  information. So the method in `Platform` should be changed to:
  `boolean createTransaction(byte[] transaction, boolean needsToReachConsensus)`.

### Configuration

A new configuration option is needed to enable/disable quiescence.

### Metrics

The following metrics should be added:

- `numTransNeedCons` the number of non-ancient transactions that need to reach consensus.

The following metrics should be modified:

- `secC2C` & `secR2C` should be modified to only track events that have transactions that need to reach consensus. If
  this is not done, these metrics will have huge spikes when quiescence is broken.

The following metrics should be removed since they would need to be modified, but are not used:

- `secC2RC`
- `secSC2T`
- `secOR2T`
- `secR2F`
- `secR2nR`

---

## Test Plan

### Unit Tests

New unit tests for the event creator should be written for the following scenarios:

- If no non-ancient transactions need to reach consensus, there are no consensus transactions without a boundary round
  after them, and there is no upcoming freeze; it should stop creating events.
- If the wall-clock time is less than 1 minute before the freeze time, it should create events regardless of any
  transactions that are pending or need to reach consensus.
- If it is quiesced and there is a pending transaction that does not need to reach consensus (a state/block signature),
  it should create only a single event with that transaction.
- If it is quiesced and there is a pending transaction that needs to reach consensus, it should create a QB event with
  that transaction.
- If a QB is created, it should not create another QB with the same self-parent even if there are pending transactions
  that need to reach consensus.

### Integration Tests

An Otter test should be created that submits transactions periodically and validates the following:

- event creation is stopped when there are no transactions and no freeze is upcoming
- event creation is started again when a transaction is submitted
- platform status is updated to `QUIESCED` when event creation is stopped
- a freeze should be set at the end of this test, and the freeze should occur even if no transactions are submitted
