# address-appearance-index-specs

A specification of an [index](#addressappearanceindex) that maps addresses to the transactions that they appear in.
The index is divisible and distributable.

> Status: draft

Table of contents
<!-- START doctoc -->
- [address-appearance-index-specs](#address-appearance-index-specs)
  - [Introduction](#introduction)
  - [Scope](#scope)
  - [Notation](#notation)
  - [References](#references)
    - [Ethereum](#ethereum)
    - [Ethereum consensus specification](#ethereum-consensus-specification)
    - [EIP-4444](#eip-4444)
    - [Portal Network](#portal-network)
    - [trueblocks-core](#trueblocks-core)
    - [Unchained Index](#unchained-index)
    - [SSZ Spec](#ssz-spec)
    - [Snappy](#snappy)
  - [Constants](#constants)
    - [Design parameters](#design-parameters)
    - [List lengths](#list-lengths)
    - [Derived](#derived)
  - [Containers](#containers)
    - [Core (required to use the index)](#core-required-to-use-the-index)
      - [`AppearanceTx`](#appearancetx)
      - [`AddressAppearances`](#addressappearances)
      - [`BlockRange`](#blockrange)
      - [`AddressIndexSubset`](#addressindexsubset)
    - [Informative (not required to use the index)](#informative-not-required-to-use-the-index)
      - [`AddressDivision`](#addressdivision)
      - [`AddressAppearanceIndex`](#addressappearanceindex)
  - [Definitions and aliases](#definitions-and-aliases)
  - [Overview of the address-appearance-index](#overview-of-the-address-appearance-index)
    - [Service provided by the index](#service-provided-by-the-index)
    - [Service assumed from the data layer](#service-assumed-from-the-data-layer)
    - [Service assumed from the network layer](#service-assumed-from-the-network-layer)
    - [Functions of the index](#functions-of-the-index)
  - [Index architecture](#index-architecture)
    - [Data](#data)
    - [Manifest](#manifest)
  - [Procedures](#procedures)
    - [Maintenance: Creation](#maintenance-creation)
    - [Maintenance: Extension](#maintenance-extension)
    - [Maintenance: Correctness audit](#maintenance-correctness-audit)
    - [User: Consumption](#user-consumption)
    - [User: Completeness audit](#user-completeness-audit)
  - [Design principles](#design-principles)
<!-- END doctoc -->

## Introduction

TODO

The index structure definition begins at [`AddressAppearanceIndex`](#addressappearanceindex).

## Scope

- Index architecture:
    - Data
        - Address divisions
        - address index subsets
        - Serialization
        - Compression
    - Manifest
        - Specification version
        - Unchained Index version
        - Latest block height
- Procedures:
    - Maintenance: Creation
    - Maintenance: Extension
    - Maintenance: Correctness audit
    - User: Consumption
    - User: Completeness audit

## Notation

Code snippets appearing in `this style` are to be interpreted as Python 3 psuedocode. The
style of the document is intended to be readable by those familiar with the
Ethereum [consensus](#ethereum-consensus-specification),
[ssz](#ssz-spec) and [portal network](#portal-network) specifications. Part of this
rationale is that SSZ serialization is used in this index to benefit from ecosystem tooling.
No doctesting is applied and functions are not likely to be executable.

## References

### Ethereum

[A secure decentralised transaction ledger](https://ethereum.github.io/yellowpaper/paper.pdf)

### Ethereum consensus specification

[Ethereum proof-of-stake specifications](https://github.com/ethereum/consensus-specs)

### EIP-4444

[Bound Historical Data in Execution Clients](https://eips.ethereum.org/EIPS/eip-4444)

### Portal Network

[Lightweight protocol access by resource constrained devices](https://github.com/ethereum/portal-network-specs/blob/master/README.md)

### trueblocks-core

[TrueBlocks creates an index that lets you access the entire Ethereum chain directly from your local machine.](https://github.com/TrueBlocks/trueblocks-core)

### Unchained Index

[The Unchained Index is a naturally-sharded, easily-shared, reproducible, and minimally-sized
immutable index for EVM-based blockchains.](https://trueblocks.io/papers/2022/file-format-spec-v0.40.0-beta.pdf)

### SSZ Spec

[Simple Serialize (SSZ) is a standard for the encoding and merkleization of structured data](https://github.com/ethereum/consensus-specs/blob/dev/ssz/simple-serialize.md)

### Snappy

[Snappy, a fast compressor/decompressor.](https://github.com/google/snappy)

## Constants

### Design parameters

| Name | Value | Description |
| - | - | - |
| `ADDRESS_CHARS_SIMILARITY_DEPTH` | `uint32(2)` | A parameter for the number of identical characters that two similar addresses share in hexadecimal representation. If the value is `2`, address`0xa350...9c45` will be evaluated for similarity using the characters `0xa3`. |
| `BLOCK_RANGE_WIDTH` | `uint32(10*5)` (=100_000) | Number of blocks in a address index subset. Index maintenance operates at this cadence (~2 weeks) |

### List lengths

Helper values for SSZ operations.

| Name | Value | Description |
| - | - | - |
| `MAX_ADDRESSES_PER_BLOCK_RANGE` | `uint32(2*30)` (=1_073_741_824) | Chosen as practical ceiling for number of addresses plausible in `BLOCK_RANGE_WIDTH` blocks. Not rigorous. |
| `MAX_TXS_PER_BLOCK_RANGE` | `uint32(2*30)` (=1_073_741_824) | Chosen as practical ceiling for number of transactions plausible in `BLOCK_RANGE_WIDTH` blocks. Not rigorous. |
| `MAX_RANGES` | `uint32(2**16)` (=65_536) | Chosen as practical ceiling for number of ranges plausible in `BLOCK_RANGE_WIDTH`. |

### Derived

Constants derived from [design parameters](#design-parameters.

| Name | Value | Description |
| - | - | - |
| `NUM_DIVISIONS` | `uint32(16**ADDRESS_CHARS_SIMILARITY_DEPTH)` | Number of unique hex character combinations possible. Represents how manny pieces the index can be split. |

## Containers

### Core (required to use the index)

The `AddressIndexSubset` is the only data that is subject to SSZ serialization/deserialization.
The remaining containers are components of that container.
Use of the index requires implementation of these containers.

#### `AppearanceTx`

A single transaction identifier (block number and index within block).
Transactions are an AppearanceTx for an address when, at some point during execution, the
address appears in the EVM trace.

```python
class AppearanceTx(Container):
    block: uint32
    index: uint32
```

#### `AddressAppearances`

This unit holds transactions for a single address. All transactions are from blocks
within the range defined in the parent AddressIndexSubset.

```python
class AddressAppearances(Container):
    address: Bytes20
    appearances: List(AppearanceTx, MAX_TXS_PER_BLOCK_RANGE)
```

#### `BlockRange`

An inclusive range of block numbers (heights) used to define which transactions are
suitable for inclusion in a given container. The `old` block is used as
the identifier for the parent `AddressIndeSubset`. The `new` block is defined
as `old` plus `BLOCK_RANGE_WIDTH`.

```python
class BlockRange(Container):
    old: uint32
    new: uint32
```

#### `AddressIndexSubset`

Many of these units form a functional unit for a user.
It is the unit that is subject to SSZ serialization then compression
or decompression then deserialization. Identified by `range.old`, with block `0` being the first
and block `BLOCK_RANGE_WIDTH` being the second.

Contains a collection representing transaction that meet both the address and block
range requirements.

```python
class AddressIndexSubset(Container):
    address_prefix: ByteVector[ADDRESS_CHARS_SIMILARITY_DEPTH]
    range: BlockRange
    appearances: List(AddressAppearances, MAX_ADDRESSES_PER_BLOCK_RANGE)
```

### Informative (not required to use the index)

The following containers illustrate the structure and operation of the index and are referred
to in subsequent sections.
No SSZ serialization/deserialization is performed on these containers.
Use of the index requires implementation of these containers.

#### `AddressDivision`

This is the functional unit that a user acquires from a peer.
A collection of similar addresses. Named using a division identifier (e.g., `0x5a`). See
[`AddressDivision`](#addressdivision). Contains serialized and
compressed units, one per `BLOCK_RANGE` blocks. Total members is calculated by latest block height divided by `BLOCK_RANGE_WIDTH`.

```python
class AddressDivision(Container):
    identifier: ByteVector[ADDRESS_CHARS_SIMILARITY_DEPTH]
    members: List(AddressIndexSubset, MAX_RANGES)
```

#### `AddressAppearanceIndex`

The entire address-appearance-index, divided into [`AddressDivision`](#addressdivision)'s that
contain block [`AddressIndexSubset`](#addressindexsubset)'s,
ultimately containing transaction identifiers as [`AppearanceTx`](#appearancetx)'s.
The index is divided into `NUM_DIVISIONS` functional units.

The index is a derivative of the Unchained Index, and contains the same data, reorganised
for a different purposse.

```python
class AddressAppearanceIndex(Container):
    divisions: List(AddressDivision, NUM_DIVISIONS)
```
## Definitions and aliases

The following terms may be used in subsequent descriptions.

*Address*: A 20 byte identifier used for wallets and accounts in Ethereum. Usually represented
in hexadecimal representation.

*Division*: See [`AddressDivision`](#addressdivision).

*Division identifier*: The hexadecimal characters that is common to all addresses in an [`AddressDivision`](#addressdivision). E.g., `0xa3`.

*Subset*: See [`AddressIndexSubset`](#addressindexsubset).

*Subset identifier*: The block height of the oldest block in a
[`AddressIndexSubset`](#addressindexsubset).
E.g., subset `13_000_000` contains blocks `13_000_000` to `13_099_999` inclusive.

*Similar address*: Two addresses are similar if they share their first characters in hexadecimal representation to a depth of `ADDRESS_CHARS_SIMILARITY_DEPTH`.

*The index*: The address-appearance-index. See [`AddressAppearanceIndex`](#addressappearanceindex).

## Overview of the address-appearance-index

### Service provided by the index
### Service assumed from the data layer
### Service assumed from the network layer
### Functions of the index

## Index architecture

### Data

- Address group
- Block range division
- Serialization
- Compression

### Manifest


## Procedures

### Maintenance: Creation

- Index
- Manifest

### Maintenance: Extension

- Index
- Manifest

### Maintenance: Correctness audit
### User: Consumption
### User: Completeness audit

## Design principles

