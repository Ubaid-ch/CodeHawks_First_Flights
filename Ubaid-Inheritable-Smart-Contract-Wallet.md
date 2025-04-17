# Inheritable Smart Contract Wallet - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. Rounding Error in withdrawInheritedFunds( ) Leaves Residual Funds in Contract](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #35

### Dates: Mar 6th, 2025 - Mar 13th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-03-inheritable-smart-contract-wallet)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 0
- Low: 1



    


# Low Risk Findings

## <a id='L-01'></a>L-01. Rounding Error in withdrawInheritedFunds( ) Leaves Residual Funds in Contract            



## Summary

In `withdrawInheritedFunds` function in `InheritanceManager.sol`, rounding errors can arise when dividing the total asset balance (`ethAmountAvailable` or `assetAmountAvailable`) by the number of beneficiaries (`divisor`).

In Solidity (and most programming languages), division of integers (`uint256`) truncates any fractional part. For example:

* If `ethAmountAvailable = 100` and `divisor = 3`, then `amountPerBeneficiary = 100 / 3 = 33` (the fractional part `0.333...` is discarded).
* This means the total distributed amount would be `33 * 3 = 99`, leaving `1` unit of the asset undistributed in the contract.

## Vulnerability Details

```solidity
 function withdrawInheritedFunds(address _asset) external {
    if (!isInherited) {
        revert NotYetInherited();
    }
    uint256 divisor = beneficiaries.length;
    if (_asset == address(0)) {
        uint256 ethAmountAvailable = address(this).balance;
        uint256 amountPerBeneficiary = ethAmountAvailable / divisor;
        for (uint256 i = 0; i < divisor; i++) {
            address payable beneficiary = payable(beneficiaries[i]);
            (bool success,) = beneficiary.call{value: amountPerBeneficiary}("");
            require(success, "something went wrong");
        }
    } else {
        uint256 assetAmountAvailable = IERC20(_asset).balanceOf(address(this));
        uint256 amountPerBeneficiary = assetAmountAvailable / divisor;
        for (uint256 i = 0; i < divisor; i++) {
            IERC20(_asset).safeTransfer(beneficiaries[i], amountPerBeneficiary);
        }
    }
}
```

# POC

This test makes sure the fair distribution of funds and no rseidual funds should be left behind

```solidity
  function test_withdrawInheritedFundsEtherSuccess() public {
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);
        vm.stopPrank();
        vm.warp(1);
        vm.deal(address(im), 10e18);
        vm.warp(1 + 90 days);
        vm.startPrank(user1);
        im.inherit();
        im.withdrawInheritedFunds(address(0));
        vm.stopPrank();
        //Equal Balances to user1, user2, user3
        assertEq(user1.balance, user2.balance); 
        assertEq(user2.balance, user3.balance); 
        // there should be no funds left in the contract
        assertEq(address(im).balance, 0); 
        
    }
```

The last assertion fails which tells that  residual funds are trapped in contract

```Solidity
[FAIL. Reason: assertion failed: 1 != 0] test_withdrawInheritedFundsEtherSuccess() (gas: 264719)
```

And same goes for ERC20 withdrawal

```Solidity
 function test_withdrawInheritedFundsERC20Success() public {
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);
        vm.stopPrank();
        vm.warp(1);
        usdc.mint(address(im), 10e18);
        vm.warp(1 + 90 days);
        vm.startPrank(user1);
        im.inherit();
        im.withdrawInheritedFunds(address(usdc));
        vm.stopPrank();
        assertEq(usdc.balanceOf(user1)==usdc.balanceOf(user2), usdc.balanceOf(user1)==usdc.balanceOf(user3));
      
        assertEq(0, usdc.balanceOf(address(im)));
        console.log(usdc.balanceOf(user1));
        console.log(usdc.balanceOf(user2));
        console.log(usdc.balanceOf(user3));
        console.log(usdc.balanceOf(address(im)));
    }
```

The last assertion fails which tells that residual funds are locked in contract

```Solidity
[FAIL. Reason: assertion failed: 0 != 1] test_withdrawInheritedFundsERC20Success() (gas: 295217)
```

## Impact

* Leftover funds are locked in contract.

  There is no rules defined for those residual funds

## Tools Used

* Foundry

## Recommendations

You can decide what to do with residual funds.\
I have decided to give the residual funds to the first beneficiary(as crypto rewards the early adapters lol🥱)



```solidity
  
 function withdrawInheritedFunds(address _asset) external {
    if (!isInherited) {
        revert NotYetInherited();
    }
    uint256 divisor = beneficiaries.length;
    
    if (_asset == address(0)) {
        uint256 ethAmountAvailable = address(this).balance;
        uint256 amountPerBeneficiary = ethAmountAvailable / divisor;
        for (uint256 i = 0; i < divisor; i++) {
            uint256 amountToSend = (i == 0) 
                ? (ethAmountAvailable - (amountPerBeneficiary * (divisor - 1))) 
                : amountPerBeneficiary;
            address payable beneficiary = payable(beneficiaries[i]);
            (bool success, ) = beneficiary.call{value: amountToSend}("");
            require(success, "Ether transfer failed");
        }
    } else {
        uint256 assetAmountAvailable = IERC20(_asset).balanceOf(address(this));
        uint256 amountPerBeneficiary = assetAmountAvailable / divisor;
        for (uint256 i = 0; i < divisor; i++) {
            uint256 amountToSend = (i == 0) 
                ? (assetAmountAvailable - (amountPerBeneficiary * (divisor - 1))) 
                : amountPerBeneficiary;
            IERC20(_asset).safeTransfer(beneficiaries[i], amountToSend);
        }
    }
}
```



The improved tests are below

```Solidity
  function test_withdrawInheritedFundsEtherSuccess() public {
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);
        vm.stopPrank();
        vm.warp(1);
        vm.deal(address(im), 10e18);
        vm.warp(1 + 90 days);
        vm.startPrank(user1);
        im.inherit();
        im.withdrawInheritedFunds(address(0));
        vm.stopPrank();
        //User1 should have more funds than user2 and user3
        assertGt(user1.balance, user2.balance); 
        //user2 and user3 should have the same balance
        assertEq(user2.balance, user3.balance); 
        // there should be no funds left in the contract
        assertEq(address(im).balance, 0); 
        
    }

    function test_withdrawInheritedFundsERC20Success() public {
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);
        vm.stopPrank();
        vm.warp(1);
        usdc.mint(address(im), 10e18);
        vm.warp(1 + 90 days);
        vm.startPrank(user1);
        im.inherit();
        im.withdrawInheritedFunds(address(usdc));
        vm.stopPrank();
        assertGt(usdc.balanceOf(user1), usdc.balanceOf(user2)); // user1 > user2 (remainder)
        assertEq(usdc.balanceOf(user2), usdc.balanceOf(user3));
        assertEq(0, usdc.balanceOf(address(im)));
        console.log(usdc.balanceOf(user1));
        console.log(usdc.balanceOf(user2));
        console.log(usdc.balanceOf(user3));
        console.log(usdc.balanceOf(address(im)));
    }  

```



