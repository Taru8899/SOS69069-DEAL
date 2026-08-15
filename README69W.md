# SOS69069WEB – On-chain Archive Registry

**SOS69069WEB** is a minimal, permissionless on-chain registry for “archive-type” smart contracts.

It acts as a permanent, censorship-resistant table of contents for large data contracts (for example the Horse Knowledge Archive and similar projects). The registry itself stays very small and never approaches Ethereum’s 24 576-byte contract size limit, regardless of how many archives are added.

The actual content lives inside the individual archive contracts. This registry only stores pointers + metadata.

---

## Core Idea

- Anyone can deploy a data contract that follows a simple pattern (token-style façade + pure functions returning text).
- Anyone can then register that contract here by calling `submitArchive()`.
- The registry stores:
  - contract address
  - topic / title
  - keywords
  - free-form metadata
  - submitter
  - timestamp
- Everything is emitted as events → easy for indexers, The Graph, or custom scanners.
- On-chain search by exact keyword (case-insensitive) is supported.

This creates a growing, decentralized library of knowledge that lives entirely on Ethereum.

---

## Key Features

| Feature                    | Description                                      |
|---------------------------|--------------------------------------------------|
| Permissionless            | Anyone can submit any archive contract           |
| Permanent                 | Data is stored forever on Ethereum               |
| Searchable                | Exact keyword search (case-insensitive)          |
| Event-driven              | Easy off-chain indexing                          |
| Tiny footprint            | Registry stays far below the 24 KB size limit    |
| Compatible template       | Built-in example of how data contracts should look |

---

## Main Functions

### Writing
- `submitArchive(address, string topic, string[] keywords, string metadata)` → registers a new archive

### Reading
- `totalArchives()` → total number of registered archives
- `getArchive(uint256 id)` → full entry by ID
- `getArchiveIds(uint256 offset, uint256 limit)` → paginated list of IDs
- `searchByKeyword(string keyword)` → returns IDs matching the keyword
- `getIdByAddress(address)` → lookup ID of a known contract
- `isRegistered(address)` → check if an address is already registered
- `dataContractTemplate()` → returns a ready-to-copy example of a compatible data contract

---

## How to add your own archive

1. Deploy a data contract that follows the pattern shown in `dataContractTemplate()` (or the larger Horse Archive style).
2. Call `submitArchive()` on this registry with:
   - the deployed address
   - a clear topic
   - an array of keywords
   - any extra metadata (version, description, links, author, etc.)
3. Your archive is now permanently indexed and discoverable.

---

## Design Philosophy

- The registry is only an **index**.
- Heavy text data stays inside the individual archive contracts (up to \~19–21 KB of useful content each).
- Multiple archives can be combined by front-ends into a browsable on-chain library.
- No admin, no ownership, no upgradeability — pure permissionless permanence.

---

## License

MIT