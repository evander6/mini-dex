// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20; // 升级到最新稳定版本

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract ThreeKingdomsNFT is ERC721, Ownable {


    using Counters for Counters.Counter;
    Counters.Counter private _tokenIdCounter;

    struct Character {
        uint16 strength;    // uint16节省gas
        uint16 wisdom;
        uint16 loyalty;
        Rarity rarity;
    }

    enum Rarity { Common, Rare, Epic }
    
    // 内存优化：使用紧凑型存储
    mapping(uint256 => Character) private _characters;
    uint256 public constant MAX_MINT_PER_TX = 5;
    uint256 public mintPrice = 0.001 ether;

    event MintSuccess(address indexed minter, uint256 tokenId, Rarity rarity);

    constructor() ERC721("ThreeKingdomsNFT", "TKNG") Ownable(msg.sender) {}

    // 安全增强的随机数生成
    function _generateRandomTraits() internal view returns (Character memory) {
        bytes32 entropy = keccak256(abi.encodePacked(
            block.prevrandao, 
            msg.sender,
            blockhash(block.number - 1)
        );
        uint256 seed = uint256(entropy);
        
        uint256 baseRand = seed % 100;
        return Character({
            strength: uint16((seed % 90) + 10),   // 武力值10-99
            wisdom: uint16(((seed >> 8) % 90) + 10), 
            loyalty: uint16(((seed >> 16) % 90) + 10),
            rarity: _determineRarity(baseRand)
        });
    }

    function _determineRarity(uint256 rand) internal pure returns (Rarity) {
        if (rand < 5) return Rarity.Epic;
        if (rand < 20) return Rarity.Rare;
        return Rarity.Common;
    }

    // 支持批量铸造（带数量限制）
    function mintCharacters(uint256 quantity) external payable {
        require(quantity <= MAX_MINT_PER_TX, "Exceed mint limit");
        require(msg.value >= mintPrice * quantity, "Insufficient payment");

        for (uint256 i = 0; i < quantity; i++) {
            uint256 tokenId = _tokenIdCounter.current();
            _tokenIdCounter.increment();
            
            _safeMint(msg.sender, tokenId);
            _characters[tokenId] = _generateRandomTraits();
            
            emit MintSuccess(msg.sender, tokenId, _characters[tokenId].rarity);
        }
    }

    // 查询接口（带存在性验证）
    function getCharacter(uint256 tokenId) public view returns (Character memory) {
        require(_ownerOf(tokenId) != address(0), "Token does not exist");
        return _characters[tokenId];
    }

    // 合约管理功能
    function withdraw() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }

    function setMintPrice(uint256 newPrice) external onlyOwner {
        mintPrice = newPrice;
    }
}
