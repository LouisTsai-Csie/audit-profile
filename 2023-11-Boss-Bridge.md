# First Flight #4: Boss Bridge - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. withdrawTokensToL1 AllowUnauthorized Token Transfers through Stolen (v, r, s) Value Pairs](#H-01)
    - ### [H-02. Vulnerabilities in L1BossBridge contract allow token accumulation through signature reuse](#H-02)

- ## Low Risk Findings
    - ### [L-01. Function lacks validation for duplicate token symbols, risking unexpected token addresses and operational errors](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #4

### Dates: Nov 9th, 2023 - Nov 15th, 2023

[See more contest details here](https://www.codehawks.com/contests/clomptuvr0001ie09bzfp4nqw)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>[H-01] withdrawTokensToL1 AllowUnauthorized Token Transfers through Stolen (v, r, s) Value Pairs            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L112

## Summary

The vulnerability allows a user to potentially take over the (v, r, s) value pair of another user, leading to unauthorized execution of the `withdrawTokensToL1` function and consequent token theft.

## Vulnerability Details

Exploiting this vulnerability involves stealing the (v, r, s) value pair from another user and triggering the `L1BossBridge::withdrawTokensToL1` function. The current implementation only validates the `signer` mapping value, allowing successful execution of `transferFrom` and resulting in token loss.

## Impact

The compromised (v, r, s) value pair poses a risk of fund loss, enabling arbitrary users to initiate unauthorized `L1BossBridge::withdrawTokensToL1` transactions.

## Tools Used

Manual Review

## Recommendations

To address this issue, consider implementing user address validation to ensure that the user initiating the `L1BossBridge::withdrawTokensToL1` transaction is the rightful owner of the tokens. This additional step will enhance security and prevent unauthorized token transfers.

## <a id='H-02'></a>[H-02] Vulnerabilities in `L1BossBridge` contract allow token accumulation through signature reuse            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L115

## Summary

The `L1BossBridge` contract lacks recording of used signatures in the `L1BossBridge::sendToL1` function, allowing potential misuse of the same message for multiple transactions.

## Vulnerability Details

In the `sendToL1` function, there is no restriction on the message, enabling a user to repeatedly trigger the function with the same message, leading to multiple token transfers from L2 to L1.

## Impact

Exploiting this vulnerability enables users to accumulate more tokens on the bridge than initially deposited by reusing the same message.

## Tools Used

Manual Review

## Recommendations

To mitigate this issue, consider implementing one or both of the following:

1. Update the `signers` mapping value to `false` when a user withdraws the token.
2. Include a nonce value in the signing message and validate the nonce to prevent signature replay attacks.		


# Low Risk Findings

## <a id='L-01'></a>[L-01] Function lacks validation for duplicate token symbols, risking unexpected token addresses and operational errors            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/TokenFactory.sol#L27

## Summary
The `deployToken` function in `TokenFactory.sol` lacks validation for duplicated token symbols.

## Vulnerability Details

In the `deployToken` function, there is no validation to check whether a symbol has already been registered. If a user registers token A with symbol S, and another user deploys token B with the same symbol S, the token mapping from symbol S to token address will be overridden. Subsequently, if the original user triggers `getTokenAddressFromSymbol` within the same contract, the return value may differ from their expectations.

## Impact

Users may receive unexpected token addresses, potentially causing issues such as minting a different token than intended (e.g., minting token B instead of token A).

## Tools Used

Manual Review

## Recommendations

To mitigate this issue, it is advised to validate the return value by checking whether the symbol has been registered before deploying a new token. Consider adding a validation statement at the beginning of the token deployment, such as `require(s_tokenToAddress[symbol] != 0, "Symbol already registered")`. This validation step will prevent the override of token symbols and ensure users receive the expected return values.


