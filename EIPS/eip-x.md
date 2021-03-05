---
eip: <to be assigned>
title: Universal Smart Contract Access Control
author: Jan Turk (jan.turk@infinum.com)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2021-03-01
---

## Simple Summary

Univesal Smart Contract Access Control (USCAC) provides a unified access control for any number of related, or unrelated, smart contracts. In stead of specifying access control in each smart contract, the USCAC interface is used to provide the mechanic for limiting the execution of functions to various tenants like owners, administrators and other tennants. It allows smart contracts to specify the number of tenant tiers it wants to use and specifies the management API.

## Abstract

Access control is currently specified in each smart contract that need it. Usuallt there are owners and amdinistrators defined. If a smart contract requires additional levels of tennants, that has to be added by the developer. Addinf access control to a smart contract increases its size and that is an issue in an ecosystem where each bit matters.
USCAC provides a unified interface to be securely used in stead of including the access control to each smart contract that needs it. It specifies the interface that includes adding, removing and viewing the tennants.

## Motivation

Each smart contract that isn't self owned requires some level of access control. This means that most of the smart contracts have to implement the same part of code that makes sure, only authorised users can call certain functions. Even if that part of the source code is copy-pasted, it is not impervious to errors. Additionally it takes away some cunk of smart contract file size which is limited and could be used by business logic.

Implementing USCAC allows the developer of a smart contract to focus on the mechanics of the smart conrtact and hand off the access control to USCAC. This way the access control is executed in a standardised and reliable way and alleviates a part of smart contract development.

## Specification

### Initial setup

- This hould be called in the `constructor()`, so that the USCAC gets the information about the number of access tiers to setup.
- Top level of access tiers is reserved for owners and there can only be one (so if you whish to have owners and administrators included, the number of tiers should be `2`)
- If the `initialOwner` value is set to `0x0`, the owner position will default to `msg.sender`

```js
function initialSetupOfUSCAC(uint256 numberOfTenantTiers, address initialOwner) public returns (bool)
```

### Managing access control

These functions allow authorised users to change the access permissions of the tenants.

#### Transferring ownership

Transfers the `owner` permissions to the `newOwner`.

`bool` value returned signals successful execution of the call.

**NOTES**

- Since there is only one owner it makes no sense to change access rights, they should only be transferrable
- Because another smart contract is called, the `msg.sender`should be passed as the `caller`value

```js
function transferOwnership(address caller, address newOwner) public returns (bool)
```

#### Managing access rights

Allows authorised tenant to modify per-level access rights for other tenants.

`level` signifies the tenant level. It is and `uint256` value, which can be applied by the developer of the smart contract.

`tenant` is the address of the tennant that we are modifying access rights for.

`accessPermitted` specifies whether we are granting or revoking access rights to the before mentioned tenant.

`MODIFIER` is the custom modifier applied for managing this level of the access rights.

`bool` value returned by the function signals successful execution of the call.

**NOTES**

- Modifiers should check if the `msg.sender`is allowed to call this function
- Assuming that higher tier of tenant can modify lower tier of tenants is wrong, because some use cases might just require different tenants that are not hierarchically lower or higher than another, they are just limited to calling different functions
- If the `tenant`allready has this `level` of access rights, the function should be reverted by USCAC, since there is no use in performing redundant operations

```js
function manageAccessRights(uint256 level, address tenant, bool accessPermitted) MODIFIER public returns (bool)
```

### Verifying access control

These view function are used to verify whether a tenant has a certain level of access rights or not.

#### Verifying owner

Checks whether the specified address is the owner or not.

`tenant` is the address that is checked by USCAC for whether it is an owner address or not.

`bool` is a feedback value that tells us if the `tenant` is the `owner` (`true`) or not (`false`)

```js
function isOwner(address tenant) public view returns (bool)
```

#### Verifying tenant access permissions

Checks whether the specified address has specified level's access permissions or not.

`level`specifies the tenant level we are chesking.

`tenant` specifies the address we are checkig the access rights for.

`bool` is a feedbas for whether the `tenant` has (`true`) or doesn't have (`false`) the access rights for the specified `level`.

```js
function hasLevelAccessPermission(uint256 level, address tenant) public view returns (bool)
```

### Transferring the access controll database

If a smart contract needs to be updated, but the access permissions should remain the same for all of the tenants, the migration mechanic should be used.

If any smart contract could just specify which address should receive it's access control configuration, this could be potentially catastrophic. This is why proper migration procedure has to be followed.

#### Initiate access control migration

The *original* smart contract should initiate the migration.

`newSmartContract` is the address of the smart contract intended to receive the migrated access control database.

`MODIFIER` restrihe bility to initia migration to the desired level or levels of tenant(s).

`bool` signalls that the mighration was initiated.

**NOTES**

- This should be done after the updated smart contract is deployed, since the *updated* smart contract's address is required
- Once the migration is complete, the access control database is owned and managed by the *updated* smart contract in order to protect from accidental or malicious modifications done from the *original* smart contract
- When access control database of a smart contract enters migration state, the modifications are halted until the process completes of is stopped.

``` js
function initiateAccessControlMigration(address newSmartContract) MODIFIER public returns (bool)
```

#### Accept access control migration

The *updated* smart contract should accept the migration.

`previousSmartContract` is the address of the smart contract from which we are accepting the access control database.

`bool` signal the successful migration of the access controsle and subsequent completion of the migration.

**NOTES**

- Once the migration is accepted, ALL of the access control configuration is available to the `updated` smart contract
- Upon completion of the migration, the access control database can once again be modified

```js
function acceptAccessControlMigration(address previousSmartContract) public returns (bool)
```

#### Cancel access control migration

The *original* smart contract should be able to cancel the access control database migration if that is required for some reason.

`MODIFIER` should restrict calling of the function to only desired level or levels of tenant(s).

`bool` signifies the successful termination of the migratioon process.

**NOTES**

- Once the access control database migration is terminated, the access control database can once again be modified by the *original* smart contract
- If the access control database migration is terminated *updated* smart contract can no longer accept the migration
- After the termination of access control database, the migration can be reinitiated

```js
function cancelAccessControlMigration() MODIFIER public returns (bool)
```

### Events

#### OwnershipTransfer

MUST trugger when ownership is transferred.

When initially setting the owner, the ownership is transferred from address `0x0`to the initial owner.

```js
event OwnershipTransfer(address indexed smartContract, address indexed previousOwner, address indexed newOwner)
```

#### TenantPermissionChange

MUST trigger when tenant's permissions are changed.

```js
event TenantPermissionChange(address indexed smartContract, uint256 tenantLevel, address indexed tenant, bool permission)
```

#### AccessControlMigration

MUST trigger when access control migration changes state.

The `state` should be an enumerable value that translates to state:

- `initiated` when access control migration is initiated
- `accepted` when access control migration is accepted
- `cancelled` when access control migration is cancelled

```js
event AccessControlMigration(address originalSmartContract, address newSmartContract, uint state)
```

## Test Cases
The example smart contrat using USCAC is available at the [Infinum repository](https://github.com/infinum/ethereum-universal-access-control).

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
