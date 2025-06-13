# Quiescence details

The high level description of quiescence can be found in the [Quiescence HIP](../../HIP/hip-xxxx-quiescence.md).

## Changes

Rule 4:

- After each consensus round is handled, the consensus module can ask execution
  for the next TCT.

### Functionality changes

Changes needed for [Rule 1](../../HIP/hip-xxxx-quiescence.md#rule-1-transactions-that-need-to-reach-consensus):

- Each transaction we store in an event or the transaction pool needs to have an additional boolean that indicates if it
  needs to reach consensus or not.
- The event creator module should keep a set of all non-ancient non-consensus events. If any of these events have
  transactions that need to reach consensus, we should not quiesce. This means that the event creator module also needs
  to receive consensus rounds.

Changes needed for [Rule 2](../../HIP/hip-xxxx-quiescence.md#rule-2-fully-signed-blocks):

- The event creator module should store the round number of the latest consensus transaction.
- The event creator module should also store the latest block number that is fully signed.
- If the latest fully signed block is less than the latest consensus transaction round, the event creator should not
  quiesce.

Changes needed for [Rule 3](../../HIP/hip-xxxx-quiescence.md#rule-3-target-consensus-timestamp-tct):

- The event creator module should store the latest TCT.
- If the latest TCT is less than the current time plus the configured `tctDuration`, the event creator should not
  quiesce.

Other changes:

- If all the above conditions are met, the event creator should stop creating events.
- The event creator should update the platform status to `QUIESCED` when appropriate.
- The event creator should create a QB if there are pending transactions in the transaction pool, and there are
  restrictions on creating events that advance consensus.

### Wiring changes

- Event creator needs to receive consensus rounds (how will the event creator know which events are consensus after a
  reconnect?)
- Event creator needs to receive info about which blocks are fully signed (after reconnect as well)
- Event creator needs to receive the latest TCT (after reconnect as well)
- Event creator needs to update the platform status

### API

Changes needed for [Rule 1](../../HIP/hip-xxxx-quiescence.md#rule-1-transactions-that-need-to-reach-consensus):

- An additional API is needed for the consensus module to determine if a transaction needs to reach consensus or not.
  When we receive transactions as part of events through gossip, we need to check if they need to reach consensus. This
  should be done with a new method in the `SwirldMain` interface with a definition like:
  `boolean needsToReachConsensus(Bytes transaction)`. This method should become part of `ApplicationCallbacks`.

Changes needed for [Rule 2](../../HIP/hip-xxxx-quiescence.md#rule-2-fully-signed-blocks):

- In order for the consensus module to know when a block is fully signed, the execution module would need to notify it,
  an additional API is needed for this. (How would this API look like?)

Changes needed for [Rule 3](../../HIP/hip-xxxx-quiescence.md#rule-3-target-consensus-timestamp-tct):

- Execution should provide the latest TCT to the consensus module. This can be done by adding a new field to
  `TransactionHandlerResult`.

## Side effects of quiescence

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

## Configuration

The following configuration record should be introduced:

```java

import java.time.Duration;

/**
 * Configuration for quiescence.
 * @param enabled       indicates if quiescence is enabled
 * @param tctDuration   the amount of time before the target consensus timestamp (TCT) when quiescence should not be 
 *                      active
 */
@ConfigData("quiescence")
public record QuiescenceConfig(
        @ConfigProperty(value = "enabled", defaultValue = "true")
        boolean enabled,
        @ConfigProperty(value = "tctDuration", defaultValue = "5s")
        Duration tctDuration) {
}

```

## Metrics

The following metrics should be added:

- `numTransNeedCons` the number of non-ancient transactions that need to reach consensus.
- `lastSignedBlock` the latest block number that is fully signed

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

- If all the quiescence rules are met, it should stop creating events.
- If the `wallClockTime` + `tctDuration` is less than the next TCT, it should create events regardless of any
  transactions that are pending or need to reach consensus.
- If it should not quiesce until all transactions are in a fully signed block.
- If it is quiesced, there is a pending transaction that needs to reach consensus, and there are no eligible parents; it
  should create a QB event with that transaction.
- If a QB is created, it should not create another QB with the same self-parent even if there are pending transactions
  that need to reach consensus.

### Integration Tests

An Otter test should be created that submits transactions periodically and validates the following:

- Event creation is stopped when transactions stop being submitted
- Event creation is started again when a transaction is submitted
- Platform status is updated to `QUIESCED` when event creation is stopped
- A freeze should be set at the end of this test, and the freeze should occur even if no transactions are submitted
