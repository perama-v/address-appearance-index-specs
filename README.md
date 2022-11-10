# address-appearance-index-specs

A specification of an index that maps addresses to the
transactions that they appear in.

The index conforms to ERC-time-ordered-distributed-database.

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
      - [`AddressIndexVolumeChapter`](#addressindexvolumechapter)
    - [Metadata (required to use the index manifest)](#metadata-required-to-use-the-index-manifest)
      - [`VolumeIdentifier`](#volumeidentifier)
      - [`ManifestVolumeChapter`](#manifestvolume)
      - [`ChapterIdentifier`](#chapteridentifier)
      - [`ManifestChapter`](#manifestchapter)
      - [`NetworkName`](#networkname)
      - [`IndexManifest`](#indexmanifest)
    - [Informative (not required to use the index)](#informative-not-required-to-use-the-index)
      - [`AddressChapter`](#addresschapter)
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
    - [User: Find transactions](#user-find-transactions)
    - [User: Check completeness](#user-check-completeness)
  - [Design principles](#design-principles)
  - [Versioning](#versioning)

<!-- END doctoc -->

## Introduction

The index structure definition begins at [`AddressAppearanceIndex`](#addressappearanceindex).

## Scope

- Index architecture:
    - Data
        - Address chapters
        - address index volumes
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
    - User: Find transactions
    - User: Check completeness

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
| `BLOCKS_PER_VOLUME` | `uint32(10*5)` (=100_000) | Number of blocks in a address index volume. Index maintenance operates at this cadence (~2 weeks) |

### Fixed-size type parameters


| Name | Value | Description |
| - | - | - |
| `DEFAULT_BYTES_PER_ADDRESS` | `uint32(20)` | Number of bytes for an address (E.g., `20` bytes on mainnet). May be different on some networks. |
| `MAX_NETWORK_NAME_BYTES` | `uint32(2**5)` (=32) | Maximum number of bytes available to represent the ASCII-encoded string network name. E.g. The string ["mainnet"](https://eips.ethereum.org/EIPS/eip-2228) is 7 bytes. |
| `MAX_SCHEMAS_RESOURCE_BYTES` | `uint32(2**7)` (=128) |  Maximum number of bytes available to represent the ASCII-encoded string that refers to this specification. |
| `MAX_PUBLISH_ID_BYTES` | `uint32(2**6)` (=64) | Maximum number of bytes available to represent the ASCII-encoded string to publishing the manifest under. |


### Variable-size type parameters

Helper values for SSZ operations. SSZ variable-size elements require a maximum length field.

| Name | Value | Description |
| - | - | - |
| `MAX_ADDRESSES_PER_VOLUME` | `uint32(2*30)` (=1_073_741_824) | Chosen as practical ceiling for number of addresses plausible in `BLOCKS_PER_VOLUME` blocks. Not rigorous. |
| `MAX_TXS_PER_VOLUME` | `uint32(2*30)` (=1_073_741_824) | Chosen as practical ceiling for number of transactions plausible in `BLOCKS_PER_VOLUME` blocks. Not rigorous. |
| `MAX_VOLUMES` | `uint32(2**16)` (=65_536) | Chosen as practical ceiling for number of volumes if volumes containing `BLOCKS_PER_VOLUME`. |
| `MAX_BYTES_PER_CID` |  `uint32(2**7)` (=128) | Number of bytes an IPFS CID may use (CIDv1 has variable length) |

### Derived

Constants derived from [design parameters](#design-parameters).

| Name | Value | Description |
| - | - | - |
| `NUM_CHAPTERS` | `uint32(16**ADDRESS_CHARS_SIMILARITY_DEPTH)` (=256) | Number of unique hex character combinations possible. Represents how many pieces the index can be split. |
| `NUM_COMMON_BYTES` | `uint32((ADDRESS_CHARS_SIMILARITY_DEPTH + 1) // 2)` (=1) | The number of bytes needed to represent the `ADDRESS_CHARS_SIMILARITY_DEPTH` characters. E.g., 1 or 2 chars is 1 byte, 3 or 4 chars is 2 bytes|

## Containers

The ordering of the fields for each container is defined in the code snippets below and is
important for serialization/deserialization.

### Data (required to use the index data)

The `AddressIndexVolumeChapter` is the main container in this section and is
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
within the range defined in the parent AddressIndexVolumeChapter.

The elements in the appearances list are sorted lexicographically first by their block
(`AppearanceTx.block`) field, and then by their index (`AppearanceTx.index`) field
(where the address appears twice in a block).

```python
class AddressAppearances(Container):
    address: ByteVector[DEFAULT_BYTES_PER_ADDRESS]
    appearances: List[AppearanceTx, MAX_TXS_PER_VOLUME]
```

#### `AddressIndexVolumeChapter`

Many of these units form a functional unit for a user.
It is the unit that is subject to SSZ serialization then compression
or decompression then deserialization.

Contains a collection representing transaction that meet both the address and block
range requirements.

The elements in the addresses list are sorted lexicographically by their address
(`AddressAppearances.address`) field

Components:
- `address_prefix`. The byte representation of the common characters that the addresses
all start with. Defined by the parent [`chapter`](#addresschapter) and included here for
clarity.
- `identifier`. The [volume identifier](#volumeidentifier) that defines which blocks the volume
includes.
- `addresses`. Contains transaction data for addresses.

```python
class AddressIndexVolumeChapter(Container):
    address_prefix: ByteVector[NUM_COMMON_BYTES]
    identifier: VolumeIdentifier
    addresses: List[AddressAppearances, MAX_ADDRESSES_PER_VOLUME]
```

### Metadata (required to use the index manifest)

The `IndexManifest` is the main container in this section and is
subject to SSZ serialization/deserialization to create/access the index manifest.

The remaining containers are components of that container.
Use of the index manifest requires implementation of these containers.

#### `VolumeIdentifier`

Used to refer to a particular [volume](#addressindexvolumechapter).
The identifier is used in the [manifest](#manifest) and in the
[naming conventions](#string-naming-conventions) for index files.

Components:
- `oldest_block` defines the oldest block that a [volume](#addressindexvolumechapter] includes.
  - Volumes represent `BLOCKS_PER_VOLUME` blocks, therefore the inclusive range is
  `[oldest_block, oldest_block + BLOCKS_PER_VOLUME - 1]`.
  - E.g., a volume identifier with `oldest_block=15_300_000` and `BLOCKS_PER_VOLUME=100_000` then
  blocks included are `[15_300_000, 15_399_999]`.

```python
class VolumeIdentifier(Conainer):
    oldest_block: uint32
```

#### `ManifestVolumeChapter`

Used to create the index [manifest](#manifest). This unit represents a store for
the hash of a single
[`AddressIndexVolumeChapter`](#addressindexvolumechapter).

Components:
- `identifier`, the [VolumeIdentifier](#volumeidentifier) that refers to a particular volume.
- `ipfs_cid`, the byte representation of an Interplanetary File System (IPFS) Content Identifier (CID). This CID may be either v0 or V1.
- The hash is the tree root hash,
as defined in the [ssz spec](#ssz-spec).


```python
class ManifestVolumeChapter(Container):
    identifier: VolumeIdentifier
    ipfs_cid: List[byte, MAX_BYTES_PER_CID]
    hash_tree_root: uint256
```

#### `ChapterIdentifier`

An identifier defines the member addresses for a [chapter](#addresschapter).
It is the byte representation of the hex characters that are common to all addresses
within a chapter E.g., byte representation of `0x5a`.

The identifier is also used in the [manifest](#manifestchapter) and in
[naming conventions](#string-naming-conventions) for index directories.

Components:
- `address_common_bytes` the byte representation of the hex characters that similar addresses
share. The number of characters are defined by `ADDRESS_CHARS_SIMILARITY_DEPTH`, and
the bytes needed to represent these characters are defined as `NUM_COMMON_BYTES`.

```python
class ChapterIdentifier(Container):
    address_common_bytes: ByteVector[NUM_COMMON_BYTES]
```

#### `ManifestChapter`

Used to create the index [manifest](#manifest).
This unit stores the [volume metadata](#manifestvolume) for a single
[`AddressChapter`](#addresschapter) with a given [identifier](#chapteridentifier).

The elements in the `volume_metadata` list are sorted lexicographically by their
identifier (`ManifestVolumeChapter.identifier`) field.

```python
class ManifestChapter(Container):
    identifier: ChapterIdentifier
    volume_metadata: List[ManifestVolumeChapter, MAX_VOLUMES]
```

#### `NetworkName`

Used to identify the network that the index data belongs to. The network
is represented as an ASCII-encoded byte vector with maximum length `MAX_NETWORK_NAME_BYTES`.

```python
class NetworkName(Container):
    name: List[uint8, MAX_NETWORK_NAME_BYTES]
```

#### `IndexSpecificationVersion`

Used to manage compatibility between different instantiations of data. For
example, a breaking change that one party has upgraded to, sharing a manifest with
another who has not upgraded. The version notifies the second party that some
of their data may need to be replaced.

This specification is versioned according to [semver][1]. This may be useful in the situation
where a user has received parts of the index from two different sources. By examining the
manifest from each source, the peer can decide if the sources are compatible (use both) or not
(select one).

[1]: https://semver.org/

```python
class IndexSpecificationVersion(Container):
    spec_version_major: uint32
    spec_version_minor: uint32
    spec_version_patch: uint32
```

#### `IndexSpecificationSchemas`

The ASCII-encoded bytes that can be used to obtain this specification.

The specification may then be a resource for using the manifest. For example, a link to this
github page or an Interplanetary File System (IPFS) Content Identifier (CID).

The byte ASCII-encoded string has a maximum byte length of `MAX_SCHEMAS_RESOURCE_BYTES`.

```python
class IndexSpecificationSchemas(Container):
    resource: List[uint8, MAX_SCHEMAS_RESOURCE_BYTES]
```

#### `IndexPublishingIdentifier`

The  ASCII-encoded bytes that can be used to publish or broadcast the manifest with.

This serves as a coordination mechanism for parties sharing the manifest. For example,
the manifest hash may be published and looked up in a smart contract using this identifier.

The decoded string must a concatenation of "address_appearance_index_" and the network
ASCII-decoded `NetworkName`. For example "address_appearance_index_mainnet".

```python
class IndexPublishingIdentifier(Container):
    topic: List[uint8, MAX_PUBLISH_ID_BYTES]
```

#### `IndexManifest`

This is a container that represents a file containing metadata about the index.
It is a record of the components of a specific instantiation of the index.
The manifest keeps the hash of all [`AddressIndexVolumeChapter`](#addressindexvolumechapter)s.
If the index is updated to include newer data as the chain progresses,
a new manifest is generated to reflect the contents of the index.

The manifest includes:
- The [version](#indexspecificationversion) of the specification used to construct the data.
- The [schemas](#indexspecificationschemas) resource link.
- The [publish_as_topic](#indexpublishingidentifier) identifier.
- The [network](#networkname)
- The [latest_volume_identifier](#volumeidentifier) of the highest (latest) completed [volume](#addressindexvolumechapter).
- The [chapter_metadata](#manifestchapter) which each contain the hashes of the volume.
  - The elements in the chapter metadata vector are sorted lexicographically by their
identifier (`ManifestChapter.identifier`) field.

```python
class IndexManifest(Containr):
    version: IndexSpecificationVersion
    schemas: IndexSpecificationSchemas
    publish_as_topic: IndexPublishingIdentifier
    network: NetworkName
    latest_volume_identifier: VolumeIdentifier
    chapter_metadata: Vector[ManifestChapter, NUM_CHAPTERS]
```

### Informative (not required to use the index)

The following containers illustrate the structure and operation of the index and are referred
to in subsequent sections. No SSZ serialization/deserialization is performed on these containers.

Use of the index or manifest does not require implementation of these containers.

#### `AddressChapter`

This represents the functional collection that a user acquires from a peer.
It consists of data for similar addresses. A chapter contains serialized and compressed
[chapters from many volumes](#addressindexvolumechapter) that cover discrete block ranges of transaction data (one
volume for every `BLOCKS_PER_VOLUME` blocks).

Total members is calculated by latest block height divided by `BLOCKS_PER_VOLUME`.

The elements in the members list are sorted lexicographically by their
oldest block (`AddressIndexVolumeChapter.range.old`) field.

```python
class AddressChapter(Container):
    identifier: ChapterIdentifier
    members: List[AddressIndexVolumeChapter, MAX_VOLUMES]
```

#### `AddressAppearanceIndex`

The entire address-appearance-index, divided into [`AddressChapter`](#addresschapter)s that
contain block [`AddressIndexVolumeChapter`](#addressindexvolumechapter)s,
ultimately containing transaction identifiers as [`AppearanceTx`](#appearancetx)s.
The index is divided into `NUM_CHAPTERS` functional units.

The index is a derivative of the Unchained Index, and contains the same data, reorganised
for a different purpose.

```python
class AddressAppearanceIndex(Container):
    chapters: Vector[AddressChapter, NUM_CHAPTERS]
```

## Definitions and aliases

The following terms may be used in subsequent descriptions.

*Address*: An identifier used for wallets and accounts in Ethereum. Is a byte vector
of length `DEFAULT_BYTES_PER_ADDRESS`. Usually displayed in hexadecimal representation.

*Chapter*: See [`AddressChapter`](#addresschapter).

*Chapter identifier*: The hexadecimal characters that is common to all addresses in an
[`AddressChapter`](#addresschapter). E.g., `0xa3`.

*Manifest*: The [`IndexManifest`](#indexmanifest) that contains metadata about a particular
instantiation of the index. Useful for obtaining and checking the index data.

*Volume*: See [`AddressIndexVolumeChapter`](#addressindexvolumechapter).

*Volume identifier*: The block height of the oldest block in a
[`AddressIndexVolumeChapter`](#addressindexvolumechapter).
E.g., volume `13_000_000` contains blocks `13_000_000` to `13_099_999` inclusive.

*Similar address*: Two addresses are similar if they share their first characters in hexadecimal
representation to a depth of `ADDRESS_CHARS_SIMILARITY_DEPTH`.

*The index*: The address-appearance-index. See [`AddressAppearanceIndex`](#addressappearanceindex).

## Overview of the address-appearance-index

### Service provided by the index

A mapping of addresses to appearances.

There are two types of agents that are defined below for separation of subsequent
procedure descriptions.

#### Agent type: Maintainer

A entity that has the ability to create or distribute the full index.

#### Agent type: User

A entity that has a part of the index (such as one or more [chapter](#addresschapter)s).

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

- Volumes.
- Chapters.
- Serialization.
- Compression.

#### String naming conventions

In the case where files are being stored, the following naming conventions are recommended.
This makes direct file transfer between peers more robust in some situations. The "{}" braces
are excluded from the file name. Patterns are provided using regular expressions.

---
Directory representing the [index](#addressappearanceindex), where:
- "NETWORK" is the ASCII-decoded string representation of the
[network name](#networkName).
```sh
Format: address_appearance_index_{NETWORK}

Regular expression: "/^address_appearance_index_+[a-z]$/"

Example directory: ./address_appearance_index_mainnet/
```
---
Directory representing a [chapter](#addresschapter), named using the
[chapter identifier](#chapteridentifier), where:

- "CHAPTER_DESCRIPTOR" is the hexadecimal string representation of the
[chapter identifier](#chapteridentifier) with `ADDRESS_CHARS_SIMILARITY_DEPTH` characters.
```sh
Format: chapter_0x{CHAPTER_DESCRIPTOR}

Regular expression: "/^chapter_0x[a-z0-9]{ADDRESS_CHARS_SIMILARITY_DEPTH}$/"

Example directory: ./chapter_Ox4e/
```
---
File representing a [volume](#addressindexvolumechapter), named using the
[volume identifier](#volumeidentifier), with components:

- "CHAPTER_DESCRIPTOR"
  - The hexadecimal string representation of the [chapter identifier](#chapteridentifier), that has `ADDRESS_CHARS_SIMILARITY_DEPTH` characters.
- "VOLUME_DESCRIPTOR"
  - The decimal string representation of the [volume identifier](#volumeidentifier).
  - Decimal string is left-padded to 9 decimal characters then divided into groups of 3 characters. Block `14_500_000` is shown in the example.
- "ENCODING", which may be one of two choices:
  - "ssz" for data encoded with [SSZ](#ssz-spec) serialization.
  - "ssz_snappy" for data encoded with [SSZ](#ssz-spec) serialization followed by
  encoding with [snappy](#snappy). This format is preferred for network transmission.

```sh
Format: chapter_0x{CHAPTER_DESCRIPTOR}_volume_{VOLUME_DESCRIPTOR}.{ENCODING}

Regular Expression: "/^chapter_0x[a-z0-9]{ADDRESS_CHARS_SIMILARITY_DEPTH}_volume(_[a-z0-9]{3}){3}.ssz(_snappy)?$/"

Example file: ./chapter_0x4e_volume_014_500_000.ssz
Example file: ./chapter_0x4e_volume_014_500_000.ssz_snappy
```
---
File representing a [manifest](#indexmanifest). The file name contains the
spec version used, which is required to know the definitions of the SSZ types.
The components of the name are:

- "MAJOR", the [`spec_version_major`](#indexmanifest).
- "MINOR", the [`spec_version_minor`](#indexmanifest).
- "PATCH", the [`spec_version_patch`](#indexmanifest).

```sh
Format: manifest_v_{MAJOR}_{MINOR}_{PATCH}.json

Regular Expression: "/^manifest_v(_[0-9]{2}){3}.json$/"

Example file (spec v0.1.2): ./manifest_v_00_01_02.json
```
---
Example of a suggested complete index folder structure:
```sh
- ./address_appearance_index_mainnet/
    - manifest_v_00_01_00.ssz_snappy
    - ...
    - /chapter_Ox4e/
        - ...
        - /chapter_0x4e_volume_014_500_000.ssz_snappy
        - /chapter_0x4e_volume_014_600_000.ssz_snappy
        - ...
    - /chapter_Ox4f/
        - ...
        - /chapter_0x4e_volume_014_500_000.ssz_snappy
        - ...
    - ...
```
#### Estimated file count

The estimated file count for a complete index on a network at a certain block height can
be calculated as follows:
```python
def estimate_file_count(block_height):
  chapter_directories = NUM_CHAPTERS
  volume_files_per_directory = block_height // BLOCKS_PER_VOLUME
  file_count = chapter_directories * volume_files_per_directory
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

The manifest is produced by maintenance [creation](#maintenance-creation) and
[extension](#maintenance-extension) procedures and utilised during user
[check completeness](#user-check-completeness). This enables a user to efficiently determine
the the nature of any pieces of the index that they require from a third party.

## Procedures

Descriptions of actions that agents ([maintainer](#agent-type-maintainer) or
[user](#agent-type-user)) in relation to the index.

"Maintenance" procedures are performed by [maintainer](#agent-type-maintainer) agent types.
"User" procedures are performed by [user](#agent-type-maintainer) agent types.

### Maintenance: Creation

- Index: for each appearance, store under the relevant chapter -> volume -> address.
- Manifest: Save specification version and decide manifest version then for each volume file, hash and append to file.

### Maintenance: Extension

- Index: for each new appearance, store under the relevant chapter -> volume -> address.
- Manifest: Save specification version and increment manifest version then for each volume file, hash and append to file.

### Maintenance: Correctness audit

- For a selected appearance(s), check that the transaction appears in the reference ground truth
(E.g., Unchained Index or tracing node).

### User: Find transactions

- For a given address go through each volume file and look up the trasaction information for
each appearance. Use transaction information to request transaction data from other services
(portal network peer)

### User: Check completeness

- Obtain a manifest from a peer. For each volume file, compute the root tree hash and verify
it matches the record in the manifest. Check that there are no missing volume files for the
given chapter.

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
