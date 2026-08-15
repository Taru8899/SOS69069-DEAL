// SPDX-License-Identifier: MIT
pragma solidity 0.8.36;

/**
 * @title SOS69069 DEAL
 * @author SOS69069
 * @notice Reputation-based deal confirmation layer on top of SOS69069 PRESENCE (69P).
 * @dev
 * Name   : SOS69069 DEAL
 * Symbol : 69D
 *
 * Compatible with SOS69069 PRESENCE (symbol "69P"):
 *   - No getPresence(id)
 *   - Data lives in actionOf[poster][typeId] (typeId 1–9)
 *   - Accept is by (poster, typeId), not by a numeric presence id
 *
 * Flow:
 * 1. Poster calls createPresence on SOS69069 PRESENCE (stores action strings).
 * 2. Seeker calls acceptOffer(poster, typeId).
 * 3. Both parties call doMyPart(dealId, externalTxHash).
 * 4. Counterparty calls confirmDeal(dealId) → mints 1 SOS to the poster.
 *
 * Reputation:
 * - accepts, confirmations, partsDone per address
 * - Goal: accepts ≈ confirmations
 *
 * SOS contract : 0x61af906f53Eb927790055AC8eA99916a01873c15
 * CREATOR      : 0x1C10e6574ee696f54b21A611a21313E4714628ad
 *
 * Presence (69P) is passed in the constructor (not hardcoded).
 * First version: 0x40c24f56E5a492b3EB26c287E5cA2604BD29E1BF
 */

interface ISOS {
    function balanceOf(address account) external view returns (uint256);
    function pushCountOf(address user) external view returns (uint256);
    function pushTo(address to) external;
}

/// @notice Minimal interface for SOS69069 PRESENCE (69P).
interface IPresence {
    function actionOf(address poster, uint8 typeId) external view returns (string memory);
}

contract SOS69069DEAL {

    string public constant name   = "SOS69069 DEAL";
    string public constant symbol = "69D";

    address public constant SOS_CONTRACT = 0x61af906f53Eb927790055AC8eA99916a01873c15;
    address public constant CREATOR      = 0x1C10e6574ee696f54b21A611a21313E4714628ad;

    ISOS public immutable sos;
    IPresence public immutable presence;

    enum DealStatus { None, Open, PartDone, Confirmed, Cancelled }

    struct Deal {
        address poster;
        address seeker;
        uint8   typeId;          // 1–9 from Presence
        string  actionSnapshot;  // actionOf[poster][typeId] at accept time
        DealStatus status;
        bytes32 posterPartTx;
        bytes32 seekerPartTx;
        uint256 posterSosBefore;
        uint256 seekerSosBefore;
        uint256 createdAt;
        uint256 confirmedAt;
    }

    uint256 public nextDealId = 1;
    mapping(uint256 => Deal) public deals;

    // Reputation
    mapping(address => uint256) public accepts;
    mapping(address => uint256) public confirmations;
    mapping(address => uint256) public partsDone;

    // Prevent multiple open deals for the same (poster, typeId)
    // key = keccak256(poster, typeId) → dealId (0 = none / free)
    mapping(bytes32 => uint256) public activeDealOf;

    event DealAccepted(
        uint256 indexed dealId,
        address indexed seeker,
        address indexed poster,
        uint8 typeId,
        string actionSnapshot,
        uint256 timestamp
    );

    event PartDone(
        uint256 indexed dealId,
        address indexed who,
        bytes32 externalTxHash,
        uint256 sosBalance,
        uint256 timestamp
    );

    event DealConfirmed(
        uint256 indexed dealId,
        address indexed confirmedBy,
        address indexed poster,
        uint256 timestamp
    );

    event DealCancelled(
        uint256 indexed dealId,
        address indexed who,
        uint256 timestamp
    );

    /**
     * @param presence_ Address of SOS69069 PRESENCE (69P).
     *        First version: 0x40c24f56E5a492b3EB26c287E5cA2604BD29E1BF
     */
    constructor(address presence_) {
        require(presence_ != address(0), "Presence required");
        sos = ISOS(SOS_CONTRACT);
        presence = IPresence(presence_);
    }

    /**
     * @notice Accept an existing Presence entry. Opens a Deal.
     * @param poster The address that posted the Presence (from ActionsRecorded).
     * @param typeId Type 1–9 used in that Presence.
     */
    function acceptOffer(address poster, uint8 typeId) external returns (uint256 dealId) {
        require(poster != address(0), "Poster required");
        require(poster != msg.sender, "Cannot accept own offer");
        require(typeId >= 1 && typeId <= 9, "invalid type");

        bytes32 key = keccak256(abi.encodePacked(poster, typeId));
        require(activeDealOf[key] == 0, "Active deal already exists for this presence");

        string memory action = presence.actionOf(poster, typeId);
        require(bytes(action).length != 0, "No presence data for this type");

        dealId = nextDealId++;
        Deal storage d = deals[dealId];
        d.poster          = poster;
        d.seeker          = msg.sender;
        d.typeId          = typeId;
        d.actionSnapshot  = action;
        d.status          = DealStatus.Open;
        d.createdAt       = block.timestamp;
        d.posterSosBefore = sos.balanceOf(poster);
        d.seekerSosBefore = sos.balanceOf(msg.sender);

        activeDealOf[key] = dealId;
        accepts[msg.sender]++;

        emit DealAccepted(
            dealId,
            msg.sender,
            poster,
            typeId,
            action,
            block.timestamp
        );
    }

    /**
     * @notice Record that you performed your side of the deal.
     * @param dealId         Deal id
     * @param externalTxHash Proof (ETH tx hash, etc.). Use bytes32(0) if none.
     */
    function doMyPart(uint256 dealId, bytes32 externalTxHash) external {
        Deal storage d = deals[dealId];
        require(d.status == DealStatus.Open || d.status == DealStatus.PartDone, "Deal not open");

        uint256 bal = sos.balanceOf(msg.sender);

        if (msg.sender == d.poster) {
            d.posterPartTx = externalTxHash;
        } else if (msg.sender == d.seeker) {
            d.seekerPartTx = externalTxHash;
        } else {
            revert("Not a party to this deal");
        }

        d.status = DealStatus.PartDone;
        partsDone[msg.sender]++;

        emit PartDone(dealId, msg.sender, externalTxHash, bal, block.timestamp);
    }

    /**
     * @notice Counterparty confirms the other side did their part.
     *         Mints 1 SOS to the original Presence poster.
     */
    function confirmDeal(uint256 dealId) external {
        Deal storage d = deals[dealId];
        require(d.status == DealStatus.PartDone || d.status == DealStatus.Open, "Cannot confirm");
        require(d.poster != address(0), "Invalid deal");

        if (msg.sender == d.seeker) {
            require(d.posterPartTx != bytes32(0), "Poster has not recorded their part");
        } else if (msg.sender == d.poster) {
            require(d.seekerPartTx != bytes32(0), "Seeker has not recorded their part");
        } else {
            revert("Not a party to this deal");
        }

        d.status = DealStatus.Confirmed;
        d.confirmedAt = block.timestamp;
        confirmations[msg.sender]++;

        // Clear active slot so another deal can be opened later for same (poster, typeId)
        bytes32 key = keccak256(abi.encodePacked(d.poster, d.typeId));
        if (activeDealOf[key] == dealId) {
            activeDealOf[key] = 0;
        }

        // Mint 1 SOS to the original Presence poster
        sos.pushTo(d.poster);

        emit DealConfirmed(dealId, msg.sender, d.poster, block.timestamp);
    }

    function cancelDeal(uint256 dealId) external {
        Deal storage d = deals[dealId];
        require(d.status == DealStatus.Open, "Only open deals can be cancelled");
        require(msg.sender == d.seeker || msg.sender == d.poster, "Not a party");

        d.status = DealStatus.Cancelled;

        bytes32 key = keccak256(abi.encodePacked(d.poster, d.typeId));
        if (activeDealOf[key] == dealId) {
            activeDealOf[key] = 0;
        }

        emit DealCancelled(dealId, msg.sender, block.timestamp);
    }

    // -------------------------------------------------------------------------
    // Views
    // -------------------------------------------------------------------------

    function getDeal(uint256 dealId) external view returns (Deal memory) {
        return deals[dealId];
    }

    function getReputation(address user) external view returns (
        uint256 _accepts,
        uint256 _confirmations,
        uint256 _partsDone
    ) {
        return (accepts[user], confirmations[user], partsDone[user]);
    }

    function reputationScore(address user) external view returns (int256) {
        // 2*confirmations - accepts (closer to 0 or positive is better)
        return int256(confirmations[user]) * 2 - int256(accepts[user]);
    }

    function activeDealKey(address poster, uint8 typeId) external pure returns (bytes32) {
        return keccak256(abi.encodePacked(poster, typeId));
    }

    receive() external payable { revert("No ETH"); }
    fallback() external payable { revert("No ETH"); }
}