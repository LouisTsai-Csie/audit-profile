# First Flight #1: PasswordStore - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. On-chain password data are accessible by off-chain tools and lead to privacy concerns](#H-01)
    - ### [H-02. The setPassword function can be modified by any user](#H-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>[H-01] On-chain password data are accessible by off-chain tools, leading to privacy concerns            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L35

## Summary

The sensitive password, `PasswordStore::s_password`, can potentially be retrieved by off-chain tools, which violates the restriction outlined in the `PasswordStore::getPassword` function.

## Vulnerability Details

According to the description of the protocol, other users are unable to access the `PasswordStore::s_password`. However, the `PasswordStore::s_password` state variable with private visibility does not guarantee the value is inaccessible to user utilizing off-chain tools. The private keyword only restricts the accessibility of the variable to inheritance contracts and external contracts.

Several off-chain tools support the operation to query certain storage slot value under an address, such as `eth_getStorageAt` from Alchemy and `getStorageAt` from `ethers.js`, those tools can retrieve the value of state variables from given contract, leading to the leak of the sensitive password value.

## Impact

Considering the protocol is utilized for health-related scenarios, and users store their sensitive data in the protocol without knowing the possibility of data breach, it will lead to critical privacy concerns and should be prevented at once.

## Tools Used

Manual Review

## Recommendations

Considering the property of blockchain system, the data stored on the system is accessible to all of the users. If the user still wants to store such privacy data on chain. encryption and other technique should be taken into consideration.

## <a id='H-02'></a>[H-02] The setPassword function can be modified by any user            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L26C5-L26C5

## Summary

The `setPassword` function does not obey the function specification and arbitrary user can update the password.

## Vulnerability Details

In the description of `PasswordStore::setPassword` function, there is a limitation that only the owner, `PasswordStore::s_owner`, can set the password. However, the deficiency of authentication leads to critical issue that arbitrary user can modify the password. No corresponding modifiers or statement is provided.

Run the following proof-of-concept in the `PasswordStoreTest` with ```forge test --mt test_arbitrary_user_can_set_password```

```js
function test_arbitrary_user_can_set_password() public {
    address user = makeAddr("user");

    string memory newPassword = "newPassword";
    vm.prank(user);
    passwordStore.setPassword(newPassword);

    vm.prank(owner);
    string memory actualPassword = passwordStore.getPassword();

    assertEq(newPassword, actualPassword);
}
```

## Impact

Since the password can be updated by arbitrary user, the owner of the contract might retrieve distinct password as the he/she stores. It will be a critical issue if the protocol serves as authentication purposes, owner will get incorrect password and leads to unintended behavior.

## Tools Used

Manual Review

## Recommendations

Make use of openzeppelin Ownable template and initialize the owner at the constructor. Also, mark the `setPassword` with `onlyOwner` modifier. If not using oppezeppelin contract, developer should design their own `onlyOwner` modifier and validate the user when the function is triggered.

```diff
function setPassword(string memory newPassword) external {
+   if (msg.sender != s_owner) {
+       revert PasswordStore__NotOwner();
+   }
    s_password = newPassword;
    emit SetNetPassword();
}
```