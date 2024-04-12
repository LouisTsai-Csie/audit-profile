# First Flight #5: Santa's List - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Arbitrary user can call `buyPresent` on behalf of token owner without validation](#H-01)
    - ### [H-02. Missing check for uninitialized status enum value leads to arbitrary minting](#H-02)
    - ### [H-03. Anyone can trigger`checkList` due to the missging validation](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Mismatch amount in `PURCHASED_PRESENT_COST` and burn amount.](#M-01)
- ## Low Risk Findings
    - ### [L-01. Solidity version 0.8.22 might not work on Arbitrum](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 1
   - Low: 1


# High Risk Findings
## <a id='H-01'></a>[H-01] Arbitrary user can call `buyPresent` on behalf of token owner without validation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L172

## Summary

Anyone can call `buyPresent` even though they do not have santaTokens. There is mismatch between the protocol document and the implementation.

## Vulnerability Details

In the documentation, it points out that the `buyPresent` function is only callable by anyone with `SantaToken`. However, there is no balance check in the function. Consider there are two users, A and B. A has `santaToken` and approve to the `santasList` contract,  he/she is going to buy present for friends. B sees the chance and trigger `buyPresent` right before A executes the function. B can simply pass the address of A in the `presentReceiver` parameter and the token of A will be burnt and B will receive the NFT.

## Impact

Unintended behavior for `buyPresent` function might lead to the loss of token for token owner.

## Tools Used

Manual Review

## Recommendations

1. Check whether `msg.sender` in `buyPresnet` function has enough balance.
2. burn the token of `msg.sender`, not the `presentReceiver`.
3. allocating NFT for `presentReceiver`, not `msg.sender`.
## <a id='H-02'></a>[H-02] Missing check for uninitialized status enum value leads to arbitrary minting            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L70C12-L70C12

## Summary

When the `SantasList::s_theListCheckedOnce` and `SantasList::s_theListCheckedTwice` are not initialized, the default value of the enum will be `NICE`. It will bypass the validation rules in `SantasList::collectPresent` and earn the NFT. 

## Vulnerability Details

Since the default value of enum type will be its first value in the enum structure. In this case, all the default mapping value will be `NICE`. User bypasses the validation rules in `SantasList::collectPresent` since `s_theListCheckedOnce[msg.sender] == Status.NICE` and `s_theListCheckedTwice[msg.sender] == Status.NICE`. Users are able to mint and receive the NFT even though they should be marked naughty later.

## Impact

Unintended amount of NFTs will be allocated.

## Tools Used

Manual Review

## Recommendations

Add a new type in the first element in `Status`, such as `UNINITIALIZED`.
## <a id='H-03'></a>[H-03] Anyone can trigger `SantasList::checkList` due to the missging validation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L121

## Summary

Arbitrary user can update the `s_theListCheckedOnce` array due to the missing access control. In the natspec of the protocol, it points out that there should be `onlySanta` modifier for `SantasList::checkList` function.

## Vulnerability Details

In the `SantasList::checkList` function where the `s_theListCheckedOnce` array is updated, there is no access control and anyone can update the value. An user can bypass the first validation easily.
 
## Impact

The first validation is useless since anyone can update its value.

## Tools Used

Manual Review

## Recommendations

Add `onlySanta` modifier in the `SantasList::checkList` function
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Mismatch amount in `PURCHASED_PRESENT_COST` and burn amount.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L173C3-L173C3

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L88

## Summary

The protocol documentation outline the price to buy the present is `2e18` but the `buyPresent` funciton only charge `1e18`.

## Vulnerability Details

In the protocol documentation, it states that:

`The cost of santa tokens for naughty people to buy presents` is `2e18`.

However, in the `buyPresent` function it only burn `1e18` for each operation, that is, users pay less than they are required.

## Impact

Protocol lose funds due to the incorrect calculation of the cost buying the gift.

## Tools Used

Manual review

## Recommendations

Update the unit of burning tokens to `2e18`.

# Low Risk Findings

## <a id='L-01'></a>L-01. Solidity version 0.8.22 might not work on Arbitrum            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantaToken.sol#L2

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L47

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/TokenUri.sol#L2

## Summary

There is important update after solidity version 0.8.20, the push0 opcode is introduced which might not work in other EVM compatible chains.

## Vulnerability Details

EIP-3855 introduces a breaking change in solidity version 0.8.20. And the push0 opcode is utilized which might not be updated on layer 2 chains such as Arbitrum or others. 

## Impact

It will lead to unintended behavior and the failure of deployment.

## Tools Used

Manual Review

## Recommendations

Use solidity 0.8.19 or earlier version instead.


