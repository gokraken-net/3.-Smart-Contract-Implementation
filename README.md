
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/**
 * @title KrakenToken
 * @dev Main token contract for KrakenNet ecosystem
 */
contract KrakenToken is ERC20, AccessControl, Pausable, ReentrancyGuard {
    bytes32 public constant GOVERNANCE_ROLE = keccak256("GOVERNANCE_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    
    uint256 public constant TOTAL_SUPPLY = 1_000_000_000 * 10**18;  // 1B tokens
    
    // Vesting schedule structure
    struct VestingSchedule {
        uint256 total;
        uint256 released;
        uint256 startTime;
        uint256 cliff;
        uint256 duration;
        bool revocable;
    }
    
    // Mapping from address to vesting schedule
    mapping(address => VestingSchedule) public vestingSchedules;
    
    // Staking parameters
    struct Stake {
        uint256 amount;
        uint256 startTime;
        uint256 multiplier;
        bool isNodeOperator;
    }
    
    mapping(address => Stake) public stakes;
    
    // Events
    event TokensVested(address indexed beneficiary, uint256 amount);
    event TokensStaked(address indexed user, uint256 amount);
    event StakeWithdrawn(address indexed user, uint256 amount);
    event RewardsClaimed(address indexed user, uint256 amount);
    
    constructor() ERC20("KrakenNet", "INK") {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(GOVERNANCE_ROLE, msg.sender);
        _mint(msg.sender, TOTAL_SUPPLY);
    }
    
    /**
     * @dev Creates a vesting schedule for an address
     */
    function createVestingSchedule(
        address beneficiary,
        uint256 amount,
        uint256 cliffDuration,
        uint256 vestingDuration,
        bool revocable
    ) external onlyRole(GOVERNANCE_ROLE) {
        require(vestingSchedules[beneficiary].total == 0, "Schedule exists");
        require(amount > 0, "Amount = 0");
        
        vestingSchedules[beneficiary] = VestingSchedule({
            total: amount,
            released: 0,
            startTime: block.timestamp,
            cliff: cliffDuration,
            duration: vestingDuration,
            revocable: revocable
        });
        
        _transfer(msg.sender, address(this), amount);
    }
    
    /**
     * @dev Releases vested tokens for msg.sender
     */
    function releaseVestedTokens() external nonReentrant {
        VestingSchedule storage schedule = vestingSchedules[msg.sender];
        require(schedule.total > 0, "No schedule");
        
        uint256 releasable = _calculateReleasable(schedule);
        require(releasable > 0, "Nothing to release");
        
        schedule.released += releasable;
        _transfer(address(this), msg.sender, releasable);
        
        emit TokensVested(msg.sender, releasable);
    }
    
    /**
     * @dev Calculates releasable tokens for a schedule
     */
    function _calculateReleasable(VestingSchedule memory schedule) 
        private 
        view 
        returns (uint256) 
    {
        if (block.timestamp < schedule.startTime + schedule.cliff) {
            return 0;
        }
        
        if (block.timestamp >= schedule.startTime + schedule.duration) {
            return schedule.total - schedule.released;
        }
        
        uint256 timeFromStart = block.timestamp - schedule.startTime;
        uint256 vestedAmount = (schedule.total * timeFromStart) / schedule.duration;
        
        return vestedAmount - schedule.released;
    }
    
    /**
     * @dev Stake tokens for governance and rewards
     */
    function stake(uint256 amount, bool asNodeOperator) external nonReentrant {
        require(amount > 0, "Amount = 0");
        require(balanceOf(msg.sender) >= amount, "Insufficient balance");
        
        stakes[msg.sender] = Stake({
            amount: amount,
            startTime: block.timestamp,
            multiplier: asNodeOperator ? 200 : 150,  // 2x for nodes, 1.5x for regular
            isNodeOperator: asNodeOperator
        });
        
        _transfer(msg.sender, address(this), amount);
        emit TokensStaked(msg.sender, amount);
    }
    
    // Additional functions for governance, rewards, and network operations...
}
Made with
Artifacts are user-generated and may contain unverified or potentially unsafe content.
Report
Remix Artifact

