# First Flight #3: Thunder Loan - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Mismatch storage layout for upgradeable logic contract](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #3

### Dates: Nov 1st, 2023 - Nov 8th, 2023

[See more contest details here](https://www.codehawks.com/contests/clocopz26004rkx08q1n61wnz)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>[H-01] Mismatch storage layout for upgradeable logic contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/upgradedProtocol/ThunderLoanUpgraded.sol#L96

## Summary

In proxy pattern, the upgraded storage layout should follow the previous one. If not, it will lead to storage collision and several sensitive data will be override.

## Vulnerability Details

In the `ThunderLoanUpgraded.sol` contract, the storage layout is the following format:

```
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;
uint256 private s_flashLoanFee;
mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

which is not coherent to the storage layout of `ThunderLoan.sol`

```
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;
uint256 private s_feePrecision;
uint256 private s_flashLoanFee;
mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

If logic contract is updated, the storage slot that stores the `s_flashLoanFee` will be overrided by `s_currentlyFlashLoaning`.

## Impact

The value of `s_flashLoanFee` will be incorrect and lead to wrong calculation.

## Tools Used

Manual Review

## Recommendations

Coherent to the previous storage layout and avoid adding or deleting new state variables.


		





