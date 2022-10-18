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
    - [Fixed-size type parameters](#fixed-size-type-parameters)
    - [Variable-size type parameters](#variable-size-type-parameters)
    - [Derived](#derived)
  - [Containers](#containers)
    - [Data (required to use the index data)](#data-required-to-use-the-index-data)
      - [`AppearanceTx`](#appearancetx)
      - [`AddressAppearances`](#addressappearances)
      - [`BlockRange`](#blockrange)
      - [`AddressIndexSubset`](#addressindexsubset)
    - [Metadata (required to use the index manifest)](#metadata-required-to-use-the-index-manifest)
      - [`SubsetIdentifier`](#subsetidentifier)
      - [`ManifestSubset`](#manifestsubset)
    - [`DivisionIdentifier`](#divisionidentifier)
      - [`ManifestDivision`](#manifestdivision)
      - [`NetworkIdentifier`](#networkidentifier)
      - [`IndexManifest`](#indexmanifest)
    - [Informative (not required to use the index)](#informative-not-required-to-use-the-index)
      - [`AddressDivision`](#addressdivision)
      - [`AddressAppearanceIndex`](#addressappearanceindex)
  - [Definitions and aliases](#definitions-and-aliases)
  - [Overview of the address-appearance-index](#overview-of-the-address-appearance-index)
    - [Service provided by the index](#service-provided-by-the-index)
      - [Agent type: Maintainer](#agent-type-maintainer)
      - [Agent type: User](#agent-type-user)
    - [Service assumed from the data layer](#service-assumed-from-the-data-layer)
    - [Service assumed from the network layer](#service-assumed-from-the-network-layer)
    - [Functions of the index](#functions-of-the-index)
  - [Index architecture](#index-architecture)
    - [Data architecture description](#data-architecture-description)
      - [String naming conventions](#string-naming-conventions)
      - [Estimated file count](#estimated-file-count)
    - [Manifest architecture description](#manifest-architecture-description)
  - [Procedures](#procedures)
    - [Maintenance: Creation](#maintenance-creation)
    - [Maintenance: Extension](#maintenance-extension)
    - [Maintenance: Correctness audit](#maintenance-correctness-audit)
    - [User: Consumption](#user-consumption)
    - [User: Completeness audit](#user-completeness-audit)
  - [Design principles](#design-principles)
  - [Versioning](#versioning)
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

### Fixed-size type parameters


| Name | Value | Description |
| - | - | - |
| `BYTES_PER_ADDRESS` | `uint32(20)` | Number of bytes for an address (E.g., `20` bytes on mainnet). May be different on some networks. |
| `NETWORK_IDENTIFIER_BYTES` | `uint32(2**5)` (=32) | Maximum number of bytes available to represent the ASCII-encoded network identifier. E.g. The string ["mainnet"](https://eips.ethereum.org/EIPS/eip-2228) is 7 bytes. |

### Variable-size type parameters

Helper values for SSZ operations. SSZ variable-size elements require a maximum length field.

| Name | Value | Description |
| - | - | - |
| `MAX_ADDRESSES_PER_BLOCK_RANGE` | `uint32(2*30)` (=1_073_741_824) | Chosen as practical ceiling for number of addresses plausible in `BLOCK_RANGE_WIDTH` blocks. Not rigorous. |
| `MAX_TXS_PER_BLOCK_RANGE` | `uint32(2*30)` (=1_073_741_824) | Chosen as practical ceiling for number of transactions plausible in `BLOCK_RANGE_WIDTH` blocks. Not rigorous. |
| `MAX_RANGES` | `uint32(2**16)` (=65_536) | Chosen as practical ceiling for number of ranges plausible in `BLOCK_RANGE_WIDTH`. |

### Derived

Constants derived from [design parameters](#design-parameters).

| Name | Value | Description |
| - | - | - |
| `NUM_DIVISIONS` | `uint32(16**ADDRESS_CHARS_SIMILARITY_DEPTH)` (=256) | Number of unique hex character combinations possible. Represents how many pieces the index can be split. |

## Containers

The ordering of the fields for each container is defined in the code snippets below and is
important for serialization/deserialization.

### Data (required to use the index data)

The `AddressIndexSubset` is the main container in this section and is
subject to SSZ serialization/deserialization to create/access the index data.

The remaining containers are components of that container.
Use of the index requires implementation of these containers.

#### `AppearanceTx`

A single transaction identifier (block number and index within block).
Transactions are an AppearanceTx for an address when, at some point during execution, the
address appears in the transaction execution trace.

```python
class AppearanceTx(Container):
    block: uint32
    index: uint32
```

#### `AddressAppearances`

This unit holds transactions for a single address. All transactions are from blocks
within the range defined in the parent AddressIndexSubset.

The elements in the appearances list are sorted lexicographically first by their block
(`AppearanceTx.block`) field, and then by their index (`AppearanceTx.index`) field
(where the address appears twice in a block).

```python
class AddressAppearances(Container):
    address: Vector[byte, BYTES_PER_ADDRESS]
    appearances: List[AppearanceTx, MAX_TXS_PER_BLOCK_RANGE]
```

#### `BlockRange`

An inclusive range of block numbers (heights) used to define which transactions are
suitable for inclusion in a given container. The `old` block is used as
the identifier for the parent `AddressIndexSubset`. The `BlockRange` represents
`BLOCK_RANGE_WIDTH` members and as such, the `new` block is defined
as ` new = old + BLOCK_RANGE_WIDTH - 1`.

```python
class BlockRange(Container):
    old: uint32
    new: uint32
```

#### `AddressIndexSubset`

Many of these units form a functional unit for a user.
It is the unit that is subject to SSZ serialization then compression
or decompression then deserialization.

Contains a collection representing transaction that meet both the address and block
range requirements.

The elements in the addresses list are sorted lexicographically by their address
(`AddressAppearances.address`) field

```python
class AddressIndexSubset(Container):
    address_prefix: ByteVector[ADDRESS_CHARS_SIMILARITY_DEPTH]
    range: BlockRange
    addresses: List[AddressAppearances, MAX_ADDRESSES_PER_BLOCK_RANGE]
```

### Metadata (required to use the index manifest)

The `IndexManifest` is the main container in this section and is
subject to SSZ serialization/deserialization to create/access the index manifest.

The remaining containers are components of that container.
Use of the index manifest requires implementation of these containers.

#### `SubsetIdentifier`

Each [subset](#addressindexsubset) has an identifer, defined as `AddressIndexSubset.range.old`
(the oldest block that the subset data may contain).
For example, the first subset has the identifier block `0` (it contains data
for block `0`) and the second subset has identifier `BLOCK_RANGE_WIDTH`.

The identifier is used in the [manifest](#manifest) and in the
[naming conventions](#string-naming-conventions) for index files.

```python
class SubsetIdentifier(Conainer):
    identifier: uint32
```

#### `ManifestSubset`

Used to create the index [manifest](#manifest). This unit represents a store for the hash of a single
[`AddressIndexSubset`](#addressindexsubset). The hash is the tree root hash,
as defined in the [ssz spec](#ssz-spec).

A single subset may be referred to by it's [identifier](#subsetidentifier).

```python
class ManifestSubset(Container):
    identifier: SubsetIdentifier
    tree_root_hash: uint256
```

### `DivisionIdentifier`

An identifier defines the member addresses for a [division](#addressdivision). Addresses
all start with the same bytes, E.g., `5a`.

The identifier is also used in [naming conventions](#string-naming-conventions) for
index directories.

```python
class DivisionIdentifier(Container):
    identifier: ByteVector[ADDRESS_CHARS_SIMILARITY_DEPTH]
```

#### `ManifestDivision`

Used to create the index [manifest](#manifest).
This unit stores the [subset metadata](#manifestsubset) for a single
[`AddressDivision`](#addressdivision) with a given [identifier](#divisionidentifier).

The elements in the `subset_metadata` list are sorted lexicographically by their
identifier (`ManifestSubset.identifier`) field.

```python
class ManifestDivision(Container):
    identifier: DivisionIdentifier
    subset_metadata: List[ManifestSubset, MAX_RANGES]
```

#### `NetworkIdentifier`

Used to identify the network that the index data belongs to. The network
is represented as an ASCII-encoded byte vector with maximum length `NETWORK_IDENTIFIER_BYTES`.

```python
class NetworkIdentifier(Container):
    name: ByteVector[NETWORK_IDENTIFIER_BYTES]
```

#### `IndexManifest`

This is a container that represents a file containing metadata about the index.
It is a record of the components of a specific instantiation of the index.
The manifest keeps the hash of all [`AddressIndexSubset`](#addressindexsubset)s.
If the index is updated to include newer data as the chain progresses,
a new manifest is generated to reflect the contents of the index.

The manifest includes:
- The [specification version](#versioning) declared as three separate `spec_version_major`,
`spec_version_minor` and `spec_version_patch` integers.
- The [network](#networkidentifier)
- The [latest_subset_identifier](#subsetidentifier) of the highest (latest) completed [subset](#addressindexsubset).
- The [division_metadata](#manifestdivision) which each contain the hashes of the subset.
  - The elements in the division metadata vector are sorted lexicographically by their
identifier (`ManifestDivision.identifier`) field.

```python
class IndexManifest(Containr):
    spec_version_major: uint32
    spec_version_minor: uint32
    spec_version_patch: uint32
    network: NetworkIdentifier
    latest_subset_identifier: SubsetIdentifier
    division_metadata: Vector[ManifestDivision, NUM_DIVISIONS]
```

### Informative (not required to use the index)

The following containers illustrate the structure and operation of the index and are referred
to in subsequent sections. No SSZ serialization/deserialization is performed on these containers.

Use of the index or manifest does not require implementation of these containers.

#### `AddressDivision`

This is the functional unit that a user acquires from a peer.
A collection of data for similar addresses. A division contains serialized and compressed
[subsets](#addressindexsubset) that cover discrete block ranges of transaction data (one
subset for every `BLOCK_RANGE` blocks).

Total members is calculated by latest block height divided by `BLOCK_RANGE_WIDTH`.

The elements in the members list are sorted lexicographically by their
oldest block (`AddressIndexSubset.range.old`) field.

```python
class AddressDivision(Container):
    identifier: DivisionIdentifier
    members: List[AddressIndexSubset, MAX_RANGES]
```

#### `AddressAppearanceIndex`

The entire address-appearance-index, divided into [`AddressDivision`](#addressdivision)s that
contain block [`AddressIndexSubset`](#addressindexsubset)s,
ultimately containing transaction identifiers as [`AppearanceTx`](#appearancetx)s.
The index is divided into `NUM_DIVISIONS` functional units.

The index is a derivative of the Unchained Index, and contains the same data, reorganised
for a different purposse.

```python
class AddressAppearanceIndex(Container):
    divisions: Vector[AddressDivision, NUM_DIVISIONS]
```

## Definitions and aliases

The following terms may be used in subsequent descriptions.

*Address*: An identifier used for wallets and accounts in Ethereum. Is a byte vector
of length `BYTES_PER_ADDRESS`. Usually displayed in hexadecimal representation.

*Division*: See [`AddressDivision`](#addressdivision).

*Division identifier*: The hexadecimal characters that is common to all addresses in an [`AddressDivision`](#addressdivision). E.g., `0xa3`.

*Manifest*: The [`IndexManifest`](#indexmanifest) that contains metadata about a particular
instantiation of the index. Useful for obtaining and checking the index data.

*Subset*: See [`AddressIndexSubset`](#addressindexsubset).

*Subset identifier*: The block height of the oldest block in a
[`AddressIndexSubset`](#addressindexsubset).
E.g., subset `13_000_000` contains blocks `13_000_000` to `13_099_999` inclusive.

*Similar address*: Two addresses are similar if they share their first characters in hexadecimal representation to a depth of `ADDRESS_CHARS_SIMILARITY_DEPTH`.

*The index*: The address-appearance-index. See [`AddressAppearanceIndex`](#addressappearanceindex).

## Overview of the address-appearance-index

### Service provided by the index

A mapping of addresses to appearances.

There are two types of agents that are defined below for separation of subsequent
procedure descriptions.

#### Agent type: Maintainer

A entity that has the ability to create or distribute the full index.

#### Agent type: User

A entity that has a part of the index (such as one or more [division](#addressdivision)s).

### Service assumed from the data layer

The following are assumed to be available from external systems.

- A complete index of appearances of all address for all transactions in the network.
  - Unchained Index, either
    - Obtained from peers (IPFS)
    - Constructed from local node, using:
      - A node that can execute all transaction traces.
      - [trueblocks-core](#trueblocks-core).

### Service assumed from the network layer

- Providers of the address-appearance-index
  - Peers in peer-to-peer network
    - Portal network
- Users of the address-appearance-index

### Functions of the index

## Index architecture

Descriptions of components of the index.

### Data architecture description

- Address group.
- Block range division.
- Serialization.
- Compression.

#### String naming conventions

In the case where files are being stored, the following naming conventions are recommended.
This makes direct file transfer between peers more robust in some situations. The "{}" braces
are excluded from the file name. Patterns are provided using regular expressions.

Directory representing [the index](#addressappearanceindex), where:
- "NETWORK" is the ASCII-decoded string representation of the
[network identifier](#networkidentifier).
```sh
Format: address_appearance_index_{NETWORK}

Regular expression: "/^address_appearance_index_+[a-z]$/"

Example directory: ./address_appearance_index_mainnet/
```

Directory representing a [division](#addressdivision), named using the
[division identifier](#divisionidentifier), where:

- "DIVISION_DESCRIPTOR" is the hexadecimal string representation of the
[division identifier](#divisionidentifier) with `ADDRESS_CHARS_SIMILARITY_DEPTH` characters.
```sh
Format: division_0x{DIVISION_DESCRIPTOR}

Regular expression: "/^division_0x[0-9]{ADDRESS_CHARS_SIMILARITY_DEPTH}$/"

Example directory: ./division_Ox4e/
```

File representing a [subset](#addressindexsubset), named using the
[subset identifier](#subsetidentifier), with components:

- "DIVISION_DESCRIPTOR"
  - The hexadecimal string representation of the [division identifier](#divisionidentifier), that has `ADDRESS_CHARS_SIMILARITY_DEPTH` characters.
- "SUBSET_DESCRIPTOR"
  - The decimal string representation of the [subset identifier](#subsetidentifier).
  - Decimal string is left-padded to 9 decimal characters then divided into groups of 3 characters. Block `14_500_000` is shown in the example.
- "ENCODING", which may be one of two choices:
  - "ssz" for data encoded with [SSZ](#ssz-spec) serialization.
  - "ssz_snappy" for data encoded with [SSZ](#ssz-spec) serialization followed by
  encoding with [snappy](#snappy). This format is preferred for network transmission.

```sh
Format: division_0x{DIVISION_DESCRIPTOR}_subset_{SUBSET_DESCRIPTOR}.{ENCODING}

Regular Expression: "/^division_0x[0-9]{ADDRESS_CHARS_SIMILARITY_DEPTH}_subset(_[0-9]{3}){3}.ssz(_snappy)?$/"

Example file: ./division_0x4e_subset_014_500_000.ssz
Example file: ./division_0x4e_subset_014_500_000.ssz_snappy
```

File representing a [manifest](#indexmanifest), named with component:

- "ENCODING", which may be one of two choices:
  - "ssz" for data encoded with [SSZ](#ssz-spec) serialization.
  - "ssz_snappy" for data encoded with [SSZ](#ssz-spec) serialization followed by
  encoding with [snappy](#snappy). This format is preferred for network transmission.

```sh
Format: manifest.{ENCODING}

Regular Expression: "/^manifest.ssz(_snappy)?$/"

Example file: ./manifest.ssz
Example file: ./manifest.ssz_snappy
```

Example of a suggested complete index folder structure:
```sh
- ./address_appearance_index_mainnet/
    - manifest.ssz_snappy
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
### Manifest architecture description

The manifest is a file that serves as a reference for users to maintain up to date data. It
contains the [index manifest](#indexmanifest) data encoded with
[SSZ](#ssz-spec) serialization followed by encoding with [snappy](#snappy). It contains:

- Specification version
- Manifest version
- Lists of files and their hashes.

The manifest is produced by maintenance [creation](#maintenance-creation) and [extension](#maintenance-extension) procedures and utilised during user
[completeness audits](#user-completeness-audit). This enables a user to efficiently determine
the the nature of any pieces of the index that they require from a third party.

## Procedures

Descriptions of actions that agents ([maintainer](#agent-type-maintainer) or
[user](#agent-type-user)) in relation to the index.

"Maintenance" procedures are performed by [maintainer](#agent-type-maintainer) agent types.
"User" procedures are performed by [user](#agent-type-maintainer) agent types.

### Maintenance: Creation

- Index: for each appearance, store under the relevant division -> subset -> address.
- Manifest: Save specification version and decide manifest version then for each subset file, hash and append to file.

### Maintenance: Extension

- Index: for each new appearance, store under the relevant division -> subset -> address.
- Manifest: Save specification version and increment manifest version then for each subset file, hash and append to file.

### Maintenance: Correctness audit

- For a selected appearance(s), check that the transaction appears in the reference ground truth
(E.g., Unchained Index or tracing node).

### User: Consumption

- For a given address go through each subset file and look up the trasaction information for each appearance. Use transaction information to request transaction data from other services
(portal network peer)

### User: Completeness audit

- Obtain a manifest from a peer. For each subset file, compute the root tree hash and verify
it matches the record in the manifest. Check that there are no missing subset files for the
given division.

## Design principles

Local first: Users posess the data that is important to them locally.

User first: Data is useful for basic address activity introspection.

Privacy first: A user can obtain knowledge of their own address without directly sharing the
address with a third party. They reveal which wedge of the address pie (e.g., 1/256th) they
belong in.

Low disk and bandwidth: Useful information can be obtained with <500MB data.

Resilient: The index can be reconstructed if lost.

Multiclient: The index can be used by different projects, wherever information about a particular
address is required (such as wallets and light-style clients).

## Versioning

This specification is versioned according to [semver][1]. This may be useful in the situation
where a user has received parts of the index from two different sources. By examining the
manifest from each source, the peer can decide if the sources are comptatible (use both) or not
(select one).


[1]: https://semver.org/