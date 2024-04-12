# First Flight #7: Horse Store - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Unable to mint NFT for the second time in huff version](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Lack of `onERC721Received` in `safeTransferFrom` in the huff version](#M-01)
    - ### [M-02. Inconsistent behavior between Huff and Solidity version that can lead to lock of token](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #7

### Dates: Jan 11th, 2024 - Jan 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/clr6s75ut00013qg9z8bpkalo)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 2
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Unable to mint NFT for the second time in huff version            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L326

## Summary

The contract can only mint the NFT once, and was unable to mint other NFT due to bad implementation of `MINT` function in huff.

## Vulnerability Details

The following fuzzing test will fail in huff version:

```
function testMintHorseMultipleTimes(uint256 n) public {
        vm.assume(n<=5);
        vm.startPrank(user);
        for(uint256 i=0;i < n;i++) 
            horseStore.mintHorse();
        vm.stopPrank();
        assertEq(horseStore.balanceOf(user), n);
    }
```

The huff version can not mint more than one NFT due to logic flaw in `MINT` function.

The `MINT` function wants to validate the `to` variable to ensure it is not zero address, but the order of stack value is incorrect, and it turns out the validation only allows token id to be zero.

```
dup1 iszero invalid_recipient jumpi             // [to, tokenId]
```

It first duplicates the top of stack value, it will be token id. It is the token id that is validated not the `to` variable.

## Impact

Only the first NFT can be minted, and other user can not execute the mint operation

## Tools Used

Update the validation of the following

`dup1 iszero invalid_recipient jumpi             // [to, tokenId]`


## Recommendations
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of `onERC721Received` in `safeTransferFrom` in the huff version            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L485

## Summary

The `safeMInt` function in Solidity version will verify whether the address is an EOA or an contract address, if it is the latter one, the `mint` operation should check the return value of `onERC721Received` in receiving contract to safety.

## Vulnerability Details

Add a mock contract that implement the `onERC721Received` function.

```
contract MockReceiver {

    HorseStore internal horseStore;

    constructor(HorseStore _horseStore) payable {
        horseStore = _horseStore;
    }

    function mintHorse() external {
        horseStore.mintHorse();
    }
  
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        return bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"));
    }
}
```

Add the following test in the `Base_Test`, run `forge test --match-test testMintHorseContract`

```
function testMintHorseContract() public {
        vm.prank(user);
        MockReceiver receiver = new MockReceiver(horseStore);
        receiver.mintHorse();

        assertEq(horseStore.balanceOf(address(receiver)), 1);
}
```

It will both succeed in solidity version and huff version. However, if we remove the `onERC721Received` function in the receiving contract, the anticipated behavior is to revert the transaction since the receiving contract does not follow the `IERC721Receiver` pattern.
## Impact

Not implementing the `IERC721Receiver` validation is dangerous since the receiving contract might not be able to withdraw or do operation to the received NFT, in this case, the NFT will be locked in the contract forever, and the difference between the two version is also an issue.

## Tools Used

Foundry

## Recommendations

Implement the validation of `onERC721Received` in the mint operation and check whether contract address has the ability to handle the NFT.
## <a id='M-02'></a>M-02. Inconsistent behavior between Huff and Solidity version that can lead to lock of token            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L205

## Summary

The solidity version of horse store does not implement `receive` and `fallback` function, the contract should be unable to receive native ether token, but the. huff version does not follow the pattern, either does the huff version implement withdraw function, leading to lock of ether.

## Vulnerability Details

Add the following testing:

```
function testReceive() public {    
        deal(user, 1 ether);
        vm.prank(user);
        (bool success, ) = address(horseStore).call{value: 1 ether}("X");
        assertTrue(success);
    }
```

The huff version success indicating that the ether is transferred to the user, however, the solidity version contract reverts since both `fallback` and `receive` is not implemented.

## Impact

The huff version of `horseStore` is able to receive ether but does not implement the withdraw function, it will lead to lock of ether.

## Tools Used

Foundry

## Recommendations

Implement `withdraw` function in the contract or restrict the ether transfer to the contract.