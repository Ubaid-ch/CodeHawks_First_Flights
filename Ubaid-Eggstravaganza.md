# Eggstravaganza - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Random number generation](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #37

### Dates: Apr 3rd, 2025 - Apr 10th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-04-eggstravaganza)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Random number generation            



## Summary

The function `EggHuntGame::searchForEgg` attempts to implement randomness to determine whether a player finds an egg. However, it uses **insecure pseudo-random number generation** based on on-chain variables like `block.timestamp`, `block.prevrandao`, `msg.sender`, and `eggCounter`. This technique is **predictable and manipulable**, leading to potential **game manipulation or unfair advantage**.

## Vulnerability Details

```Solidity
uint256 random = uint256(
  keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender, eggCounter))) % 100;
```

The use of `block.timestamp` and `block.prevrandao` is **not secure for randomness** in Ethereum.

&#x20;

* These values can be **influenced by miners/validators** within acceptable protocol limits.
* Since `msg.sender` and `eggCounter` are known or controllable by the user, the overall entropy of the hash is **low**.
* This allows a user to **repeatedly call the function or simulate outcomes off-chain** to eventually get a favorable random number below `eggFindThreshold`.

## Impact

Impact: High, Players can **predict or influence egg discoveries**, violating fairness.



## Recommendations

Use a **verifiable randomness source** such as **Chainlink VRF** for secure and tamper-proof random number generation.

    





