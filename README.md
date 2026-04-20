# Ethernaut Writeups

My solutions and writeups for [Ethernaut](https://ethernaut.openzeppelin.com/) — a Web3/Solidity-based wargame focused on smart contract security.

## Progress

- [x] Level 0: Hello Ethernaut
- [x] Level 1: Fallback
- [ ] Level 2: Fallout
- [ ] Level 3: Coin Flip
- [ ] Level 4: Telephone
- [ ] ...

## Writeups

### Level 1: Fallback

**Vulnerability:** Improper access control on fallback function

**Exploit:**
1. Send a small amount of ETH directly to the contract
2. Trigger the fallback function with value > 0
3. Call `withdraw()` to drain the contract

**Key Takeaway:** Always validate msg.sender in fallback/receive functions when they modify critical state.

**Solution:** [Link to your solution file or gist]

---

*More writeups coming soon as I progress through the challenges.*
