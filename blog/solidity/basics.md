@def title = "Solidity Guide: Syntax, Concepts & Patterns"
@def published = "26 December 2025"
@def tags = ["solidity"]

# Solidity Guide: Syntax, Concepts & Patterns

## What is Solidity, Really?

You already know it's for smart contracts, but here's the thing: Solidity is basically a language that runs on the Ethereum Virtual Machine (EVM). When you write Solidity code, it gets compiled into bytecode that lives on the blockchain forever. Every function call costs gas (real money!), so writing efficient code isn't just good practice—it's literally saving users cash.

> **What's the EVM?** Think of it as a giant, distributed computer that runs across thousands of nodes worldwide. When you deploy a smart contract, every node runs the same bytecode in this virtual machine. It's deterministic (same input = same output always) and sandboxed (can't access your file system or make random API calls). The EVM processes transactions in 256-bit words and charges gas for every operation—adding two numbers costs 3 gas, storing data costs thousands. It's intentionally slow and expensive to prevent spam and ensure every node can keep up.

## Core Syntax Overview

### The Basics

Every Solidity file starts with a version pragma. This tells the compiler which version to use:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
```

The `^0.8.0` means "use version 0.8.0 or any newer minor version, but not 0.9.0."

### Contract Structure

Think of a contract like a class in object-oriented programming:

```solidity
contract MyContract {
    // State variables (stored on blockchain)
    uint256 public myNumber;
    address public owner;
    
    // Events (for logging)
    event NumberChanged(uint256 newNumber);
    
    // Constructor (runs once on deployment)
    constructor() {
        owner = msg.sender;
    }
    
    // Functions
    function setNumber(uint256 _newNumber) public {
        myNumber = _newNumber;
        emit NumberChanged(_newNumber);
    }
}
```

### Data Types

**Value Types** (stored directly):
- `bool`: true or false
- `uint256`: unsigned integer (0 to 2^256-1)
- `int256`: signed integer
- `address`: 20-byte Ethereum address
- `bytes32`: fixed-size byte array

**Reference Types** (passed by reference):
- `string`: dynamic text
- `bytes`: dynamic byte array
- `arrays`: like `uint256[]` or `uint256[5]`
- `mapping`: like a hash map, e.g., `mapping(address => uint256)`
- `struct`: custom data structures

### Visibility Modifiers

Functions and state variables have visibility:

- `public`: callable by anyone, generates a getter for state variables
- `private`: only callable within this contract
- `internal`: callable within this contract and derived contracts
- `external`: only callable from outside (saves gas vs public)

```solidity
uint256 public publicVar;      // Anyone can read
uint256 private secretVar;     // Only this contract
uint256 internal sharedVar;    // This + child contracts

function publicFunc() public {}
function externalFunc() external {}
function internalFunc() internal {}
function privateFunc() private {}
```

### Function Modifiers

These change how functions behave:

- `view`: reads state but doesn't modify it
- `pure`: doesn't read or modify state
- `payable`: can receive Ether

```solidity
function getBalance() public view returns (uint256) {
    return address(this).balance;
}

function calculateSum(uint a, uint b) public pure returns (uint) {
    return a + b;
}

function deposit() public payable {
    // Can receive ETH
}
```

### Custom Modifiers

These are reusable checks:

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not the owner");
    _;  // This is where the function body gets inserted
}

function criticalFunction() public onlyOwner {
    // Only owner can call this
}
```

### Special Variables

Solidity gives you some built-in globals:

- `msg.sender`: address that called this function
- `msg.value`: amount of Ether sent with the call
- `block.timestamp`: current block timestamp
- `block.number`: current block number
- `address(this).balance`: contract's Ether balance

## How Solidity Actually Works

### The EVM and Gas

Every operation costs gas. Storage is expensive, computation is cheaper. That's why you'll see patterns that minimize storage writes:

```solidity
// Expensive - multiple storage writes
function badExample(uint[] memory data) public {
    for (uint i = 0; i < data.length; i++) {
        myArray.push(data[i]);  // Storage write each iteration
    }
}

// Better - single storage write
function goodExample(uint[] memory data) public {
    uint len = data.length;
    for (uint i = 0; i < len; i++) {
        myArray.push(data[i]);
    }
}
```

### Storage vs Memory vs Calldata

- `storage`: persistent data on the blockchain (expensive)
- `memory`: temporary data during function execution (cheaper)
- `calldata`: read-only temporary data for function parameters (cheapest)

```solidity
function processData(uint[] calldata data) external {
    // data is in calldata - can't modify it
    uint[] memory tempData = new uint[](data.length);
    // tempData is in memory - temporary
}
```

### Mappings and Arrays

Mappings are super efficient for lookups but can't be iterated:

```solidity
mapping(address => uint256) public balances;

function updateBalance(address user, uint256 amount) public {
    balances[user] = amount;  // O(1) lookup
}
```

Arrays can be iterated but be careful with loops (gas limits!):

```solidity
address[] public users;

function addUser(address user) public {
    users.push(user);
}

function getUserCount() public view returns (uint) {
    return users.length;
}
```

### Events

Events are cheap ways to log data. They're stored in transaction logs, not in contract storage:

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);

function transfer(address to, uint256 amount) public {
    // ... transfer logic ...
    emit Transfer(msg.sender, to, amount);
}
```

The `indexed` keyword (max 3 per event) lets you filter events by those parameters.

### Inheritance

Solidity supports multiple inheritance:

```solidity
contract Animal {
    function makeSound() public virtual returns (string memory) {
        return "Some sound";
    }
}

contract Dog is Animal {
    function makeSound() public override returns (string memory) {
        return "Woof";
    }
}
```

### Interfaces and Abstract Contracts

Interfaces define function signatures without implementation:

```solidity
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract MyToken is IERC20 {
    // Must implement all interface functions
    function transfer(address to, uint256 amount) external returns (bool) {
        // Implementation here
    }
    
    function balanceOf(address account) external view returns (uint256) {
        // Implementation here
    }
}
```

## Most Used Patterns

### 1. Access Control Pattern

The most common pattern - restrict who can do what:

```solidity
contract AccessControl {
    address public owner;
    mapping(address => bool) public admins;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier onlyAdmin() {
        require(admins[msg.sender], "Not admin");
        _;
    }
    
    function addAdmin(address admin) public onlyOwner {
        admins[admin] = true;
    }
}
```

### 2. Withdrawal Pattern

Never send funds directly in the same transaction. Let users withdraw instead:

```solidity
contract Marketplace {
    mapping(address => uint256) public pendingWithdrawals;
    
    function sellItem(uint256 itemId) public {
        // ... logic ...
        uint256 proceeds = 100 ether;
        pendingWithdrawals[msg.sender] += proceeds;
    }
    
    function withdraw() public {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        
        pendingWithdrawals[msg.sender] = 0;  // Set to 0 BEFORE sending
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 3. Checks-Effects-Interactions Pattern

Guard against reentrancy attacks:

```solidity
function withdraw(uint256 amount) public {
    // 1. CHECKS
    require(balances[msg.sender] >= amount, "Insufficient balance");
    
    // 2. EFFECTS (update state)
    balances[msg.sender] -= amount;
    
    // 3. INTERACTIONS (external calls last)
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}
```

### 4. Factory Pattern

Contracts that create other contracts:

```solidity
contract TokenFactory {
    address[] public allTokens;
    
    event TokenCreated(address tokenAddress);
    
    function createToken(string memory name) public returns (address) {
        Token newToken = new Token(name, msg.sender);
        allTokens.push(address(newToken));
        emit TokenCreated(address(newToken));
        return address(newToken);
    }
}

contract Token {
    string public name;
    address public owner;
    
    constructor(string memory _name, address _owner) {
        name = _name;
        owner = _owner;
    }
}
```

### 5. Oracle Pattern

Getting off-chain data on-chain:

```solidity
contract PriceOracle {
    address public oracle;
    uint256 public price;
    
    event PriceUpdated(uint256 newPrice);
    
    modifier onlyOracle() {
        require(msg.sender == oracle, "Not oracle");
        _;
    }
    
    function updatePrice(uint256 newPrice) public onlyOracle {
        price = newPrice;
        emit PriceUpdated(newPrice);
    }
}
```

## Design Patterns

### Proxy Pattern (Upgradeable Contracts)

Since contracts are immutable, proxies let you "upgrade" by pointing to new implementation:

```solidity
contract Proxy {
    address public implementation;
    address public admin;
    
    constructor(address _implementation) {
        implementation = _implementation;
        admin = msg.sender;
    }
    
    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    function upgrade(address newImplementation) public {
        require(msg.sender == admin, "Not admin");
        implementation = newImplementation;
    }
}
```

### State Machine Pattern

For workflows with distinct states:

```solidity
contract Auction {
    enum State { Created, Bidding, Ended, Canceled }
    State public state;
    
    modifier inState(State _state) {
        require(state == _state, "Invalid state");
        _;
    }
    
    function startBidding() public inState(State.Created) {
        state = State.Bidding;
    }
    
    function placeBid() public payable inState(State.Bidding) {
        // Bidding logic
    }
    
    function endAuction() public inState(State.Bidding) {
        state = State.Ended;
    }
}
```

### Circuit Breaker (Emergency Stop)

Pause contract in case of emergency:

```solidity
contract EmergencyStop {
    bool public paused;
    address public owner;
    
    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function pause() public onlyOwner {
        paused = true;
    }
    
    function unpause() public onlyOwner {
        paused = false;
    }
    
    function normalFunction() public whenNotPaused {
        // Only works when not paused
    }
}
```

### Rate Limiting Pattern

Prevent spam or abuse:

```solidity
contract RateLimited {
    mapping(address => uint256) public lastCallTime;
    uint256 public cooldown = 1 hours;
    
    modifier rateLimit() {
        require(
            block.timestamp >= lastCallTime[msg.sender] + cooldown,
            "Too soon"
        );
        lastCallTime[msg.sender] = block.timestamp;
        _;
    }
    
    function limitedFunction() public rateLimit {
        // Can only be called once per hour per address
    }
}
```

## Quick Tips

1. **Always use the latest stable version** - Older versions have known bugs
2. **Use SafeMath for versions < 0.8.0** - Newer versions have built-in overflow protection
3. **Prefer `uint256` over smaller uints** - EVM works in 256-bit words anyway
4. **Use `calldata` for external function array parameters** - Saves gas
5. **Emit events for important state changes** - Makes your contract observable
6. **Test extensively** - Bugs in production can't be fixed!
7. **Keep functions simple** - Easier to audit and less gas
8. **Mind the gas costs** - Storage > Memory > Calldata

## Common Gotchas

- **Integer division truncates**: `5 / 2 = 2` (not 2.5)
- **Default values**: uninitialized `uint` is 0, `bool` is false, `address` is 0x0
- **Array length is not free**: `array.length` in a loop condition recalculates each iteration
- **Sending Ether can fail**: Always check the return value
- **Block timestamp manipulation**: Miners can manipulate timestamps slightly
- **tx.origin vs msg.sender**: Never use `tx.origin` for authorization

That should give you a solid foundation! The key to good Solidity is understanding that every line costs money, so efficiency and security are paramount.