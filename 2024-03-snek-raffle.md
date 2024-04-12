# First Flight #11: Snek-Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Potential DoS Due to Revert in fulfillRandomWords Function](#H-01)
- ## Medium Risk Findings
    - ### [M-01. The fulfillRandomWords function not compare the random number with the rarity value, resulting in all NFTs having the same rarity](#M-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #11

### Dates: Mar 7th, 2024 - Mar 14th, 2024

[See more contest details here](https://www.codehawks.com/contests/cltd8lz860001y058i5ug72ko)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Potential DoS Due to Revert in fulfillRandomWords Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-snek-raffle/blob/6b12b719f5d8587f9bbbf3b029954f23a5a38704/contracts/snek_raffle.vy#L147

## Summary

The fulfillRandomWords function might revert due to multiple reasons. According to Chainlink VRF documentation, a revert in fulfillRandomWords prevents the VRF service from attempting a second call. 

## Vulnerability Details

Several scenarios within the fulfillRandomWords function can lead to transaction reverting:

1. If the **`recent_winner`** is a contract account but does not implement the **`onERC721Received`** function or returns an incorrect value.
2. If the **`recent_winner`** is a contract account without receive or fallback functions, resulting in an inability to receive ether and leading to a revert.
3. If the **`recent_winner`** is a contract account, insufficient gas forwarded may lead to an out-of-gas problem during execution.

Any of these scenarios will result in a transaction revert, preventing the VRF service from calling fulfillRandomWords again, thus preventing the update of the **`raffle_state`.**

## Impact

The inability to update the **`raffle_state`** variable due to transaction reverting could prevent the system from going to the next round, potentially disrupting the functionality of the contract.

## Tools Used

Manual Review

## Recommendations

Ensure that the fulfillRandomWords function is solely responsible for storing the random variable and does not contain any logic that could lead to reverting transactions. It should be designed to never revert, thereby preventing potential DoS attacks and ensuring the smooth operation of the system.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. The fulfillRandomWords function not compare the random number with the rarity value, resulting in all NFTs having the same rarity            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-snek-raffle/blob/6b12b719f5d8587f9bbbf3b029954f23a5a38704/contracts/snek_raffle.vy#L154

## Summary
The fulfillRandomWords function does not compare the rarity value, resulting in equal rarity for all NFTs, which contradicts the documented rarity distribution.

## Vulnerability Details
Within the fulfillRandomWords function, rarity is solely determined by the result of a random number modulo 3 operation. Consequently, the chances of obtaining common, rare, and legendary NFTs are identical. This deviates from the documented rarity distribution, which specifies:

Common: 70%
Rare: 25%
Legendary: 5%
## Impact
This oversight allows anyone to obtain a legendary NFT with a 33% chance instead of the intended 5% probability. Similarly, users have a 33% chance of receiving a rare NFT instead of the intended 25% probability.

## Tools Used
Manual Review

## Recommendations
Before assigning rarity, ensure that the calculated rarity is compared to 100.
```diff
@internal
def fulfillRandomWords(request_id: uint256, random_words: uint256[MAX_ARRAY_SIZE]):
    index_of_winner: uint256 = random_words[0] % len(self.players)
    recent_winner: address = self.players[index_of_winner]
    self.recent_winner = recent_winner
    self.players = []
    self.raffle_state = RaffleState.OPEN
    self.last_timestamp = block.timestamp
-   rarity: uint256 = random_words[0] % 3 # @audit-issue 
+   result: uint256 = random_words[0] % 100
+   rarity: uint256 = COMMON
+   if(result>70) rarity = RARE
+   if(result>95) rarity = LEGEND
    self.tokenIdToRarity[ERC721._total_supply()] = rarity 
    log WinnerPicked(recent_winner)
    ERC721._mint(recent_winner, ERC721._total_supply()) 
    send(recent_winner, self.balance)
```
