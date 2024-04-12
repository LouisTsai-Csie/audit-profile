# First Flight #8: Math Master - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Low Risk Findings
    - ### [L-01. Unsecure Handling of Free Memory Pointer in `mulWad` and `mulWadUp` Functions](#L-01)
    - ### [L-02. Customer error is inconsistent to revert data in assembly code](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 0
   - Low: 2

# Low Risk Findings

## <a id='L-01'></a>L-01. Unsecure Handling of Free Memory Pointer in `mulWad` and `mulWadUp` Functions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L41

https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L54

## Summary

**Introduction:**
Custom error handling was introduced in Solidity version 0.8.4, allowing developers to showcase different error events. However, a vulnerability has been identified in the handling of free memory pointers in the `mulWad` and `mulWadUp` functions.

**Error Event:**
Consider the following error event declaration in Solidity:

```js
error Unauthorized();
```

This error event is used in the code to revert a transaction with an `Unauthorized` error:

```js
revert Unauthorized();
```

The equivalent Yul code for this operation is as follows:

```js
let freeMemPtr := mload(0x40)
mstore(freeMemPtr, Unauthorized.selector)
revert(freeMemPtr, 0x04)
```

## Vulnerability Details

The vulnerability lies in the direct manipulation of memory offset 0x40 without first loading the free memory pointer. This could lead to unpredictable behavior and memory corruption issues.

## Impact

The impact of this vulnerability includes potential memory corruption and unexpected behavior during the execution of the `mulWad` and `mulWadUp` functions. An attacker could potentially exploit this vulnerability to compromise the integrity of the contract.

## Tools Used

Manual Review

## Recommendations

Always load the free memory pointer (freeMemPtr) before manipulating memory at the offset 0x40. This ensures proper handling of memory allocation and prevents unintended consequences.

`mulWad`

```diff
function mulWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
-                mstore(0x40, 0xbac65e5b) 
-                revert(0x1c, 0x04) 
+               let freeMemPtr = mload(0x40)
+               mstore(freeMemPtr, selector)
+               revert(freeMemPtr, 4)
            }
            z := div(mul(x, y), WAD) 
        }
}
```

It should be modified in `mulWadUp` as well.
## <a id='L-02'></a>L-02. Customer error is inconsistent to revert data in assembly code            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L40

https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L53

## Summary

Custom error is introduced since Solidity 0.8.4, allowing developer to showcase different error events. Consider the following error event:

```solidity
error Unauthorized();
```

And the code section below revert the transaction with `Unauthorized` error.

```solidity
revert Unauthorized();
```

It is equivalent to the Yul code as follow:

```solidity
let freeMemPtr := mload(0x40)
mstore(freeMemPtr, Unauthorized.selector)
revert(freeMemPtr, 0x04)
```

The `freeMemPtr` first load the free memory pointer into the variable, and loading the signature of custom error `Unauthorized`, finally revert the transaction with the signature.

However, the value in mstore opcode is mismatched with the error signature.

## Vulnerability Details

In `MathMasters::mulWadUp()` and `MathMasters::mulWad()` functions, there are custom error specified, which is `MathMasters__MulWadFailed()`. However, the value in `revert` opcode is inconsistent to the error signature.

add the following test in `test/MathMasters.t.sol`,

```solidity
function testMulWadErroreEvent() public pure {
    assertEq(MathMasters__MulWadFailed.selector, 0xbac65e5b);
}
```

The test will fail, indicating the revert data is incorrect.

## Impact

The parameter passed in revert opcode is inconsistent to the comment, and it will confuse the developer about the error message.
```diff
assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, div(not(0), y))) {
@>                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04) // @TODO check 0x1c data
            }
            z := div(mul(x, y), WAD) 
}
```

## Tools Used

Foundry

## Recommendations

correct the error signature, updating 0xbac65e5b to 0xe243bb9b0, which is the signature of MathMasters__MulWadFailed error.

`mulWad(uint256 x, uint256 y)`

```diff
assembly {
    // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
    if mul(y, gt(x, div(not(0), y))) {
-     mstore(0x40, 0xbac65e5b) // Incorrect `MathMasters__MulWadFailed()` signature
+     mstore(0x40, 0xe243bb9b) // Correct `MathMasters__MulWadFailed()` signature
        revert(0x1c, 0x04) 
    }
    z := div(mul(x, y), WAD) 
}
```

`mulWadUp(uint256 x, uint256 y)`

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly { 
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, or(div(not(0), y), x))) {
-               mstore(0x40, 0xbac65e5b) //  Correct `MathMasters__MulWadFailed()` signature
+              mstore(0x40, 0xe243bb9b) // Incorrect `MathMasters__MulWadFailed()` signature
                revert(0x1c, 0x04)
            }
            if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
            z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
        }
    }
```


