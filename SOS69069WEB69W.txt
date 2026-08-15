// SPDX-License-Identifier: MIT
pragma solidity ^0.8.36;

/**
 * @title SOS69069WEB
 * @notice Central, permissionless ledger of "archive-type" contracts (like the horse
 *         data archive). Anyone can submit a contract address + topic + keywords +
 *         free-form metadata. Entries are stored on-chain, emitted as events for easy
 *         off-chain indexing, and are searchable on-chain by exact keyword.
 * @dev Deliberately kept small (no big text blobs) so it never approaches the
 *      24,576-byte EIP-170 runtime size limit, regardless of how many contracts get
 *      registered into it. This is the "central entry point" — the data itself still
 *      lives in each individual archive contract.
 */
contract SOS69069WEB {

    string public constant name = "SOS69069 WEB";
    string public constant symbol = "69W";

    struct ArchiveEntry {
        address contractAddress; // deployed address of the archive-type contract
        string topic;            // short human-readable title, e.g. "Horse Knowledge Archive"
        string keywords;         // comma-separated keywords as submitted, stored for display
        string metadata;         // free-form text or JSON: description, version, links, author, etc.
        address submitter;
        uint256 timestamp;
    }

    ArchiveEntry[] private archives;

    // keccak256(lowercased keyword) => list of archive ids tagged with that keyword
    mapping(bytes32 => uint256[]) private keywordIndex;

    // prevents the same contract address from being registered twice
    mapping(address => bool) public isRegistered;

    // convenience: address => id, for quick lookup after registering
    mapping(address => uint256) public idOf;

    event ArchiveSubmitted(
        uint256 indexed id,
        address indexed contractAddress,
        address indexed submitter,
        string topic,
        string keywords,
        string metadata,
        uint256 timestamp
    );

    /// @notice Register a new archive-type contract into the ledger.
    /// @param contractAddress The deployed address of the archive contract being indexed.
    /// @param topic Short human-readable topic/title, e.g. "Horse Knowledge Archive".
    /// @param keywordList Array of individual search keywords, e.g. ["horses","equine","history"].
    /// @param metadata Free-form text (or JSON) with any extra description, version, links, etc.
    /// @return id The index assigned to this entry.
    function submitArchive(
        address contractAddress,
        string calldata topic,
        string[] calldata keywordList,
        string calldata metadata
    ) external returns (uint256 id) {
        require(contractAddress != address(0), "zero address");
        require(!isRegistered[contractAddress], "already registered");
        require(bytes(topic).length > 0, "topic required");
        require(keywordList.length > 0, "at least one keyword required");

        id = archives.length;

        bytes memory kwJoined;
        for (uint256 i = 0; i < keywordList.length; i++) {
            require(bytes(keywordList[i]).length > 0, "empty keyword");
            bytes32 kwHash = keccak256(bytes(_toLower(keywordList[i])));
            keywordIndex[kwHash].push(id);
            if (i > 0) {
                kwJoined = bytes.concat(kwJoined, ", ");
            }
            kwJoined = bytes.concat(kwJoined, bytes(keywordList[i]));
        }
        string memory kwString = string(kwJoined);

        archives.push(ArchiveEntry({
            contractAddress: contractAddress,
            topic: topic,
            keywords: kwString,
            metadata: metadata,
            submitter: msg.sender,
            timestamp: block.timestamp
        }));

        isRegistered[contractAddress] = true;
        idOf[contractAddress] = id;

        emit ArchiveSubmitted(id, contractAddress, msg.sender, topic, kwString, metadata, block.timestamp);
    }

    /// @notice Total number of registered archives.
    function totalArchives() external view returns (uint256) {
        return archives.length;
    }

    /// @notice Fetch a single archive entry by id.
    function getArchive(uint256 id) external view returns (
        address contractAddress,
        string memory topic,
        string memory keywords,
        string memory metadata,
        address submitter,
        uint256 timestamp
    ) {
        require(id < archives.length, "invalid id");
        ArchiveEntry storage a = archives[id];
        return (a.contractAddress, a.topic, a.keywords, a.metadata, a.submitter, a.timestamp);
    }

    /// @notice Paginated list of archive ids, oldest first. Use for browsing.
    /// @param offset Starting index.
    /// @param limit Max number of ids to return.
    function getArchiveIds(uint256 offset, uint256 limit) external view returns (uint256[] memory ids) {
        uint256 total = archives.length;
        if (offset >= total) return new uint256[](0);
        uint256 end = offset + limit;
        if (end > total) end = total;
        ids = new uint256[](end - offset);
        for (uint256 i = offset; i < end; i++) {
            ids[i - offset] = i;
        }
    }

    /// @notice Returns archive ids that were tagged with `keyword` at submission time.
    ///         Matching is exact (case-insensitive), not fuzzy/substring.
    function searchByKeyword(string calldata keyword) external view returns (uint256[] memory) {
        return keywordIndex[keccak256(bytes(_toLower(keyword)))];
    }

    /// @notice Lookup the registry id for a known contract address. Reverts implicitly to 0
    ///         if never registered — check isRegistered first if that distinction matters.
    function getIdByAddress(address contractAddress) external view returns (uint256) {
        return idOf[contractAddress];
    }

    function _toLower(string memory str) private pure returns (string memory) {
        bytes memory b = bytes(str);
        for (uint256 i = 0; i < b.length; i++) {
            if (b[i] >= 0x41 && b[i] <= 0x5A) {
                b[i] = bytes1(uint8(b[i]) + 32);
            }
        }
        return string(b);
    }

    // ================= TEMPLATE: how to build a compatible data contract =================

    /// @notice Returns a minimal example Solidity source showing the expected structure
    ///         for a "data contract" that can be registered into this ledger. This mirrors
    ///         the pattern used by the Horse Archive — a token-style name/symbol
    ///         facade, a set of `public pure` "chapter" functions returning strings, one
    ///         `fullArchive()` that concatenates them, and a `dataVersion()` tag — but with
    ///         placeholder one-line text instead of full paragraphs. Copy this structure,
    ///         swap in your own content, deploy, then call `submitArchive()` on this
    ///         registry with the deployed address to index it.
    function dataContractTemplate() external pure returns (string memory) {
        return
            "// SPDX-License-Identifier: MIT\n"
            "pragma solidity ^0.8.36;\n"
            "\n"
            "contract MyDataArchive {\n"
            "\n"
            "    // Token-style facade, purely cosmetic, mirrors the registry pattern\n"
            "    string public constant name = \"My Data Archive\";\n"
            "    string public constant symbol = \"MDA\";\n"
            "\n"
            "    // One function per \"chapter\" of content. Keep each self-contained.\n"
            "    // Use `public pure` (not `external`) so fullArchive() can call them internally.\n"
            "    function chapterOne() public pure returns (string memory) {\n"
            "        return \"Short example text for chapter one.\";\n"
            "    }\n"
            "\n"
            "    function chapterTwo() public pure returns (string memory) {\n"
            "        return \"Short example text for chapter two.\";\n"
            "    }\n"
            "\n"
            "    // Add as many chapterN() functions as you need, each with its own topic.\n"
            "\n"
            "    // Concatenates every chapter into one readable string.\n"
            "    function fullArchive() public pure returns (string memory) {\n"
            "        return string.concat(chapterOne(), \" \", chapterTwo());\n"
            "    }\n"
            "\n"
            "    // A simple version/date tag, useful for the registry's metadata field.\n"
            "    function dataVersion() public pure returns (string memory) {\n"
            "        return \"MyDataArchive v1\";\n"
            "    }\n"
            "}\n";
    }
}