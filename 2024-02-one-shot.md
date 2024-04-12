# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The internal _battle function contains an unsafe random function that could be exploited by a malicious user](#H-01)
- ## Medium Risk Findings
    - ### [M-01. NFT in `MintRapper` is initialized after the token distribution, resulting in a reentrancy vulnerability in the `goOnStageOrBattle` function](#M-01)
- ## Low Risk Findings
    - ### [L-01. battlesWon variable unused, not update after the result of goOnStageOrBattle operation when defender is not null address](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. The internal _battle function contains an unsafe random function that could be exploited by a malicious user            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62

## Summary

The provided code includes a vulnerability related to the usage of the `keccak256` hashing algorithm for generating randomness. The `random` variable, which influences the outcome of the battle, relies on a deterministic method that uses the block timestamp, previous random value, and the sender's address. This approach is deemed unsafe due to the predictability of the generated values.

## Vulnerability Details

The vulnerable code segment is as follows in `_battle` function in `RapBattle` file:

```js
uint256 random =
    uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill; // @audit-issue pseudo random
```

Here, the `keccak256` hash function is used to generate a pseudo-random number for determining the outcome of a battle. However, relying on block information and sender's address for randomness introduces predictability, making the system susceptible to manipulation.

## Impact

The impact of using an insecure randomness generation method in the context of a battle function is significant. It may allow malicious actors to exploit the predictability of the generated random values, potentially gaining an unfair advantage in battles.

## Tools Used
- Manual Review

## Recommendations

To enhance the security and unpredictability of randomness in the `_battle` function, it is recommended to replace the current method with a more robust and secure randomness generation solution. One such solution is the Chainlink Verifiable Random Function (VRF), a decentralized oracle network that provides secure and tamper-resistant randomness.

By integrating Chainlink VRF, the application can ensure a more secure and unbiased source of randomness, reducing the risk of manipulation and providing a fairer gaming experience.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. NFT in `MintRapper` is initialized after the token distribution, resulting in a reentrancy vulnerability in the `goOnStageOrBattle` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/OneShot.sol#L29

## Summary

The mintRapper function in the OneShot file has a vulnerability where the ERC-721 token is distributed before updating the `rapperStats`. This sequence allows a potential reentrancy attack, as a malicious contract can mint the NFT, triggered the `onERC721Received` function in their contract, and then exploit the NFT to initiate a rapper battle using the `goOnStageOrBattle` function in the `rapperBattle` file. This results in the NFT having a `rapperStats` value of 65 instead of the expected initialized value of 50, potentially impacting the outcome of battles and allowing users to gain an extra 15 points without proper staking.

## Vulnerability Details

The vulnerability arises from the order of execution in the `mintRapper` function, allowing a reentrancy attack. A malicious contract can mint an NFT, receive it through the `onERC721Received` function, and then immediately trigger a rapper battle, taking advantage of the uninitialized `rapperStats` value. The impact of this vulnerability is verified through the `MockERC721Receiver` contract and the `testReentrancyIssue` test in `RapBattleTest`, displaying a final skill value of 65 instead of the expected initialized value of 50.

Adding `MockERC721Receiver` in `OneShotTest.t.sol`:

```js
contract MockERC721Receiver is Test {
    OneShot oneShot;
    Credibility cred;
    RapBattle rapBattle;

    event Battle(address indexed challenger, uint256 tokenId, address indexed winner);

    constructor(OneShot _oneShot, Credibility _cred, RapBattle _rapBattle) payable {
        oneShot = _oneShot;
        cred = _cred;
        rapBattle = _rapBattle;
    }

    function mintRapper() public {
        oneShot.mintRapper();
    }

    function onERC721Received(address, address, uint256 tokenId, bytes calldata) external returns (bytes4) {
        oneShot.approve(address(rapBattle), tokenId);
        uint256 finalSkill = rapBattle.getRapperSkill(tokenId);
        console.log(finalSkill);

        rapBattle.goOnStageOrBattle(tokenId, cred.balanceOf(address(this))); 
        return bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"));
    }
}
```

Adding the test in `RapBattleTest` test contract

```js
function testReentrancyIssue() public mintRapper {
        address attacker = makeAddr("attacker");
        
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, 0);

        vm.startPrank(attacker);
        MockERC721Receiver receiver = new MockERC721Receiver(oneShot, cred, rapBattle);
        receiver.mintRapper();
        vm.stopPrank();
    }
```
In the console, it display the final skill as the following:

```md

[PASS] testReentrancyIssue() (gas: 1242062)
Logs:
  65

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.64ms
```

The result is 65 instead of 50.

## Impact

This vulnerability enables users to gain an additional 15 points without proper staking, influencing the results of battles. It deviates from the original protocol design, potentially compromising the fairness and integrity of the system.

## Tools Used

Manual Review

## Recommendations

To mitigate the reentrancy attack and ensure proper initialization, the following recommendations are suggested:

1. **Reentrancy Guard:** Implement a reentrancy guard in relevant functions to introduce a mutex lock, protecting shared state variables from reentrancy attacks.

2. **CEI Pattern (Check-Effect-Interaction):** Reorder the statements in the mintRapper function to follow the CEI pattern. Ensure that the rapperStats are initialized before any battles are initiated, preventing the exploitation of uninitialized values in reentrancy attacks.

# Low Risk Findings

## <a id='L-01'></a>L-01. battlesWon variable unused, not update after the result of goOnStageOrBattle operation when defender is not null address            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L54

## Summary
The One Shot documentation does not detail the battlesWon variable within the RapperStats structure in the IOneShot interface. Given the variable's name, we infer that it is meant to represent the number of battles won by the rapper. However, the battlesWon variable remains unaltered throughout the entire codebase, rendering it unused.

## Vulnerability Details
Assuming two rappers, Bob and Alice, engage in a battle, the battlesWon variable for one of them should increase from zero to a specific number, as outlined in the white paper. To verify this behavior, the testBattlesWonIncreaseOrNot test can be added to RapBattleTest. Upon running the command `forge test --match-test testBattlesWonIncreaseOrNot`, the test fails, indicating that the battlesWon variables are not updating as expected.

```js
function testBattlesWonIncreaseOrNot() public {
        address Bob = makeAddr("Bob"); 
        IOneShot.RapperStats memory bobStats;
        vm.prank(Bob);
        oneShot.mintRapper(); // tokenId: 0

        address Alice = makeAddr("Alice");
        IOneShot.RapperStats memory aliceStats;
        vm.prank(Alice);
        oneShot.mintRapper(); // tokenId: 1

        vm.startPrank(Bob);
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, 0);  // Bob will be the defender
        vm.stopPrank();

        vm.startPrank(Alice);
        oneShot.approve(address(rapBattle), 1);
        rapBattle.goOnStageOrBattle(1, 0); // Alice will battle with Bob
        vm.stopPrank();

        bobStats = oneShot.getRapperStats(0); 
        aliceStats = oneShot.getRapperStats(1);
        
        // We do not know the result of the battle, however, the battlesWon variable should increase in rapper status of Bob or Alice
        assertTrue(bobStats.battlesWon==1 || aliceStats.battlesWon==1);
    }
```

## Impact
The battlesWon variable, although present in the code, remains unused. This lack of utilization may lead to confusion for users seeking clarity on its intended purpose and meaning within the context of the codebase.

## Tools Used
Manual Review

## Recommendations
Either remove the battlesWon variable or ensure its update within the _battle function.

```diff
function _battle(uint256 _tokenId, uint256 _credBet) internal {
...

defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
+        uint256 tokenId = random < defenderRapperSkill? defenderTokenId: _tokenId;
        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
+        IOneShot.RapperStats memory rapperStats = oneShotNft.getRapperStats(tokenId);
+        rapperStats.battlesWon += 1;
+       oneShotNft.updateRapperStats(
+           tokenId,
+            rapperStats.weakKnees,
+            rapperStats.heavyArms,
+            rapperStats.spaghettiSweater,
+            rapperStats.calmAndReady,
+            rapperStats.battlesWon
+        );
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId); // @audit-issue not update battlesWon
}
```


