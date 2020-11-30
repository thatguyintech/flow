# Storage Fees

| Status        | Proposed                                             |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Janez Podhostnik (janez.podhostnik@dapperlabs.com)   |
| **Sponsor**   | Janez Podhostnik (janez.podhostnik@dapperlabs.com)   |
| **Updated**   | 2020-10-13                                           |

## Objective

Limit the maximum amount of storage each account has to prevent storing huge amount of data on chain.

A minimum amount of storage will be provided to each account when it is created. Additional storage can be purchased by using the **StorageFees** smart contract. Storage can also be refunded using the same contract.

## Motivation

If the execution state grows too large, execution nodes will no longer be able to hold the execution state in memory, which is necessary to calculate the next execution state. When this happens, the execution node requirements (and the nodes themselves) will need to be updated (and it will cost more to operate them).

Limiting the amount of storage that an account can have will prevent the execution state from growing due to cases such as:

- Attacks to the system that attempt to store large amounts of data on chain with the explicit intent to slow it down or crash the network.
- Developers not considering storage costs in their smart contracts (because they don't need to).

## User Benefit

Limiting the maximum amount of storage each account has will have the long term benefit of reducing the growth of the execution state size, and thus reducing the growth of the execution node requirements. 

Prescribing a reasonable minimum amount of storage will allow almost everyone to deploy and use smart contracts normally. The minimum amount of storage must be at least large enough to allow the account to be usable, i.e. it must be large enough to store keys to allow a transaction signed with the key to be successfully executed, and it must be large enough to store the FLOW smart contract objects (vault, receiver capability, etc.) to allow the holding of FLOW tokens.

In case additional storage will be required on an account it can be purchased by depositing FLOW tokens to the storage fees smart contract.

## Design Proposal

### Overview

#### Account Storage Tracking (Execution Environment)

- For each account a 64-bit unsigned integer (uint64) value of storage used (`storage_used`) is stored in the execution state (i.e. the value is explicitly stored and not inferred). `storage_used` is the amount of storage the account is currently using (in bytes); The sum of data stored in the execution state for the account, i.e the sum of the sizes of all register values for the account.
- Each account has a `StorageCapacity` resource that has a 64-bit unsigned integer (uint64) value of storage capacity (`storage_capacity`). `storage_capacity` is the maximum amount of storage each account can store (in bytes). The minimum value is 100KB (as defined by the **StorageFees** smart contract).
- The `storage_used` value must be updated at the end of every transaction that modifies an account's storage
- The `storage_capacity` value is only writeable by the **StorageFees** smart contract. Other code only has read access.
- The `storage_used` is only writeable by the execution environment. Other code only has read access.
- If a transaction tries to store more data than the limit, the transaction should fail. The resulting error clearly states that the storage capacity was reached.

#### Storage Fees Smart Contract (part of the service account)

- The Storage Fees smart contract is a smart contract that collects storage deposits and provides functionality for accounts to acquire or release `storage_capacity`.
- The smallest unit of storage purchased is initially set to 10KB. This can be changed by the smart contract.
- Initially, the price of storage is 1kB for 1mF. This value will be assessed and updated as the chain evolves.
- Accounts can purchase more storage by calling a method on the **StorageFees**.
- An account can purchase storage for a different account.
- Accounts can reduce their `storage_capacity` in multiples of the storage chunk and be refunded. Doing so cannot reduce an account's `storage_capacity` below the account minimum storage capacity or below the account's `storage_used`. Storage reduction (and storage fee recovery) will be initially disabled to prevent the recovery of fees provided at no cost to new users as part of the network bootstrapping period. A global boolean value, managed by the Service Account, will control this feature and will be toggled when the bootstrapping period has ended.
- Calling `deductAccountCreationFee` on the **FlowServiceAccount** smart contract with an account, indirectly calls the **StorageFees** smart contract and purchases 100KB (minimum account storage capacity) for that account.

### Execution Environment Changes

The execution environment has three responsibilities:

1. Keeping the `storage_used` value up to date after every transaction.
1. Preventing accounts going over storage capacity.
1. Enabling the Flow Virtual Machine (`fvm`) package access to the `storage_capacity` and `storage_used` values:
    - Read access is permitted to both values for Cadence code
    - Write access to `storage_used` is only permitted from the execution environment.
    - Write access to `storage_capacity` is only permitted from the **StorageFees** smart contract


 `storage_used` is stored in the following registers:

```json
[
    {
        "owner": "$address",
        "controller": "",
        "key": "storage_used"
    }
]
```

#### Get and Update Storage Used

The amount of storage used by a single register on an address is the byte size of the register value and the register key.
`storage_used` by an address is also stored in the address' register. This could lead to a recursive problem of needing to update `storage_used` when `storage_used` is updated. Assuming that both `storage_capacity` and `storage_used` registers will be of type uint64, an thus size 8 (+ constant key size), removes this issue.

Two possible approaches of calculating the `storage_used` by an address were considered:

- Differential: on the current `View`, keep track of register size changes per address and sum them with the previous `storage_used`.
- Absolute: get all registers for each address (touched in the current `View`) and sum their size to get `storage_used`.

The decision was made to go with the differential approach, because iterating through all key value pairs in storage for each account touched would be too expensive and would not scale well.

##### Approach 1 - Differential

The basic principle of this approach is to update an address `storage_used` register every time a register size changes, by adding the size change to current `storage_used`:

- During `View.Set` and `View.Delete` if storage size changes, update `storage_used`.
- To increment the `storage_used` value, `GET` must be called to get the current value of `storage_used`. To reduce the number of register `GET`-s (and thus updating SPoCKs) the `View` will keep an internal cache of current `storage_used` values per account.

- Cons:
    1. Any error (due to code bugs) done in this calculation will be permanent. (e.g. If  `storage_used` change is off by 10kB because of a bug in calculation, fixing the bug will not correct the storage used)
        - This can be mitigated by adding a lot of test coverage for this functionality.
        - In the event a bad calculation is made and discovered, it can be corrected during a spork
    2. Migration is required for existing addresses to set current `storage_used`.
    3. If we decide to change what factors into `storage_used` there will need to be a migration (spork) to fix `storage_used` on all addresses.

Related PRs: 
- Storage used updated: https://github.com/onflow/flow-go/pull/76.
- Storage limiter: https://github.com/onflow/flow-go/pull/80.
- Expose storage used/capacity to fvm: TODO.
- Purchase minimum storage on address creation: TODO.

##### Approach 2 - Absolute (Rejected)

A new method will be added to the `View`'s interface that gets all `RegisterEntry`-s for an address

```go
// GetByOwner returns all register entries for this owner
GetByOwner(owner string) ([]flow.RegisterEntry, error)
```

With this method storage can be calculated as follows:

```py
register_entries, _ =  ledger.GetByOwner(owner)
storage_used = 0
for r in register_entries:
    storage_used+= len(r.Value)
```

- Cons:
    - The change to the `View` would require a lot of changes all the way down to `LedgerGetRegister` function (which already supports queries by key parts)
    - This approach is slower as all the registers need to be read

#### Create Account Changes

Account creation process currently goes through the following steps:

1. Deduct account creation fee by calling the `FlowServiceAccount` smart contract.
1. Set `exists` register which marks that the account exists.
1. Set key related registers.
1. Call `FlowServiceAccount` smart contract to initialize the FLOW token vault and receiver.

This process will be changed to account for setting `storage_capacity` and `storage_used` fields:

1. Set `storage_used` register to the size of itself
1. Set `exists` register which marks that the account exists.
1. Set key related registers.
1. Calling the `FlowServiceAccount` smart contract to:
    - Deduct account creation fees (including storage fees)
    - Create a `StorageCapacity` resource with `storage_capacity` as the minimum account storage and put it in account private storage.
    - Add a public capability to get `storage_capacity` from `StorageCapacity` resource.
    - Initialize the FLOW token vault and receiver.

Joining the calls to `FlowServiceAccount` into one transaction will also improve the performance of account creation.

#### Limit Storage Used

The amount of `storage_used` should be less then or equal to `storage_capacity`. There are two options on when to check that this constraint is not violated:

1. Deferred: Check after all register changes (and `storage_used` updates) from a transaction were made.
1. Immediate: Check on every `storage_used` update during the transaction.

The "immediate" approach has a slight added overhead as ech register set needs to also do a comparison. The "deferred" approach has the added benefit of allowing accounts to temporarily go over their storage capacity during the transaction then cleaning up before the end of the transaction. (e.g. assuming 100kB is the storage capacity an account could go to 120kB while doing a computation, as long as the account reduces its size to under 100kB before the transaction ends).

The "deferred" approach was chosen, due to it more accurately representing the amount of data that will actually get stored to the execution state.

The `StorageLimiter` transaction processor implementation will be used to prevent `storage_used` going over the `storage_capacity`. The `StorageLimiter` will use the transaction pipeline as follows:

1. Create new view.
2. Run transaction.
    1. Run transaction processors (a.k.a.: transaction pipeline):
        1. TransactionSignatureVerifier,
        2. TransactionSequenceNumberChecker,
        3. TransactionFeeDeductor,
        4. TransactionInvocator, <- actual transaction execution
        5. *There are no further register (view) writes (or deletes) in the transaction pipeline after this point. All storage size changes already happened.*
        6. **StorageLimiter** <- for all participating addresses that have touched registers, check if address' `storage_used` is over the address' `storage_capacity`. If it is, raise an error.
3. Error handling <- catch the storage over limit and revert transaction/
4. "Merge" view.

The internal behaviour of the `StorageLimiter` can be expressed with the following pseudo-code:

```py
accounts_changed = get_accounts_with_writes_or_deletes(in_current_view)
for account in accounts_changed:
    new_size = get_storage_used(account, in_current_view)
    if new_size > get_account_storage_capacity(account)
        raise Exception()  # descriptive error
```

#### Cadence Interface Changes

To expose the `storage_capacity` and `storage_used` values to Cadence programs, the following methods need to be added to Cadence's runtime interface:

```go
// GetStorageUsed gets storage used in bytes by the account at the moment of the function call.
GetStorageUsed(address Address) (value uint64, err error)
// GetStorageCapacity gets storage capacity in bytes on the account.
GetStorageCapacity(address Address) (value uint64, err error)
```

In the Cadence semantic analysis pass and in the interpreter, two new read-only fields are added to accounts (the types `AuthAccount` and `PublicAccount`):
  - `let storageCapacity: UInt64`: the maximum amount of storage that the account may store.
  - `let storageUsed: UInt64`: the currently used amount of storage. In this context "currently" refers to the state of the account as it is at that line of the programs execution.


### Storage Fees Smart Contract


Each account will have a public `StorageCapacity.depositStorage` capability and a private `StorageCapacity` resource on predetermined paths.

The `StorageCapacity` resource will be defined on the `StorageFees` smart contract:
- has a `storageCapacity` field which defines how much storage can the account holding this resource use.
- has a `FlowVault` with Flow that was used for paying for this storage. This vault cannot be accessed by anyone except the `StorageFees` smart contract. The exception is the balance of this flow vault which should be readable by anyone.
- has an array of pairs of UInt64, UFix64 `purchases` used for keeping track of how much storage was purchased for how much flow. 
- has a `depositStorage` function that can only be used by the `StorageFees` smart contract (is this possible?):
    - it should increment the `storageCapacity` field deposit into the `FlowVault` and append to `purchases`

The `StorageFees` smart contract will also have the following responsibilities:

- Definitions:
    - `minimumStorageUnit`: define the minimum unit (chunk) size of storage in bytes. Storage can only be bought (or refunded) in a multiple of the minimum storage unit.
    - `minimumAccountStorage`: defines the minimum amount of `storage_capacity` an address can have and also the amount every new account has. `minimumAccountStorage` is a multiple of `minimumStorageUnit`.
    - `flowPerByte`: defines the cost of 1 byte of storage in FLOW tokens.
    - `flowPerAccountCreation`: define the cost of purchasing the initial minimum storage in FLOW tokens.
    - `refundingEnabled`: defines if accounts can refund storage
- `minimumAccountStorage`, `refundingEnabled`, `flowPerByte` and `flowPerAccountCreation` should be able to change in the future.
- Functions:
    - `setupStorageForAccount(paymentVault, authAccount)`:
        - The paymentVault should have funds for the minimum storage needed by the address.
        - Check if `StorageCapacity` is already present on the account (if so panic; account already bought some storage).
        - Create the `StorageCapacity` resource with `minimumAccountStorage` and put the `paymentVault` into it.
        - Put this `StorageCapacity` resource onto the account with a corresponding receiver.
    - `purchaseStorageForAddress(paymentVault, address, amount)`:
        - The payer pays for extra storage needed by the address.
        - Check if `StorageCapacity` is present on the address (if not so panic; account needs to buy initial storage first).
        - Check if amount is divisible by `minimumStorageUnit` (if not panic).
        - Check if correct amount of funds were provided by `paymentVault`. Should be `flowPerByte` x amount
        - Using `StorageCapacity.depositStorage` on the destination account, increment its `StorageCapacity.storageCapacity` and deposit `paymentVault` into `StorageCapacity.FlowVault`.
    - `refundStorage(authAccount, amount)`:
        - The account refunds its own purchased storage to its own vault 
        - Check if `StorageCapacity` is present on the address (if not so panic; account needs to buy initial storage first)
        - Check if `amount` is divisible by `minimumStorageUnit` (if not panic)
        - Check if this would put account below `minimumAccountStorage` (if so panic)
        - If account refunds too much storage capacity (it is using more than it has left) the transaction will fail anyway, so no need to check for that.
        - Using `StorageCapacity.purchases` to determine how much Flow to return, withdraw from `StorageCapacity.FlowVault` edit `StorageCapacity.purchases` and reduce `StorageCapacity.storageCapacity`
        - return the withdrawn vault


The `StorageFees` smart contract needs to cover four cases:

1. When creating a new account, `StorageFees.setupStorageForAccount` will be called wit the `AuthAccount` of the account to setup:
    - A `StorageCapacity` resource needs to be added int accounts private storage with the `minimumAccountStorage` and the `FlowVault` used as payment.
    - A Capability to get the value of `storage_capacity` needs to be added to accounts public storage
2. When purchasing more storage for an account:
    - Accounts `StorageCapacity.storage_capacity` needs to increase
    - Accounts `StorageCapacity.FlowVault` needs to increase
    - Accounts `StorageCapacity.purchases` need to be appended to (for the purposes of correctly refunding storage)
3. When refunding storage.
    - Accounts `StorageCapacity.purchases` should be updated
    - A `FlowVault` should be returned
4. For convenience of use it should be possible to add a snippet of code to any transactions that does the following:
    - checks if a certain account is over storage capacity
    - if it is buy the minimum amount of storage (divisible by `minimumStorageUnit`) that puts the accounts storage used under capacity.


### Register Migration

All existing accounts will be given 1 FLOW token worth of storage (1MB). Only very few accounts currently use more than 1MB of storage. Those that do user more than 1MB will be checked individually and will be given `(n + 1)`MB of storage where `n` is the smallest integer so that the account is using less then `n`MB of storage.

Because the process of creating a register migration is not documented yet a few assumptions need to be made:
- Migration will be a function with the signature of: `(key, value) -> [(key, value)]`.
- The function must be stateless.
- It has access to other registers.

With this in mind the migration pseudo-code can be written as:

```py
def migrate(key, value) -> (keyType, valueType)[]:
    owner = get_owner_from_key(key)
    if register_exists(owner=owner, controller="", key="storage_capacity"):
        # we already migrated this account
        return [(key, value)]
    storage_used = 0
    # get all register entries for this owner and iterate over them
    for register in registers_for_owner(owner)
        storage_used += register_size(register)

    storage_used += 16 # because we are adding two more uint64 registers

    storage_capacity = calculate_capacity_needed(storage_used)

    storage_capacity_resource_register = create_storage_capacity_resource_register(storage_capacity)
    storage_capacity_resource_public_capability_register = create_storage_capacity_resource_public_capability_register()

    return [
        (key, value),
        storage_capacity_resource_register,
        storage_capacity_resource_public_capability_register,
        ((owner, "", "storage_used"), storage_used)
    ]
```

#### Future Migrations

During any future register migration if any account registers are touched storage used must be updated as well. To ensure that this is done, touching any account related registers should go through the logic in `accounts.go`.

### Performance Implications

The changes done will affect all transactions by adding a small overhead:
    - Calculation of `storage_used`.
    - Checking `storage_used` is not over `storage_capacity`.

This overhead can be measured by comparing the execution time of integration tests (that already exist).

Time complexity of transactions that are creating new accounts will actually be reduced because of batching two calls to into one (see [Create Account Changes](#create-account-changes)).

Memory usage should not significantly change since only one field one resource and one capability are being added to each account. The current minimum account size is about one order of magnitude larger (flow token receiver, flow token vault, private keys, ...), than the registers added.

### Dependencies

* Dependent projects: 
    * `flow-core-contracts` (adding **StorageFeesContract**, changing **FeesContract**),
    * `cadence` (access to new fields),
    * `flow-go` (execution change),
    * `emulator` (emulate storage size limiting),
    * SDK-s (example transactions to check/increase storage).

### Engineering Impact

* Binary size / build time / test times should not noticeably increase.
* Most of the changes are spread out over cadence, flow-core-contracts and flow-go they can be covered by unit test separately.

### Tutorials and Examples

There are no protobuf API changes.

There are Cadence API changes that can be covered with the following transactions examples:

1. Tx: get storage used and capacity of an account.
2. Tx: purchase more storage for an account.
3. Tx: transaction failing because it is storing to much.

TODO: make the examples

### Documentation changes

The documentation changes should cover the following topics:

- Why storage fees?
- How much are storage fees and where is this written?
- Minimum storage on account?
- Smallest purchasable storage chunk?
- How to check storage used and capacity?
- How to purchase more storage?
- How to refund storage?
- What error will I get if im out of storage?
- Access to storage capacity and storage used values in Cadence programs.

### Compatibility

Execution nodes will not be backwards compatible, because the execution state will be updated to include storage used/capacity fields on all accounts.

### User Impact

Current accounts will be updated to include storage used/capacity fields, with the capacity being set to the minimum capacity. For the accounts that are over the limit we need to buy sufficient storage.

Current users will not be impacted until they they reach storage capacity.

## Questions and Discussion Topics


1. Could an account be deleted for the purpose of refunding its creation fee.
1. When calling `deductAccountCreationFee` on the **FlowServiceAccount** besides storage fees what constitutes the fees?: