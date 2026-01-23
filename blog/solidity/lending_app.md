@def title = "Your First Solidity Smart Contract: A Simple Lending Protocol"
@def published = "27 December 2025"
@def tags = ["solidity"]

# Your First Solidity Smart Contract: A Simple Lending Protocol

Let's build something practical but not overwhelming—a basic lending contract where users can deposit ETH and borrow against it.

## What You'll Need

- **Node.js** (v16+)
- **A code editor** (VS Code recommended)
- **MetaMask wallet** (for testing)
- **Some test ETH** (we'll get this for free)

## Step 1: Set Up Your Environment

Install Hardhat (the easiest Solidity development framework):

```bash
mkdir simple-lending
cd simple-lending
npm init -y
npm install --save-dev hardhat@^2.22.0 @nomicfoundation/hardhat-toolbox
npx hardhat init
```

> This installs Hardhat **locally in your project directory** (inside a `node_modules` folder). That's what `--save-dev` does—it's like downloading a copy of the tool just for this project. This is good because different projects can use different versions without conflicts. When you run `npx hardhat`, you're running the local copy, not a global one. Think of it like having your own toolbox in your garage instead of sharing one with the whole neighborhood!
>
> **Note**: We're using Hardhat v2 here because the toolbox plugins are most stable with it. If you accidentally installed v3 first, just delete the `node_modules` folder and `package-lock.json`, then run the install command above again.
>
> **Troubleshooting**: If you get "No Hardhat config file found", make sure you're in the right directory (the one with `package.json`) and that `npx hardhat init` completed successfully. It should create a `hardhat.config.js` file. If it didn't, try running the init command again and make sure to select "Create a JavaScript project" when prompted.

Choose "Create a JavaScript project" when prompted.

## Understanding Mappings and Events

> **Mappings** are like dictionaries or hash tables. Think of them as a giant lookup table where you can instantly find someone's balance by their address. When you do `deposits[yourAddress]`, it's like looking up your bank account balance—super fast and efficient. We need them because we can't loop through millions of users on the blockchain (too expensive!), but we can instantly check any specific user's data.
>
> **Events** are like receipts or notifications. When something important happens (like you depositing money), the contract emits an event that gets logged on the blockchain. Front-end apps and blockchain explorers listen for these events to show you your transaction history. Without events, your app would have no idea what happened—it's how the blockchain "talks" to the outside world. They're also super cheap compared to storing data!
>
> **Breaking down the event line:**
> - `event Deposited` - This is the name of the receipt/notification
> - `address` - This is a wallet address (like 0x123...), basically someone's account number on the blockchain
> - `indexed` - This makes the field searchable. It's like adding a tag so you can later filter for "show me all deposits from MY address" instead of everyone's deposits. You can have up to 3 indexed fields per event.
> - `uint256 amount` - This is just a number (how much was deposited). `uint256` means "unsigned integer up to 256 bits"—basically, a really big positive number.
>
> **What does `emit` do?**
> `emit` is like hitting "send" on that receipt. When you call `emit Deposited(msg.sender, msg.value)`, you're broadcasting "Hey! This person just deposited this much money!" to anyone listening. Your front-end app can then catch this and update the UI to show "Deposit successful!" or add it to your transaction history. 
>
> **Yes, the blockchain has a permanent record!** Events are stored in the blockchain's logs forever. Without emit, the function would still work internally (your balance updates), but there'd be no easy way to see your transaction history later. It's the difference between: (1) your bank updating your balance in their database vs. (2) your bank also giving you a statement showing every transaction. Both matter, but events give you the audit trail!
>
> **Exactly right!** You first *define* the event structure at the top (like designing a form: "name, address, amount"), then you *emit* it (like filling out and submitting that form). Every contract can define whatever events it wants—there's no universal standard. A lending contract might have `Deposited`, `Borrowed`, while a game contract might have `PlayerWon`, `LevelUp`. Apps reading the blockchain just look for the events they care about. Think of it like: the event definition is the template, emit is filling it out and sending it!
>
> **Absolutely!** Documentation is crucial. That's why many projects follow common patterns (like ERC-20 tokens all use `Transfer` events) and use tools like NatSpec comments in their code. When you build a front-end, you need to know "what events should I listen for?" Good contracts document this, or developers reverse-engineer it from the code. It's one reason why open-source contracts and clear naming (like `Deposited` vs `Event123`) are so important in web3!

## Step 2: Write Your Lending Contract

Create `contracts/SimpleLending.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleLending {
    mapping(address => uint256) public deposits;
    mapping(address => uint256) public borrowed;
    
    uint256 public constant COLLATERAL_RATIO = 150; // 150% collateralization
    
    event Deposited(address indexed user, uint256 amount);
    event Borrowed(address indexed user, uint256 amount);
    event Repaid(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    
    // Deposit ETH as collateral
    function deposit() external payable {
        require(msg.value > 0, "Must deposit something");
        deposits[msg.sender] += msg.value;
        emit Deposited(msg.sender, msg.value);
    }
    
    // Borrow against your collateral
    function borrow(uint256 amount) external {
        uint256 maxBorrow = (deposits[msg.sender] * 100) / COLLATERAL_RATIO;
        require(borrowed[msg.sender] + amount <= maxBorrow, "Insufficient collateral");
        
        borrowed[msg.sender] += amount;
        payable(msg.sender).transfer(amount);
        emit Borrowed(msg.sender, amount);
    }
    
    // Repay your loan
    function repay() external payable {
        require(borrowed[msg.sender] > 0, "No loan to repay");
        require(msg.value <= borrowed[msg.sender], "Repaying too much");
        
        borrowed[msg.sender] -= msg.value;
        emit Repaid(msg.sender, msg.value);
    }
    
    // Withdraw your collateral (after repaying)
    function withdraw(uint256 amount) external {
        require(borrowed[msg.sender] == 0, "Repay loan first");
        require(deposits[msg.sender] >= amount, "Insufficient balance");
        
        deposits[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
        emit Withdrawn(msg.sender, amount);
    }
    
    // Check how much you can borrow
    function availableToBorrow(address user) external view returns (uint256) {
        uint256 maxBorrow = (deposits[user] * 100) / COLLATERAL_RATIO;
        return maxBorrow - borrowed[user];
    }
}
```

## The Math Explained

The collateral ratio ensures loans are overcollateralized:

$$\text{Max Borrow} = \frac{\text{Deposit} \times 100}{150} = \text{Deposit} \times \frac{2}{3}$$

So if you deposit 1 ETH, you can borrow up to ~0.67 ETH.

## Step 3: Write a Deployment Script

Create `scripts/deploy.js`:

> This script can deploy to **either local or real blockchains**—it depends on which network you specify when running it. By default (Step 4), we'll deploy to a local test blockchain running on your computer. Later (Step 6), we'll use the same script to deploy to Sepolia, a real Ethereum test network. Think of it like having one "publish" button that can either save a draft locally or publish to the internet—same script, different destinations!

```javascript
async function main() {
  const SimpleLending = await ethers.getContractFactory("SimpleLending");
  const lending = await SimpleLending.deploy();
  await lending.waitForDeployment();
  
  console.log("SimpleLending deployed to:", await lending.getAddress());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

> **Breaking down the JavaScript:**
> - `async function main()` - "async" means this function does things that take time (like talking to the blockchain). It's like marking a function as "this might take a while, don't freeze up waiting for it."
> - `await` - This means "pause here and wait for this to finish before moving to the next line." Without it, JavaScript would just rush through without waiting for the blockchain to respond. Think of it like "await pizza.deliver()" before "eat(pizza)".
> - `ethers` - This is a JavaScript library (provided by Hardhat) that lets you talk to Ethereum. It's like a translator between your JavaScript code and the blockchain.
> - `getContractFactory("SimpleLending")` - This finds your Solidity contract and prepares it for deployment. Factory = "thing that makes things."
> - `deploy()` - This actually sends your contract to the blockchain and creates it there.
> - `waitForDeployment()` - Wait for the blockchain to confirm "yep, it's deployed!"
> - `.catch()` - If anything goes wrong, print the error instead of crashing silently.
>
> **What's actually happening under the hood?**
> The blockchain itself only understands low-level stuff—raw bytes, cryptographic signatures, and JSON-RPC calls (like "send this transaction," "get this data"). Ethers is doing the heavy lifting: it takes your nice `.getContractAt()` call, translates it into the right format, sends an HTTP request to a blockchain node, gets back raw data, and converts it into something JavaScript can use. It's like how your browser translates your clicks into HTTP requests—you don't see it, but there's a whole translation layer making everything work smoothly!
>
> **Exactly!** Step 3 adds your contract to the blockchain's global state. The blockchain is essentially a big state machine that everyone agrees on—when you deploy, you're saying "hey everyone, here's a new program that lives at this address, and here's its initial state (all balances start at zero)." From that point on, every transaction that calls your contract functions updates that shared state. It's permanent and replicated across thousands of nodes. That's why blockchain is so powerful—it's a state machine that no single person controls!

## Step 4: Test Locally

Start a local blockchain:

```bash
npx hardhat node
```

In another terminal, deploy:

```bash
npx hardhat run scripts/deploy.js --network localhost
```

## Step 5: Interact with Your Contract

Create `scripts/interact.js`:

> **Yes, exactly!** This is basically what your app does. When you build a real web app (a "dApp"), your front-end code (React, Vue, whatever) does the same thing—it uses ethers.js to connect to the contract, call functions like `deposit()` and `borrow()`, and listen for events. The only difference is that a real app has a nice UI with buttons and forms, while this script is just raw JavaScript for testing. Think of this as the "engine" that your app's UI will eventually wrap around!

```javascript
async function main() {
  const contractAddress = "YOUR_CONTRACT_ADDRESS_HERE";
  const lending = await ethers.getContractAt("SimpleLending", contractAddress);
  
  // Deposit 1 ETH
  const depositTx = await lending.deposit({ value: ethers.parseEther("1.0") });
  await depositTx.wait();
  console.log("Deposited 1 ETH");
  
  // Check how much we can borrow
  const available = await lending.availableToBorrow(await ethers.provider.getSigner().getAddress());
  console.log("Can borrow:", ethers.formatEther(available), "ETH");
  
  // Borrow 0.5 ETH
  const borrowTx = await lending.borrow(ethers.parseEther("0.5"));
  await borrowTx.wait();
  console.log("Borrowed 0.5 ETH");
}

main();
```

Run it:

```bash
npx hardhat run scripts/interact.js --network localhost
```

## Step 6: Deploy to a Testnet

Get test ETH from a faucet (search "Sepolia faucet").

Update `hardhat.config.js`:

```javascript
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.20",
  networks: {
    sepolia: {
      url: "https://sepolia.infura.io/v3/YOUR_INFURA_KEY",
      accounts: ["YOUR_PRIVATE_KEY"]
    }
  }
};
```

Deploy:

```bash
npx hardhat run scripts/deploy.js --network sepolia
```

## What You Just Built

You created a lending protocol where:
- Users deposit ETH as collateral
- They can borrow up to 66.7% of their deposit
- They must repay before withdrawing
- Everything is transparent on-chain

## Next Steps

- Add interest rates
- Implement liquidations
- Support multiple tokens
- Write comprehensive tests

**Pro tip**: Never put real private keys in your code. Use environment variables or hardware wallets for production!

---

## Can Contracts Be "Layered"?

> **Absolutely! This is a super common pattern called the "Factory Pattern."** You create one master contract (the "factory") that acts as the blueprint, and it can spawn unlimited instances (sub-games). Here's how it works:
>
> ```solidity
> // The blueprint - each game instance
> contract Game {
>     address public player1;
>     address public player2;
>     uint256 public bet;
>     // ... game logic
> }
> 
> // The factory - creates new games
> contract GameFactory {
>     Game[] public games;  // Track all games
>     
>     function createGame(uint256 betAmount) external returns (Game) {
>         Game newGame = new Game();  // Deploy a new Game contract!
>         games.push(newGame);
>         emit GameCreated(address(newGame), msg.sender);
>         return newGame;
>     }
> }
> ```
>
> When someone calls `createGame()`, it deploys a completely separate Game contract on the blockchain with its own address and state. Each sub-game is independent—they don't interfere with each other. Players can join whichever game they want by interacting with that specific game's address.
>
> **Real examples:** Uniswap does this! There's one factory contract, and it creates a separate liquidity pool contract for every trading pair (ETH/USDC, ETH/DAI, etc.). Same blueprint, unlimited instances. This is **exactly** how you'd build your game scenario!

## How to Host a dApp Web Interface?

> **Great question—you're right to call out the centralization!** Here's the reality: most dApps have a centralized front-end but decentralized backend (smart contracts). There are different levels of decentralization you can choose:
>
> **Option 1: Pragmatic (most common)**
> - Host your React/Next.js front-end on AWS, Vercel, or Netlify
> - Your app is just HTML/CSS/JavaScript that connects users' wallets to your smart contracts
> - If AWS goes down, your site is down, BUT the contracts still work! Users can interact directly via Etherscan or other interfaces
> - **Examples:** Most DeFi apps do this (Uniswap, Aave). It's centralized hosting, but the critical logic (money, state) is on-chain
>
> **Option 2: Decentralized hosting**
> - Use IPFS (InterPlanetary File System) to host your static site
> - Use ENS (Ethereum Name Service) for a readable domain like `mygame.eth`
> - Your site becomes truly unstoppable—no single server can take it down
> - **Trade-off:** Slower, more complex setup, and can't use server-side features
>
> **Option 3: Hybrid (best of both)**
> - Host on AWS for speed and convenience
> - Also publish to IPFS as a backup
> - If your main site goes down, users can access the IPFS version
>
> **The key insight:** The smart contracts ARE the app's core. Your front-end is just a convenient interface. Even if you host on AWS, users aren't trusting you with their money—they're trusting the blockchain. You're just making it easier to click buttons instead of calling contract functions manually!
>
> For a game dApp, Option 1 (AWS/Vercel) is totally fine to start. Once you're successful, add IPFS as a backup for extra decentralization points!