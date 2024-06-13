---
rpip: 59
title: Deposit Mechanics
description: Describes the mechanics of Node Operator deposits and validator creation, including standard and express queues.
author: Valdorff (@Valdorff)
discussions-to: TBD
status: Draft
type: Protocol
category: Core
created: 2024-03-08
requires: 42, 43, 44
tags: tokenomics-2024, tokenomics-content
---

## Abstract

Defines the queue structures that govern validator deposits and the overall deposit process within Rocket Pool. Both a standard and express queue structure are present. Access to the express queue is managed via tickets. A limited number of tickets are provided to all new nodes, and an amount of tickets are provided to existing nodes at the time of the upgrade based on their amount of bonded ETH. The express queue is designed to progress faster than the standard queue.

This proposal also changes the deposit mechanics: In case of a queue, the initial stake transaction happens only once ETH is assigned. This makes it possible to exit from the queue and receive ETH credit up until the validator is dequeued.

## Specification

### Deposit queue specification
ETH from the deposit pool SHALL be matched with validator deposits from queues as follows:
- There SHALL be a `standard_queue`
  - When adding a validator, users MAY place their deposit on the `standard_queue`  
- There SHALL be an `express_queue`
  - When adding a validator, users MAY place their deposit on the `express_queue` by spending one `express_queue_ticket`
- When matching ETH from the deposit pool to queued deposits:
  - First, ETH SHALL be matched to the oldest deposit in the `express_queue`; this is repeated `express_queue_rate` times
  - Next, ETH SHALL be matched to the oldest deposit in the `standard_queue`
  - This sequence SHALL be repeated indefinitely until both queues are empty
    - If one queue is empty, those matches SHALL be skipped
- Each node SHALL be provided `express_queue_tickets` equal to `express_queue_tickets_base_provision` (this includes newly created nodes)
- Each node SHALL be provided additional `express_queue_tickets` equal to `(bonded ETH in legacy minipools)/4` (this will always be zero for newly created nodes)
- The initial settings SHALL be:
  - `express_queue_rate`: 2
  - `express_queue_tickets_base_provision`: 2

### Deposit mechanics specification
- A node operator MUST take 2 actions to start a validator: `deposit` and `stake`

#### `deposit` Transaction
- `deposit` SHALL place the Node Operator ETH in the deposit pool (where it can be used in validators as needed) and place the validator in a queue as described [above](#deposit-queue-specification)
- The following values for the validator SHALL be stored on chain:
    - the public key of the validator
    - the BLS signature over the public key, the withdrawal credentials (the megapool address of the node operator), and the amount (1 ETH) as required by the deposit contract
    - the deposit message root as required by the deposit contract
- The transaction SHALL validate the provided values and revert if they do not match
- `deposit` SHALL assign deposits as described below 

#### Assigning ETH from the Deposit Pool
- As ETH enters the deposit pool, it SHALL be assigned to valdiators from the queue by sending 32 ETH to the associated megapool contract
- The assignment SHALL execute the `Prestake` transaction, staking 1 ETH to the beacon chain using the values provided in the step above

#### `stake` Transaction
- `stake` SHALL revert unless at least `scrub_period` time has passed since ETH was assigned to the validator, to allow for validating the prestake
- If the beacon chain stake is invalid, the validator SHALL be scrubbed 
- `stake` SHALL stake the remaining 31 ETH to the beacon chain to make a complete validator
- If `stake` is not called within `time_before_dissolve` after the ETH was assigned, the validator SHALL be dissolved, returning the user balance to the deposit pool

#### Exiting Queue
- Until ETH is assigned to a validator, it SHALL be possible to exit the queue and receive ETH `credit` for it

#### Initial Settings
- The initial settings SHALL be:
  - `scrub_period`: 12 hours
  - `time_before_dissolve`: 2 weeks

## Rationale

- The express queue is meant to favor (a) small NOs and (b) existing NOs. The end goal in both cases is to support multiple values enshrined in [RPIP-23](RPIP-23.md) (the pDAO charter): decentralization, protocol safety, and the health of the Ethereum network.
  - The `express_queue_tickets_base_provision` is enough to get started, and currently matches the length of `base_bond_array`
  - The tickets from `(bonded ETH in legacy minipools)/4` are enough to fully migrate to 4-ETH deposits during Saturn 1 using the express queue OR to partly migrate to 1.5-ETH deposits after Saturn 2
  - It's worth emphasizing that the tickets stick around -- ie, a node operator joining during a time when we don't have an NO queue (ie, when RP has an immediate need for NO supply) gets to keep their express queue benefit for a later time if they wish
- Validators with `base_bond` deposits are prioritized to promote decentralization; new or smaller Node Operators can get up to `base_bond_array.length` validators launched ahead of larger Node Operators adding `reduced_bond` validators.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).