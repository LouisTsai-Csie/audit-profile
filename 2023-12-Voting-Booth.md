# First Flight #6: Voting Booth - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Lack of withdraw function leads to lock of ethers.](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Incompatibility of Solidity 0.8.23 with Arbitrum: Deployment Failures Due to Unsupported PUSH0 Opcode](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #6

### Dates: Dec 15th, 2023 - Dec 22nd, 2023

[See more contest details here](https://www.codehawks.com/contests/clq5cx9x60001kd8vrc01dirq)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 0


# High Risk Findings
## <a id='H-01'></a>[H-01] Lack of withdraw function leads to lock of ethers        

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L120

## Summary

The lack of `VotingBooth::withdraw` function locks the token in the contract forever.

## Vulnerability Details

Consider the following scenario, there is 5 allowed voters and the threshold of voters required to trigger the reward distribution is 3. Assume the reward is 1 ether and two of the voter votes for the proposal and the other one votes against, the reward for the two for-voter will be:

1st voter: 3333333333333333333 (wei)
2nd voter: 3333333333333333334 (wei)

and the voter votes against will not get any tokens as reward. In this case, there are 3333333333333333333 wei of ether remaining in the contract. However, there is no withdraw function that can take the token out of the contract.

Simple PoC:

```js
function testDistributeAmount() public { // Assume there are 5 allowed voters in total.
    vm.prank(address(0x1));
    booth.vote(true);

    vm.prank(address(0x2));
    booth.vote(true);

    vm.prank(address(0x3));
    booth.vote(false);

    console2.log(address(0x1).balance); // 3333333333333333333 wei
    console2.log(address(0x2).balance); // 3333333333333333334 wei
    console2.log(address(0x3).balance);  // 0 wei
    console2.log(address(booth).balance); // 3333333333333333333 wei but fail to withdraw
}
```

## Impact

Every voting result that `totalVotesAgainst < totalVotesFor` but `totalVotesFor != totalVotes` will have remaining tokens, and these tokens are unable to transfer to any address, thus locking in the contract.

## Tools Used

Manual Review

## Recommendations

At the end of the `VotingBooth::_distributeReward` funciton, send the remaining ether to the `s_creator` or design a withdraw function.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Incompatibility of Solidity 0.8.23 with Arbitrum: Deployment Failures Due to Unsupported PUSH0 Opcode            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L2

## Summary

Deployment will fail or the execution will have unintended behavior for Solidity version 0.8.23 on Arbitrum Network.

## Vulnerability Details

According to the documentation, the contract is intended to be deployed on the Arbitrum network using version 0.8.23:
However, the Arbitrum network does not support push0 opcode in the latest Solidity compiler version. 
## Impact

It will lead to compilation error or intended behavior while deployment.

## Tools Used

Manual Review

## Recommendations

Downgrade to lower version of Solidity such as 0.8.18




