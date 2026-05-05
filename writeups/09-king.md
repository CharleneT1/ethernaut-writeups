# Level 9: King

**Difficulty:** Medium

**Vulnerability:** Unchecked transfer assumption — the contract assumes ETH transfers to the current king always succeed, which a malicious contract can exploit to permanently block throne changes.

## How it Works

The King contract lets anyone become king by sending ETH equal to or greater than the current prize. The `receive()` function first transfers the incoming ETH to the current king, then reassigns the king to the new caller. The problem is that if the transfer to the current king fails, the entire transaction reverts. This means if I become king using a contract that cannot accept ETH, anyone who tries to dethrone me will always revert, and I stay king forever.

## Exploit Steps

1. Check the current prize with `(await contract.prize()).toString()` in the browser console (it was 1000000000000000 wei / 0.001 ETH).
2. Deploy an attack contract that claims the throne in its constructor by calling the King contract with enough ETH.
3. The attack contract has no `receive()` or `fallback()` function, so it cannot accept ETH transfers.
4. Submit the instance. Anyone (including Ethernaut) who tries to reclaim the throne will revert because the transfer back to my contract fails.

## Exploit Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract AttackKing {
    address public target_address = 0x406f0BdD589c449f797e5B0e5447092d210D2519;

    constructor() payable {
        (bool success, ) = target_address.call{value: msg.value}("");
        require(success);
    }

    // No receive() or fallback() — this contract cannot accept ETH
}
```

Deploy with at least the current prize amount as the value (0.001 ETH in my case). After deployment, verify with `await contract._king()` — it should return the attack contract address.

## Key Takeaway

This is a denial of service via revert attack. The King contract blindly trusts that `transfer()` will always succeed, but `transfer()` reverts if the recipient is a contract with no payable fallback. The fix is to use a withdrawal pattern: instead of pushing ETH to the previous king immediately, record how much they are owed and let them pull it themselves. That way a broken recipient only blocks their own withdrawal, not the entire contract flow.

This bug also happened in the real world. The 2016 King of the Ether contract had the same design, but used `.send()` which failed silently instead of reverting. When a contract that couldn't receive ETH was king, the dethroned king's ETH was not refunded — it was just lost forever. The Ethernaut version uses `transfer()` which at least reverts cleanly, but the root cause is the same: pushing ETH to an address you don't control is dangerous.

## What to Look For

When reading any contract for the first time, scan for these red flags:

1. **External calls** — any `.call()`, `.transfer()`, `.send()`, or interaction with another contract address. These are the boundaries where things break.
2. **Order of operations** — does state update before or after the external call? Updating state after an external call opens the door to reentrancy.
3. **Assumptions about success** — does the contract check if the call succeeded, or just assume it worked? King never considers that `transfer()` might fail.
4. **Attacker controlled inputs** — can `msg.sender`, `msg.value`, or any parameter be manipulated in a way the contract doesn't expect?

For King, the red flag was number 3: ETH is pushed to an address the attacker controls, with no fallback plan if the transfer fails.
