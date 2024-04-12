# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Lack of locking token during staking period leads to disproportionate Reward](#H-01)
    - ### [H-02. Lack of Validation Leading to Potential Fund Loss in Vault](#H-02)
- ## Medium Risk Findings
    - ### [M-01. ILoveToken does not follow ERC-20 Specification, leading to failure of operation when interacting with the protocol](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Lack of locking token during staking period leads to disproportionate Reward            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L73

## Summary

The `claimReward` function within the Staking module has a vulnerability. If a user claims a reward for the first time, the `lastClaim` mapping value is set to the timestamp of the soulmate matching. However, if the user has not staked any tokens before claiming rewards, there is a possibility to claim a disproportionate amount of reward.

## Vulnerability Details

If the `idToCreationTimestamp` mapping value for a couple is denoted as x, and after y days, where y is an extremely large value, the user can stake love tokens and immediately claim the reward. The `lastClaim` value is configured to the `idToCreationTimestamp` mapping value. Consequently, the `amountToClaim` will be extremely large, even though the user did not stake tokens for the expected duration.

## Impact

This vulnerability allows users to claim a disproportionate amount of tokens, potentially exploiting the system.

## Tools Used

Manual Review

## Recommendations

To mitigate this risk, consider implementing a mechanism to lock the tokens during the reward period. This would prevent users from exploiting the system by claiming rewards without staking tokens for the appropriate duration.
## <a id='H-02'></a>H-02. Lack of Validation Leading to Potential Fund Loss in Vault            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L56

## Summary

The `claimAirdrop` function lacks validation to determine whether the user is reunited or not. If a user who is not reunited attempts to claim the reward, the `idToCreationTimestamp` mapping value remains the default, resulting in a calculation error. This miscalculation could lead to a significant value for `numberOfDaysInCouple`, potentially exceeding the balance of the vault contract and causing the withdrawal of all funds.

## Vulnerability Details

If a user is not reunited, either because they minted a soulmate NFT but did not find a match or did not mint an NFT at all, attempting to claim the reward in the Airdrop leads to an issue. The `idToCreationTimestamp` mapping value is not updated, remaining at the default zero value. The calculation of `numberOfDaysInCouple` becomes the pure result of `block.timestamp / daysInSecond`, representing the days since the initial block creation. Consequently, the `tokenAmountToDistribute` becomes excessively large, potentially causing the user to drain all funds in the Airdrop vault contract.

## Impact

A test scenario exemplifies this potential issue:

```solidity
function test_AllFundsDrained() public {
    vm.warp(10_000 days + 1 seconds);

    vm.prank(soulmate1);
    airdropContract.claim();

    assertEq(loveToken.balanceOf(soulmate1), 10_000 ether);
}

```

In this case, soulmate1, who did not mint any tokens, still receives 10,000 love tokens as a reward, deviating from the intended specification.

## Tools Used

Manual Review

## Recommendations

To address this vulnerability, it is recommended to add validation checks to determine whether the user is reunited or not. Additionally, consider validating whether `idToCreationTimestamp` is at its default value before proceeding with the reward claim.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. ILoveToken does not follow ERC-20 Specification, leading to failure of operation when interacting with the protocol            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/interface/ILoveToken.sol#L4

## Summary

While LoveToken adheres to the ERC-20 token standard, its interface, ILoveToken, is inconsistent with the ERC-20 token interface. This discrepancy may lead to user confusion during interactions with the protocol.

## Vulnerability Details

The LoveToken contract utilizes the solmate template for ERC-20 tokens. In a standard ERC-20 token, the `transfer`, `transferFrom`, and `approve` functions are expected to return a boolean to indicate the success of the operation.

```solidity
function transfer(address _to, uint256 _value) public returns (bool success)
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
function approve(address _spender, uint256 _value) public returns (bool success)

```

However, in ILoveToken, the interface for LoveToken, there is a slight difference:

```solidity
function transfer(address to, uint256 amount) external;
function transferFrom(address from, address to, uint256 amount) external;
function approve(address to, uint256 amount) external;
```

It does not include a return boolean value. Interacting with these functions using a contract compiled with Solidity > 0.4.22 will result in execution failure due to the missing return value.

## Impact

If the protocol is deployed and users claim Love Tokens, attempting to interact with the contract using ILoveToken as a reference may prove unsuccessful.

## Tools Used

Slither

## Recommendations

To solve this issue, update the ILoveToken interface to align with the ERC-20 token standard by adding return values to the `transfer`, `transferFrom`, and `approve` functions.




