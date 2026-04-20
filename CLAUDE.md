# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a writeup repository for [Ethernaut](https://ethernaut.openzeppelin.com/) — an OpenZeppelin Web3/Solidity wargame focused on smart contract security. Each level involves exploiting a vulnerable smart contract.

## Writeup Structure

Writeups live in the README for now. As the repo grows, move individual levels to `writeups/NN-challenge-name.md` and link from the README progress table.

Each writeup should follow this template:

```markdown
## Level N: Challenge Name

**Difficulty:** Easy / Medium / Hard

**Vulnerability:** One-line description of the core issue

### How it Works
Brief explanation of what the contract does and where the flaw is.

### Exploit Steps
1. Step one
2. Step two

### Exploit Code
\`\`\`solidity
// Attack contract or browser console snippet
\`\`\`

### Key Takeaway
The class of bug and how to prevent it (explain *why* the fix works, not just what it is).
```

## Writeup Guidelines

- Include browser console commands when the exploit doesn't need a separate contract — they're often the clearest demonstration.
- For harder levels, a short "What I tried first" section adds context and honesty.
- Key takeaways should explain *why* the fix works (e.g. "use `transfer()` instead of `call()` because it limits gas to 2300, preventing reentrance").
- Note the vulnerability category (reentrancy, access control, integer overflow, etc.) — useful for cross-referencing patterns.
