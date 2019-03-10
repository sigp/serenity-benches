# Serenity Benches

TODO:

- Ensure deposit merkle root is verified.
- Add some ejections?
- Add tree hash for state root.


This document contains the results of running benchmarks on the
[Lighthouse](http://github.com/sigp/ligthouse) Eth2.0 client.

For each benchmark, it contains a description of the parameters supplied to the
benchmark and the workings of the function being metered.


## Info/Disclaimers:

- All benchmarks were performed using the
	[Lighthouse](http://github.com/sigp/ligthouse) written in Rust.
- Benchmarks are not limited to a single-core -- concurrent functions will run
	across multiple cores.
- These benchmarks are fully up-to-date with spec [v0.4.0](https://github.com/ethereum/eth2.0-specs/tree/v0.4.0).
- It is almost a certainty that this implementation has bugs -- test vectors
	for these functions are soon-to-be-release, until then it is very difficult
	to find bugs.
- There has been _some_ effort taken to optimise this code. This is our rough
	optimization ethos:
	- Use concurrency if it's easy.
	- Try and use hash maps/sets to reduce CPU time.
	- Don't be concerned about memory allocation (do that later).
	- Don't be concerned about detailed language/compiler/architecture
		optimisations (do that later).
	- Only worry about optimizations that save ~10% running time.
	- Don't be radical with restructuring -- keep it simple.
- The copied-as-is-from-spec, non-optimized version took 7+ seconds for a state
	transition. We have it down to .38 seconds and I suspect there's still
	room for optimisation, especially if you're more radical with optimisations.


## Results:

### Epoch Processing (16,384 validators)

|Benchmark| Time (Desktop) |
|-|-|
|  [calculate_active_validator_indices](#calculate_active_validator_indices) | 140.89 μs |
|  [calculate_current_total_balance](#calculate_current_total_balance) | 30.614 μs |
|  [calculate_previous_total_balance](#calculate_previous_total_balance) | 95.817 μs |
|  [process_eth1_data](#process_eth1_data) | 20.497 μs |
|  [calculate_attester_sets](#calculate_attester_sets) | 3.8771 ms |
|  [process_justification](#process_justification) | 22.718 μs |
|  [process_crosslinks](#process_crosslinks) | 1.1468 ms |
|  [process_rewards_and_penalties](#process_rewards_and_penalties) | 358.64 ms |
|  [process_ejections](#process_ejections) | 112.86 μs |
|  [process_validator_registry](#process_validator_registry) | 187.88 μs |
|  [update_active_tree_index_roots](#update_active_tree_index_roots) | 1.8973 ms |
|  [update_latest_slashed_balances](#update_latest_slashed_balances) | 21.043 μs |
|  [clean_attestations](#clean_attestations) | 34.500 μs |
|  **[per_epoch_processing](#per_epoch_processing)** | **383.95 ms** |

### Block Processing (16,384 validators)

This is a "worst-case" block. It has the maximum number of all operations. Some
care was taken to ensure the included operations are as complex as possible,
however it was not a priority.

|Benchmark| Time (Desktop) |
|-|-|
|  [verify_block_signature](#verify_block_signature) | 5.3024 ms |
|  [process_randao](#process_randao) | 5.2679 ms |
|  [process_eth1_data](#process_eth1_data) | 17.342 μs |
|  [process_proposer_slashings](#process_proposer_slashings) | 37.108 ms |
|  [process_attester_slashings](#process_attester_slashings) | 147.83 ms |
|  [process_attestations](#process_attestations) | 193.86 ms |
|  [process_deposits](#process_deposits) | 53.734 ms |
|  [process_exits](#process_exits) | 19.022 ms |
|  [process_transfers](#process_transfers) | 18.686 ms |
|  [**per_block_processing**](#per_block_processing) | **478.98 ms** |

### Cache Builds (16,384 validators)

All the previous benchmarks were done with a pre-built committee cache. These
are the times to build that cache.

|Benchmark| Time (Desktop) |
|-|-|
|  build_previous_state_cache | 9.1979 ms |
|  build_current_state_cache | 9.1075 ms |


## Per-Epoch Processing



## `calculate_active_validator_indices`

#### Benching Setup

Function is run with the following inputs:

- All validators are active.

#### Function Description

1. Iterates the validator registry and returns all active validators.




## `calculate_current_total_balance`

#### Benching Setup

Function is run with the following inputs:

- All validators are active.

#### Function Description

1. Iterates validator balances and sums the effective balances for each active
   validator (active validators is pre-computed).



## `calculate_previous_total_balance`

#### Benching Setup

Function is run with the following inputs:

- All validators are active.

#### Function Description

1. Determines the active validators for the previous epoch.
1. Iterates validator balances and sums the effective balances for each active
   validator.



## `process_eth1_data`

#### Benching Setup

Function is run with the following inputs:

- There is only a single `Eth1DataVote`.

#### Function Description

1. Attempts to find an `Eth1Data` with a super-majority vote and update the state.



## `calculate_attester_sets`

#### Benching Setup

Function is run with the following inputs:

- A single pending attestation with full participation for each committee that
  is able to include an attestation in the state.

#### Function Description

Builds an empty `Attesters` struct which contains a hashset of participating
validator indices and a total balances value for each of the following groups:

- Current epoch attesters
- Current epoch boundary attesters
- Previous epoch attesters
- Previous epoch boundary attesters
- Previous epoch head attesters

Iterates through each `PendingAttestation` in the state, builds a list of all
participants in that attestation and obtains their total balance. It then
analyses the attestation and updates the appropriate `Attesters` struct.



## `process_justification`

#### Benching Setup

Function is run with the following inputs:

- Previous justified epoch is the previous epoch - 1.
- Justified epoch is the previous epoch.
- Justification bitfield has all bits set true.
- Previous and current boundary attesting balances are 100% of respective total balances.

#### Function Description

1. Performs the actions described [here](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification).



## `process_crosslinks`

#### Benching Setup

Function is run with the following inputs:

- A single pending attestation with full participation for each committee that
  is able to include an attestation in the state.

#### Function Description

1. Attempts to determine the winning_root for each shard.
1. Updates `state_latest_crosslinks` where there is a winning root.
1. Builds a hashmap of `Shard -> WinningRoot` for later lookups.



## `process_rewards_and_penalties`

#### Benching Setup

Function is run with the following inputs:

- All validators are active.
- All validators performed their duties exactly and agreed upon everything.
- Epochs since finality is 4.

#### Function Description

1. Does some misc setups (determine latest epoch attestations, reward
   quotients, etc).
1. Does an parallel map across `state.validator_balances` and applies the
   [justification and
   finalization](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization)
   and [attestation
   inclusion](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization)
   rewards.
1. Loops (in series) through all slots in the previous epoch and applies the
   [crosslinks](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization)
   rewards to each validator. Uses the previously generated `Shard ->
   WinningRoot` hashmap for memoized `winning_root` lookups.



## `process_ejections`

#### Benching Setup

Function is run with the following inputs:

- There are no ejections.

#### Function Description

1. Performs the actions described [here](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization).



## `process_validator_registry`

#### Benching Setup

Function is run with the following inputs:

- `state.finalized_epoch > state.validator_registry_update_epoch`
- The epoch for all crosslinks is ` > state.validator_registry_update_epoch`
- There are no slashings.
- There are no exits.

#### Function Description

1. Performs the actions described [here](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization).



## `update_active_tree_index_roots`

#### Benching Setup

Function is run with the following inputs:

- All validators are active.

#### Function Description

1. Performs a tree-hash on the active validator indices.
1. Updates `state.latest_active_index_roots`.



## `update_latest_slashed_balances`

#### Benching Setup

NA.

#### Function Description

1. Rotates `state.latest_slashed_balances`.



## `clean_attestations`

#### Benching Setup

NA.

#### Function Description

1. Removes all attestations from the previous epoch from
   `state.pending_attestations`.

## `per_epoch_processing`

#### Benching Setup

This is a full run of the epoch processing function upon a state with has the
following characteristics:

- All validators are active.
- Epochs since finality == 4.
- Previous justified epoch == current_epoch - 3.
- Justified epoch == current_epoch - 2.
- Finalized epoch == current_epoch - 3.
- Full justification bitfield.
- All validators performed their duties exactly and agreed upon everything.
- Pending attestations includes a single attestation with full participation for each committee that
  is able to include an attestation in the state.

Note: this _does not_ include tree-hashing the state.

#### Function Description

1. Runs all of the functions listed in this section (except tree hashing the
   state). _Hopefully_ fully adheres to the [per-slot processing](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization).









## Per-Block Processing

This section provides detail on each benchmark. The following is provided for
each benchmark:

- **Benching Setup**: details on the setup for the benchmark. E.g., how many objects,
	were verified etc.
- **Function Description**: a brief description how the benched function
	operates. E.g., concurrent operations, etc.

### `verify_block_signature`

#### Benching Setup

Function is run with the following inputs:

- A valid block signature.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/3f988493622ab0221649b91682e4ad8296f86542/eth2/state_processing/src/per_block_processing.rs#L100)

1. Determines the slot's block producer.
1. Produces a signed root of the block's `Proposal`.
1. Verifies the block producer signed the proposal root.

### `process_randao`

#### Benching Setup

Function is run with the following inputs:

- A valid `block.randao_reveal`.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L134)

1. Determines the block proposer for the present slot.
1. Verifies that `randao_reveal` is the proposer's signature across
	`hash_tree_root(state.current_epoch())`.
1. Updates `state.latest_randao_mixes` with the new reveal.

### `process_eth1_data`

#### Benching Setup

Function is run with the following inputs:

- A matching `Eth1DataVote` does not exist in the state.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/3f988493622ab0221649b91682e4ad8296f86542/eth2/state_processing/src/per_block_processing.rs#L168)

1. Searches for a matching `Eth1DataVote` in the state.
1. If exists, increments the `vote` for that data. Otherwise adds a new
	`Eth1DataVote`.

### `process_proposer_slashings`

#### Benching Setup

Function is run with the following inputs:

- `MAX_PROPOSER_COUNT` count valid `ProposerSlashings`.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L226)

1. Verifies each `ProposerSlashing` in parallel.
1. If all are valid, slashes each validator sequentially.

### `process_attester_slashings`

#### Benching Setup

Function is run with the following inputs:

- `MAX_ATTESTER_SLASHINGS` count valid `AttesterSlashings`.
- Each `AttesterSlashing` has `MAX_INDICES_PER_SLASHABLE_VOTE` indices.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/3f988493622ab0221649b91682e4ad8296f86542/eth2/state_processing/src/per_block_processing.rs#L258)

1. Builds a list of references to each `SlashableAttestation` in the
	`AttesterSlashings` (there are two in each).
1. Verifies all the `SlashableAttestations` in parallel.
1. If all `SlashableAttestations` are valid, performs the following actions
	sequentially:
	- Verifies each `AttesterSlashing`.
	- Slashes all appropriate validators.

### `process_attestations`

#### Benching Setup

Function is run with the following inputs:

- `MAX_ATTESTATIONS` count valid `Attestations`.  In the case that there are
- not `MAX_ATTESTATION` committees available for the
	block, committees are split into two signing-groups until there are enough
	`Attestations`.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L316)

1. Ensures the previous epoch cache is built (it is always built for these
	benches).
1. Verifies each `Attestation` in parallel.
1. If all are valid, updates the state sequentially.


## `process_deposits`

Function is run with the following inputs:

- `MAX_DEPOSITS` count valid `Deposits`.

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L357)

#### Function Description

1. Partially-verifies each deposit in parallel.
1. If all were valid, performs the following actions sequentially:
	1. Builds a hashmap of `PublicKey -> ValidatorIndex` for look-up of pre-existing
		validators.
	1. Verifies the deposit index against `state.deposit_index`.
	1. Loads the validator index (if any) from the hashmap.
	1. If validator exists, checks withdrawal credentials and updates balance.
		Otherwise, creates new validator.

## `process_exits`

#### Benching Setup

Function is run with the following inputs:

- `MAX_VOLUNTARY_EXITS` count valid `VoluntaryExits`.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L426)

1. Verifies each exit in parallel.
1. Updates the state sequentially.


## `process_transfers`

#### Benching Setup

Function is run with the following inputs:

- `MAX_TRANSFERS` count valid `Transfers`.
- Each transfer transfers 1 Gwei from a withdrawn validator to itself.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L458)

1. Verifies each transfer in parallel.
1. Updates the state sequentially.

## `per_block_processing`

Function is run with the following inputs:

- A valid block.
- Maximum possible operations (`Attestations`, `Transfers`, etc.). Each has the
	same characteristics as the `process_...` functions above.

#### Function Description

[Source code](https://github.com/sigp/lighthouse/blob/timing-report/eth2/state_processing/src/per_block_processing.rs#L39)

If any of the following functions return an error (e.g., invalid object)
execution returns immediately.

1. Checks the block slot against the state slot.
1. Ensures epoch caches are built (they are always built for these benches).
1. Verifies block signature
1. Processes the randao reveal.
1. Processes the eth1 data.
1. Processes proposer slashings.
1. Processes attester slashings.
1. Processes attestations.
1. Processes deposits.
1. Processes exits.
1. Processes transfers.
