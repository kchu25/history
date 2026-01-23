@def title = "Solidity Functions & Mappings: A Practical Guide"
@def published = "28 December 2025"
@def tags = ["solidity"]

# Solidity Functions & Mappings: A Practical Guide

## Functions

### Basic Syntax

```solidity
function functionName(paramType paramName) visibility mutability returns (returnType) {
    // function body
}
```

### Visibility Modifiers

- **`public`** - Anyone can call (internally or externally)
- **`external`** - Only callable from outside the contract
- **`internal`** - Only callable within contract and derived contracts
- **`private`** - Only callable within this contract

### State Mutability Modifiers

- **`view`** - Reads state but doesn't modify it
- **`pure`** - Doesn't read or modify state
- **`payable`** - Can receive Ether
- *(no modifier)* - Can read and write state

### The Memory Question

**Why declare `memory` for function arguments?**

```solidity
function process(string memory name) public { }
```

Solidity has different data locations:
- **`storage`** - Persistent, stored on blockchain (expensive)
- **`memory`** - Temporary, exists during function execution (cheaper)
- **`calldata`** - Like memory, but immutable and only for external function parameters (cheapest)

For **reference types** (arrays, structs, strings), you *must* specify location:

```solidity
function good(string memory str) public { }     // ✓ Required
function bad(string str) public { }             // ✗ Won't compile
function better(string calldata str) external { } // ✓ More gas efficient
```

For **value types** (uint, bool, address), no location needed—they're always copied:

```solidity
function example(uint256 number) public { }  // No memory needed
```

**Rule of thumb**: Use `calldata` for external function parameters (saves gas), `memory` for internal processing.

### Does `view` Always Follow `returns`?

**No!** The order is flexible:

```solidity
// Both are valid
function getBalance() public view returns (uint256) { }
function getBalance() view public returns (uint256) { }

// Common patterns
function calculate() external pure returns (uint256) { }
function getData() internal view returns (string memory) { }
```

**Convention**: Most devs write `visibility` then `mutability` then `returns`, but Solidity doesn't enforce this.

> **Do you need `returns(type)` when using `view`?**
> 
> **Nope!** `view` just means "I'm only reading, not writing." Whether you actually return something is separate.
> 
> ```solidity
> // view WITH returns - reading and giving back data
> function getBalance() public view returns (uint256) {
>     return balance;
> }
> 
> // view WITHOUT returns - just reading to check something
> function checkBalance() public view {
>     require(balance > 100, "Balance too low");
>     // Does something but doesn't return anything
> }
> ```
> 
> Think of it like this:
> - `view` = "I promise I'm just looking, not touching"
> - `returns(type)` = "I'm giving you something back"
> 
> You can look without giving something back. You can also give something back while modifying things (no `view`). They're independent choices!

### Return Values

```solidity
// Named returns (automatically creates variable)
function divide(uint256 a, uint256 b) public pure returns (uint256 result) {
    result = a / b;  // Implicitly returned
}

// Explicit returns
function divide(uint256 a, uint256 b) public pure returns (uint256) {
    return a / b;
}

// Multiple returns
function getStats() public view returns (uint256 balance, address owner) {
    balance = address(this).balance;
    owner = msg.sender;
}
```

## Mappings

### Basic Syntax

```solidity
mapping(keyType => valueType) visibility variableName;
```

### Are Mappings State Variables?

**Yes!** Mappings are *always* state variables stored in storage:

```solidity
contract MyContract {
    mapping(address => uint256) public balances;  // State variable in storage
    
    function localMapping() public {
        mapping(address => uint256) temp;  // ✗ Not allowed! Can't be local
    }
}
```

**Key points**:
- Mappings exist only in `storage`
- You can't have mappings in `memory`
- They're not iterable (no way to list all keys)
- All possible keys exist, returning default value (0, false, etc.) if not set

### Basic Usage

```solidity
contract SimpleBank {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function getBalance() public view returns (uint256) {
        return balances[msg.sender];  // Returns 0 if never set
    }
}
```

### Nested Mappings

**Syntax**: `mapping(keyType1 => mapping(keyType2 => valueType))`

Think of it as a 2D lookup table or a "mapping of mappings."

```solidity
mapping(address => mapping(address => uint256)) public allowances;
//      owner    =>        spender   => amount
```

### Use Cases for Nested Mappings

#### 1. **Token Allowances (ERC-20 Standard)**

```solidity
// Owner approves spender to use their tokens
mapping(address => mapping(address => uint256)) public allowances;

function approve(address spender, uint256 amount) public {
    allowances[msg.sender][spender] = amount;
}

function transferFrom(address owner, address recipient, uint256 amount) public {
    require(allowances[owner][msg.sender] >= amount, "Not approved");
    allowances[owner][msg.sender] -= amount;
    // ... transfer logic
}
```

**Translation**: "How much can `spender` spend on behalf of `owner`?"

#### 2. **Multi-Signature Approvals**

```solidity
mapping(uint256 => mapping(address => bool)) public hasApproved;
//      txId    =>        approver => approved?

function approveTransaction(uint256 txId) public {
    hasApproved[txId][msg.sender] = true;
}
```

**Translation**: "Has `address` approved transaction `txId`?"

#### 3. **Access Control Systems**

```solidity
mapping(address => mapping(bytes32 => bool)) public roles;
//      account =>        role    => hasRole

function grantRole(address account, bytes32 role) public {
    roles[account][role] = true;
}

function hasRole(address account, bytes32 role) public view returns (bool) {
    return roles[account][role];
}
```

**Translation**: "Does `account` have `role`?"

#### 4. **Gaming/NFT Metadata**

```solidity
mapping(uint256 => mapping(string => uint256)) public nftAttributes;
//      tokenId =>        attribute => value

nftAttributes[1]["strength"] = 85;
nftAttributes[1]["speed"] = 60;
```

**Translation**: "What's the `attribute` value for `tokenId`?"

### Triple Nested Mappings

**Yes, they exist!** Though less common:

```solidity
mapping(address => mapping(uint256 => mapping(address => bool))) public permissions;
//      owner   =>        tokenId =>        operator => canOperate

// Use case: NFT operator approvals per token
function approveForToken(uint256 tokenId, address operator) public {
    permissions[msg.sender][tokenId][operator] = true;
}
```

### Important Mapping Limitations

```solidity
mapping(address => uint256) public balances;

// ✗ Can't iterate
for (address user in balances) { }  // Not possible

// ✗ Can't get length
uint256 count = balances.length;  // Not possible

// ✗ Can't delete entire mapping
delete balances;  // Not possible

// ✓ Can delete individual keys
delete balances[someAddress];  // Sets to 0
```

**Workaround**: Keep a separate array of keys if you need iteration:

```solidity
mapping(address => uint256) public balances;
address[] public users;

function addBalance(address user, uint256 amount) public {
    if (balances[user] == 0) {
        users.push(user);  // Track new users
    }
    balances[user] += amount;
}
```

## Quick Reference

### Function Patterns

```solidity
// Read-only, external
function getName() external view returns (string memory) { }

// State changing, public
function setName(string calldata newName) public { }

// Payment receiving
function deposit() external payable { }

// Pure calculation
function add(uint256 a, uint256 b) external pure returns (uint256) { }
```

### Mapping Patterns

```solidity
// Simple lookup
mapping(address => uint256) public balance;

// Two-dimensional lookup
mapping(address => mapping(address => uint256)) public allowance;

// Complex value type
mapping(address => User) public users;

struct User {
    string name;
    uint256 balance;
    bool isActive;
}
```