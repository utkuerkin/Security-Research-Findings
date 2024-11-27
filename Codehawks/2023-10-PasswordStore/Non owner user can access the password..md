### Severity

High Risk

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol

## Summary

Any user can access the password as they want even if they are not the owner.

## Vulnerability Details

Contract uses `string private s_password;` variable at 
Line #14 to store the password in the storage. But since everything is 
public in the blockchain anyone will have access to this password if 
they can read the storage.
In foundry we can use `cast storage ContractAddress` to get the full storage layout of this contract. Or we can use `cast storage ContractAddress, StorageSlotOfPassword` to access the stored password.

Alternatively we can use `const contents = await web3.eth.getStorageAt(contractAddress, StorageSlotOfPassword)` .

## Impact

Impact of this vulnerability is high since it defeats the whole 
purpose of the contract. Password is supposed to be private but anyone 
will have access to it.

## Tools Used

Manual review and Foundry.

## Recommendations

Do not use blockchain to store passwords as strings, everything in the blockchain is public data.

### Result
Validated