# Serenity Benches

This document contains the results of running benchmarks on the
[Lighthouse](http://github.com/sigp/ligthouse) Eth2.0 client.

Included alongside the benchmarks are descriptions of the functions being
measured, lessons learned about optimising and instructions on how to run the
benchmarks locally.

## Table of Contents:

- [Results](#results)
- [Per-epoch processing details](#per-epoch-processing-1)
- [Per-block processing details](#per-block-processing-1)
- [Computer details](#computer-details)
- [Running the benchmarks](#running-the-benchmarks)


## Info/Disclaimers:

- All benchmarks were performed using the
	[Lighthouse](http://github.com/sigp/ligthouse) Eth2 client (Rust).
- The majority of these benches are stress-tests (worst case
  scenarios).
- Benchmarks are not limited to a single-core -- concurrent functions will run
	across multiple cores.
- These benchmarks are up-to-date with spec [v0.4.0](https://github.com/ethereum/eth2.0-specs/tree/v0.4.0).
- It is almost a certainty that this implementation will have bugs -- test vectors
	for these functions are soon-to-be-released, until then it is very difficult
	to find bugs.
- All benches are purely functional -- there are no "read from disk" or "fetch
	from network" times included.
- There has been only a mild amount of effort taken to optimise this code.
	There's likely a lot of room for improvement.
- Our optimisations are currently focussed towards an all-vaildators-active
	scenario -- we will likely need to adjust our optimisations to suit a more
	diverse set of scenarios.
- There is no tree hashing caching in any tests.
- The ordering of some of the columns is a little odd, sorry.

## Results:

There are three scenarios benched. Each should be exactly the same, except the
validator count is changed. The following codes map to each scenario:

- **16K**: 16,384 validators (all active).
- **300K**: 300,032 validators (all active).
- **4M**: 4,000,000 validators (all active).

_Note: when a result is `-` it means the value has not changed since the
benchmark with the next smallest validator count. E.g., the value for 300k
validators hasn't changed from the value for 16k validators._

### Epoch Processing

### Desktop

|Benchmark| 16K [Desktop](#desktop) | 300K [Desktop](#desktop)
|-|-|-|
|  [process_eth1_data](#process_eth1_data) | 294.34 ns | 1.1548 μs
|  [initialize_validator_statuses](#initialize_validator_statuses) | 982.77 μs | 33.931 ms
|  [process_justification](#process_justification) | 276.59 ns | 1.2140 μs
|  [process_crosslinks](#process_crosslinks) | 1.0204 ms | 30.306 ms
|  [process_rewards_and_penalties](#process_rewards_and_penalties) | 804.03 μs | 15.609 ms
|  *[process_ejections](#process_ejections) | 187.60 μs | 3.2257 ms
|  *[process_validator_registry](#process_validator_registry) | 187.88 μs | 8.2305 ms
|  ^[update_active_tree_index_roots](#update_active_tree_index_roots) | 3.0462 ms | 82.961 ms
|  *[update_latest_slashed_balances](#update_latest_slashed_balances) | 400.25 ns | 952.32 ns
|  [clean_attestations](#clean_attestations) | 23.615 μs | 322.11 μs
|  **[per_epoch_processing](#per_epoch_processing)** | **6.4554 ms** | **187.98 ms**

### Laptop

|Benchmark | 16K [Laptop](#laptop) | 300K [Laptop](#laptop)
|-|-|-|
|  [process_eth1_data](#process_eth1_data) | 415.11 ns | 1.3387 μs |
|  [initialize_validator_statuses](#initialize_validator_statuses) | 1.2862 ms | 46.470 ms |
|  [process_justification](#process_justification) | 411.57 ns | 1.3773 μs |
|  [process_crosslinks](#process_crosslinks) | 1.2308 ms | 39.095 ms |
|  [process_rewards_and_penalties](#process_rewards_and_penalties) | 1.0781 ms | 22.406 ms |
|  *[process_ejections](#process_ejections) | 171.52 μs | 3.0041 ms |
|  *[process_validator_registry](#process_validator_registry) | 376.89 μs | 9.9943 ms |
|  ^[update_active_tree_index_roots](#update_active_tree_index_roots) | 4.5167 μs | 119.122 ms |
|  *[update_latest_slashed_balances](#update_latest_slashed_balances) | 606.76 ns | 1.3709 μs |
|  [clean_attestations](#clean_attestations) | 28.675 μs | 248.50 μs |
|  **[per_epoch_processing](#per_epoch_processing)** | **9.5174 ms** | **262.29 ms** |

_* We did not add an ejections or registry changes. These times are best-case
(not worst-case)._

^ _This time is tree-hashing the entire active validator indices. Technically it
doesn't need to run at all because they haven't changed since the previous
epoch._

### Block Processing

The block-processing benches are tagged with the following codes:

- **WC**: Worst-case.
  - Maximum operations (e.g., `MAX_ATTESTATIONS` attestations, etc.)
- **RC**: Reasonable-case
  - 0 slashings
  - 16 full attestations
  - 2 deposits
  - 2 exits
  - 2 transfers

#### Desktop

|Benchmark| 16K WC [Desktop](#desktop) | 300K WC [Desktop](#desktop) | 300K RC [Desktop](#desktop) | 4M RC [Desktop](#desktop) | 4M WC [Desktop](#desktop)
|-|-|-|-|-|-|
|  [verify_block_signature](#verify_block_signature) | 5.3024 ms| - | - | - | - |
|  [process_randao](#process_randao) | 5.2679 ms | - | - | - | - |
|  [process_eth1_data](#process_eth1_data) | 229.31 ns | 1.4178 μs | - | - | -|
|  [process_proposer_slashings](#process_proposer_slashings) | 37.108 ms | - | 1.4005 μs | - | - |
|  [process_attester_slashings](#process_attester_slashings) | 147.83 ms | - | 1.3960 μs | - | - |
|  [process_attestations](#process_attestations) | 193.86 ms | 309.15ms | 48.02 ms | 393.63 ms | 2.7259 s
|  [*process_deposits](#process_deposits) | 18.492 ms | - | 8.0843 ms | 20.233 ms | 30.160 s |
|  [process_exits](#process_exits) | 18.835 ms | - | 6.6976 ms | - | - |
|  [process_transfers](#process_transfers) | 18.686 ms | - | 6.4966 ms | - | - |
|  [**per_block_processing**](#per_block_processing) | **440.63 ms** | **553.25 ms** | **79.544 ms** | **433.55 ms** | **2.9739 s**

_Note: 16K RC per_block_processing comes in at 64.077 ms._

#### Laptop

|Benchmark| 16K WC [Laptop](#laptop) | 300K WC [Laptop](#laptop) | 300K RC [Laptop](#laptop) | 4M RC [Laptop](#laptop) | 4M WC [Laptop](#laptop)
|-|-|-|-|-|-|
|  [verify_block_signature](#verify_block_signature) | 7.1359 ms  | - | - | - | - |
|  [process_randao](#process_randao) | 7.0675 ms | - | - | - | - |
|  [process_eth1_data](#process_eth1_data) | 330.12 ns | 1.5468 μs | - | 4.2278 μs | - |
|  [process_proposer_slashings](#process_proposer_slashings) | 119.46 ms | - | 1.8209 μs | - | - |
|  [process_attester_slashings](#process_attester_slashings) | 211.74 ms | - | 1.8208 μs | - | - |
|  [process_attestations](#process_attestations) | 833.23 ms | 1.3424 s | 159.46 ms | 1.4417 s | 11.920 s |
|  [*process_deposits](#process_deposits) | 60.300 ms | - | 9.7835 ms | 23.987 ms | 74.589 ms |
|  [process_exits](#process_exits) | 60.163 ms | - | 7.8856 ms - | - |
|  [process_transfers](#process_transfers) | 60.163 ms | - | 8.2653 ms | - | - |
|  [**per_block_processing**](#per_block_processing) | **1.3885 s** | **1.819 s** | **207.07 ms** | **1.5126 s** | 12.494 s |

_* Merkle roots are not verified._

_Note: 16K RC per_block_processing comes in at 152.70 ms._

_Note: 4M WC per_block_processing comes in at ~12 s, where 11.920 s is
attestation verification._

### Cache Builds

All the previous benchmarks were done with pre-built committee and pubkey
caches. These are the times to build those caches.

|Benchmark| 16K [Desktop](#desktop) | 300K [Desktop](#desktop) | 16K [Laptop](#laptop) | 300K [Laptop](#laptop) | 4M [Desktop](#desktop)
|-|-|-|-|-|-|
|  [build_epoch_committee_cache](#epoch-cache-builds) | 7.4939 ms | 264.02 ms | 19.412 ms | 402.96 ms | 3.9832 s |
|  [build_pubkey_cache](#pubkey-cache-builds) | 14.874 ms | 339.41 ms | 30.178 ms | 488.15 ms | 5.0710 s |

_Note: I once benched shuffling 4M validators at 1.2s. Epoch cache build times
seem unreasonably high, I suspect they can be improved.._


### Tree Hashing

This is a full tree-hash without caching.

|Benchmark| 16K [Desktop](#desktop) | 300K [Desktop](#desktop) | 16K [Laptop](#laptop) | 300K [Laptop](#laptop)
|-|-|-|-|-|
|  [tree_hash_state](#tree_hash_state) | 81.444 ms | 1.3884 s | 121.80 ms | 1.8679 s |
|  [tree_hash_block](#tree_hash_block) | 3.0570 ms (WC) | 3.4629 ms (WC) | 4.5881 ms (WC) | 4.7180 ms (WC) |


### BLS

|Benchmark| [Desktop](#desktop) | [Laptop](#laptop) |
|-|-|-|
|  verify_a_signature | 5.2158 ms | 6.7476 ms |
|  aggregate_a_public_key | 32.311 μs |42.000 μs
|  pubkey_from_uncompressed_bytes | 826.60 ns | NA
|  pubkey_from_compressed_bytes |  39.934 μs | NA

## Lessons Learned

Here we share the lessons we learned during optimisation. We hope this
information will save time for other implementers.

### Per-Epoch Processing

We found it useful to consider rewards and penalties as a map against
`state.validator_balances`. This allowed us to very easily use
[rayon](https://github.com/rayon-rs/rayon) to do this map concurrently. The
specification is well designed for this purpose -- updating one validator's
balance only mutates state related to that validator.

Where we found speed improvements:

- Processing the validator rewards in parallel (as mentioned earlier). This
- Removing all `~O(n^2)` time blow-ups like `inclusion_distance`.

### Per-Block Processing

Most notable gains here were from introducing concurrency where hashing or
signature verification is involved.

We found a ~50% increase in
processing `AttesterSlashings` by first verifying all `SlashableAttestations`
in parallel before verifying each `AttesterSlashing`.

We found some difficulties in quickly generating a `HashMap` for checking that some
pubkey already exists in the validator registry. We found that going from our
BLS libraries object into bytes (for hashing) was rather slow. As such, we now
maintain a `PublicKey -> ValidatorIndex` map alongside the state.

# Details

## Per-Epoch Processing

## `process_eth1_data`

#### Benching Setup

Function is run with the following inputs:

- There is only a single `Eth1DataVote`.

#### Function Description

1. Attempts to find an `Eth1Data` with a super-majority vote and update the state.


## `initialize_validator_statuses`

#### Benching Setup

Function is run with the following inputs:

- A single pending attestation with full participation for each committee that
  is able to include an attestation in the state.

#### Function Description

Performs a loop over all validators and determines

- Active validators
- Current balances
- Previous balances

Then loops through all the attestations and determines:

- Current epoch attesters
- Current epoch boundary attesters
- Previous epoch attesters
- Previous epoch boundary attesters
- Previous epoch head attesters

This info is used later on during epoch processing.

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
1. Does as a loop across all participants in all committees in the previous
   epoch and determines if/how they should be rewarded for winning root
   participation.
1. Does an parallel map across `state.validator_balances` and applies the
   [justification and
   finalization](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization),
   [attestation
   inclusion](https://github.com/ethereum/eth2.0-specs/blob/0.4.0/specs/core/0_beacon-chain.md#justification-and-finalization)
   and [crosslinks](https://github.com/ethereum/eth2.0-specs/blob/v0.4.0/specs/core/0_beacon-chain.md#crosslinks-1) rewards.
1. Loops through all validators and rewards the block proposer if they included
   a block.


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
	1. Updates the already-built `PublicKey -> ValidatorIndex` map (for look-up of pre-existing
		validators).
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

## Epoch Cache Builds

Our epoch caches are on a per-epoch basis and contain:

- Each committee for each slot of the epoch.
- A map of `ValidatorIndex -> (AttestationSlot, AttestationShard,
	CommitteeIndex)` for easy access of a validator's attestation duties.
- A map of `Shard -> (EpochCommitteesIndex, SlotCommitteesIndex)` for easy
	access of a shard's committee.

## Pubkey Cache Builds

The pubkey cache is a map of `PublicKey -> ValidatorIndex`. It is used to speed
up deposit processing (checking to see if a pubkey exists in the validator
registry).

It is presently very slow because we need to convert our BLS libraries
`PublicKey` struct into bytes. We have chosen to use uncompressed bytes for
the key of the hashmap as it is quicker.

## Tree Hashing

## `tree_hash_state`

#### Benching Setup

Function is run with the following inputs:

- The state which is used in the epoch processing tests.
	See [per_epoch_processing](#per_epoch_processing) for details.

#### Function Description

1. Hashes as per the [tree hash](https://github.com/ethereum/eth2.0-specs/blob/v0.4.0/specs/simple-serialize.md#tree-hash) spec.

## `tree_hash_block`

#### Benching Setup

Function is run with the following inputs:

- The block which is used in the block processing tests.
	See [per_block_processing](#per_block_processing) for details.

#### Function Description

1. Hashes as per the [tree hash](https://github.com/ethereum/eth2.0-specs/blob/v0.4.0/specs/simple-serialize.md#tree-hash) spec.

## Computer Details

### Desktop

- Arch Linux
- 6-core [i7-8700K](https://ark.intel.com/content/www/us/en/ark/products/126684/intel-core-i7-8700k-processor-12m-cache-up-to-4-70-ghz.html).
- 16GB DDR4 2400 Mhz.

### Laptop

- Lenovo X1 Carbon 5th Gen
- Arch Linux
- 2-core [i5 7300U](https://ark.intel.com/content/www/us/en/ark/products/97472/intel-core-i5-7300u-processor-3m-cache-up-to-3-50-ghz.html)
- 16GB LPDDR3 1866 Mhz.


## Running the benchmarks

You can run these benchmarks on your local machine if you're an advanced user. There are three main steps:

1. **Setup the repo** by cloning it and ensuring you have all the deps.
1. **Generate a file of keypairs** to speed-up the setup for the benches. This can
   be skipped but it will greatly increase your setup time (e.g., it takes
   5 mins to generate 4M keypairs on 6-cores however it takes 1s to read them
   from file.)
1. **Run the benches**

### 1. Setup the repo

1. Clone the https://github.com/sigp/lighthouse repo at [`sane-case`](https://github.com/sigp/lighthouse/tree/sane-case).
1. Follow the [Running](https://github.com/sigp/lighthouse#running)
   instructions. You just need a standard Rust setup (`rustup`) and
   additionally `clang` and `protobuf`.
1. Check you can build by running the tests: `$ cargo test --all`

### 2. Generate the keypairs file

This will create the following file: `$HOME/.lighthouse/keypairs.raw_keypairs`.
It contains uncompressed, unencrypted BLS keypairs and massively speeds up setup
times for benches.

1. Navigate to `beacon_node/beacon_chain/test_harness`
1. Run `cargo run --release -- gen_keys -n KEYS` where `KEYS` is the maximum
   number of validators you wish to bench with. You'll get a panic during
   benches if your keyfile isn't big enough.

_Note: if you omit `--release` key generation will be very slow._

_Note: it takes my [desktop](#desktop) 5 minutes to build 4M keys._

### 3. Run the benches

1. Navigate to `eth2/state_processing`
1. Run `cargo bench`

_Note: you can filter benches. Using `cargo bench block` will only run benches
where `block` is in the title._

If you want to change the number of validators, change the
['VALIDATOR_COUNT'](https://github.com/sigp/lighthouse/blob/efd56ebe375c73dd658e7c86c50a5431e22ad77f/eth2/state_processing/benches/benches.rs#L10)
variable. Note, if you don't pick your number of validators correctly you might
get a panic about "Each attestation in the state should have full
participation" from [this
assertion](https://github.com/sigp/lighthouse/blob/efd56ebe375c73dd658e7c86c50a5431e22ad77f/eth2/state_processing/benches/bench_epoch_processing.rs#L55-L59).
Either choose a validator count that works for that assertion, or comment it
out.
