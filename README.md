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
      - [String naming conventions](#string-naming-conventions)
      - [Estimated file count](#estimated-file-count)
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

[Consensus spec: SSZ-snappy encoding strategy](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/p2p-interface.md#ssz-snappy-encoding-strategy)

[Google: Snappy, a fast compressor/decompressor.](https://github.com/google/snappy)

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
| `NUM_DIVISIONS` | `uint32(16**ADDRESS_CHARS_SIMILARITY_DEPTH)` (=256) | Number of unique hex character combinations possible. Represents how manny pieces the index can be split. |

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
the identifier for the parent `AddressIndexSubset`. The `new` block is defined
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

- Address group.
- Block range division.
- Serialization.
- Compression.

#### String naming conventions

In the case where files are being stored, the following naming conventions are recommended.
This makes direct file transfer between peers more robust in some situations. The "{}" braces
are excluded from the file name. Patterns are provided using regular expressions.

Directory representing [the index](#addressappearanceindex), where:
- "NETWORK" is the network name. E.g., ["mainnet"](https://eips.ethereum.org/EIPS/eip-2228).
```sh
Format: address_appearance_index_{NETWORK}

Regular expression: "/^address_appearance_index_+[a-z]$/"

Example directory: ./address_appearance_index_mainnet/
```

Directory representing a [division](#addressdivision), where:
- "DIVISION_DESCRIPTOR" has `ADDRESS_CHARS_SIMILARITY_DEPTH` characters.
```sh
Format: division_0x{DIVISION_DESCRIPTOR}

Regular expression: "/^division_0x[0-9]{ADDRESS_CHARS_SIMILARITY_DEPTH}$/"

Example directory: ./division_Ox4e/
```

File representing a [subset](#addressindexsubset), where:
- "DIVISION_DESCRIPTOR" has `ADDRESS_CHARS_SIMILARITY_DEPTH` characters.
- "SUBSET_DESCRIPTOR" is padded to 9 decimal characters and divided into groups of 3 characters.
Block `14_500_000` is shown in the example.
- "encoding" is one of two choices:
  - "ssz" for data encoded with [SSZ](#ssz-spec) serialization.
  - "ssz_snappy" for data encoded with [SSZ](#ssz-spec) serialization followed by encoding with
  [snappy](#snappy).
```sh
Format: division_0x{DIVISION_DESCRIPTOR}_subset_{SUBSET_DESCRIPTOR}.{encoding}

Regular Expression: "/^division_0x[0-9]{ADDRESS_CHARS_SIMILARITY_DEPTH}_subset(_[0-9]{3}){3}.ssz(_snappy)?$/"

Example file: ./division_0x4e_subset_014_500_000.ssz
Example file: ./division_0x4e_subset_014_500_000.ssz_snappy
```

Example of a suggested complete index folder structure:
```sh
- ./address_appearance_index_mainnet/
  - ...
  - /division_Ox4e/
    - ...
    - /division_0x4e_subset_014_500_000.ssz_snappy
    - /division_0x4e_subset_014_600_000.ssz_snappy
    - ...
  - /division_Ox4f/
    - ...
    - /division_0x4e_subset_014_500_000.ssz_snappy
    - ...
  - ...
```
#### Estimated file count

The estimated file count for a complete index on a network at a certain block height can
be calculated as follows:
```python
def estimate_file_count(block_height):
  division_directories = NUM_DIVISIONS
  subset_files_per_directory = block_height // BLOCK_RANGE_WIDTH
  file_count = division_directories * subset_files_per_directory
  return file_count

print(estimate_file_count(15_400_000))
> 39424
```
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

