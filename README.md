# ParufixKI-Token.sol
PARUFYX KI Token (KIT): Zugang und Bezahlung für ein Netzwerk aus KI-Diensten auf der BNB Smart Chain. Nutzungs-Token, keine Rendite.
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract PARUFYXKIToken is ERC20, Ownable {
    struct Service {
        string name;
        uint256 price;
        bool active;
    }

    address public treasury;
    uint256 public minHold;
    uint256 public burnBps = 2000; // 20 %

    uint256 public serviceCount;
    mapping(uint256 => Service) public services;

    event ServiceAdded(uint256 indexed serviceId, string name, uint256 price);
    event ServiceUpdated(uint256 indexed serviceId, uint256 price, bool active);
    event ServiceUsed(
        address indexed user,
        uint256 indexed serviceId,
        uint256 price,
        uint256 burned
    );

    constructor(
        address initialOwner,
        address _treasury,
        uint256 totalSupply_,
        uint256 _minHold
    ) ERC20("PARUFYX KI Token", "KIT") Ownable(initialOwner) {
        require(_treasury != address(0), "Treasury fehlt");
        treasury = _treasury;
        minHold = _minHold;
        _mint(initialOwner, totalSupply_);
    }

    function addService(string calldata name, uint256 price) external onlyOwner {
        uint256 id = serviceCount++;
        services[id] = Service(name, price, true);
        emit ServiceAdded(id, name, price);
    }

    function updateService(uint256 serviceId, uint256 price, bool active) external onlyOwner {
        require(serviceId < serviceCount, "Dienst unbekannt");
        services[serviceId].price = price;
        services[serviceId].active = active;
        emit ServiceUpdated(serviceId, price, active);
    }

    function setMinHold(uint256 _minHold) external onlyOwner {
        minHold = _minHold;
    }

    function setBurnBps(uint256 _burnBps) external onlyOwner {
        require(_burnBps <= 10000, "Max 100 %");
        burnBps = _burnBps;
    }

    function setTreasury(address _treasury) external onlyOwner {
        require(_treasury != address(0), "Treasury fehlt");
        treasury = _treasury;
    }

    function payForService(uint256 serviceId) external {
        Service memory s = services[serviceId];
        require(serviceId < serviceCount && s.active, "Dienst nicht verfuegbar");

        uint256 bal = balanceOf(msg.sender);
        require(bal >= s.price, "Zu wenig Token");
        require(bal - s.price >= minHold, "Mindestmenge unterschritten");

        uint256 burnAmount = (s.price * burnBps) / 10000;
        uint256 treasuryAmount = s.price - burnAmount;

        if (burnAmount > 0) _burn(msg.sender, burnAmount);
        _transfer(msg.sender, treasury, treasuryAmount);

        emit ServiceUsed(msg.sender, serviceId, s.price, burnAmount);
    }

    function hasAccess(address user) external view returns (bool) {
        return balanceOf(user) >= minHold;
    }
}
