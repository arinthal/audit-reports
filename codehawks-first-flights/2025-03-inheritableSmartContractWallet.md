### [H-#] No possibility to send ETH to `InheritanceManager` contract

## Summary

The `InheritanceManager` contract cannot receive ETH.

## Vulnerability Details

By default, Ethereum smart contracts cannot accept ETH unless explicitly designed to do so. A contract can receive ETH in the following ways:

1. Defining a `receive()` function.
2. Defining a `fallback()` function.
3. Implementing a `payable` function that is explicitly called.
4. Using another contract that calls `selfdestruct()` function (not recommended in this case).

The `InheritanceManager` contract lacks all of these methods, making it impossible to send ETH using standard approaches (see PoC).

## Impact

The contract does not fulfill its basic wallet functionality because it cannot receive ETH for management.

## Proof of Code

Add the following code to the `InheritanceManagerTest.t.sol` file within the `InheritanceManagerTest` contract.

```solidity
    function test_receivingEthByInheritanceManagerContractFail() public {
        vm.deal(owner, 1 ether);
        vm.prank(owner);
        vm.expectRevert();
        (bool success, ) = address(im).call{value: 1 ether}("");
        require(success, "Transfer Failed");
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

To allow ETH transfers, the `InheritanceManager` contract should implement either `receive` or `fallback` function (see docs: https://docs.soliditylang.org/en/v0.8.26/contracts.html#receive-ether-function or https://docs.soliditylang.org/en/v0.8.26/contracts.html#fallback-function).

Alternatively, a `payable` function can be added, such as

```solidity
    event Deposit(address sender, uint amount);

    function deposit() external payable {
        require(msg.value > 0, "Must send some ETH");
        emit Deposit(msg.sender, msg.value);
    }
```

This ensures the contract can accept ETH efficiently.

### [H-#] Contract Owner Rights Takeover

## Summary

An attacker can take over the rights of the contract owner by calling the `InheritanceManager::inherit` function.

## Vulnerability Details

If the only beneficiary of the contract's funds is the owner's backup wallet, then after 90 days of inactivity, any user can take over the contract owner's rights and reset the timer by calling `InheritanceManager::inherit`.

## Impact

The attacker gains full control over all funds stored in the contract, including ETH and ERC-20 tokens, and can transfer them to any address.

## Proof of Code

Add the following code to the `InheritanceManagerTest.t.sol` file within the `InheritanceManagerTest` contract.

```solidity
    function test_gainContractOwership() external {
        address attacker = makeAddr("attacker");

        // add backup wallet address to the list of beneficiaries
        // a 90-day timer is set
        vm.prank(owner);
        im.addBeneficiery(user1);

        // the designated period of contract inactivity has passed
        skip(90 days);

        // attacker calls inheritance function
        vm.prank(attacker);
        im.inherit();

        // attacker took over the rights of the owner
        assertEq(attacker, im.getOwner());
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

To prevent unauthorized ownership transfer, the `InheritanceManager::inherit` function should include the `onlyBeneficiaryWithIsInherited` modifier. This ensures that only designated beneficiaries or the owner's backup wallet can execute the function, preventing the described attack.

```diff
-   function inherit() external {
+   function inherit() external onlyBeneficiaryWithIsInherited {
        if (block.timestamp < getDeadline()) {
            revert InactivityPeriodNotLongEnough();
        }
        if (beneficiaries.length == 1) {
            owner = msg.sender;
            _setDeadline();
        } else if (beneficiaries.length > 1) {
            isInherited = true;
        } else {
            revert InvalidBeneficiaries();
        }
    }
```

### [M-#] No 90-day timer reset in some functions

## Summary

Some functions that modify the contract when called by the owner do not reset the 90-day timer.

## Vulnerability Details

According to the docs, assets can be dstributed to beneficiaries after a set period of wallet inactivity. The functions `InheritanceManager::removeBeneficiary` and `InheritanceManager::contractInteractions` should reset the 90-day timer, as they involve contract interactions and modifications.

## Impact

Failure to reset the timer breaks the core assumption of the contract: beneficiaries should only inherit funds after 90 days of wallet inactivity.

## Proof of Code

Add the following code to the `InheritanceManagerTest.t.sol` file within the `InheritanceManagerTest` contract (example for `InheritanceManager::removeBeneficiary` function).

```solidity
    function test_resetTimerAfterOwnerInteractionWithContractFail() external {
        uint256 deadline;
        uint256 expectedDeadline;
        uint256 startTimestamp = 100;

        vm.startPrank(owner);
        vm.warp(startTimestamp);
        im.addBeneficiery(user1);
        deadline = im.getDeadline();
        expectedDeadline = startTimestamp + 90 days;
        assertEq(deadline, expectedDeadline);
        skip(100);

        // after executing below function, the timer should be to 90 days ahead
        // but it's not happening
        im.removeBeneficiary(user1);
        vm.stopPrank();

        deadline = im.getDeadline();
        expectedDeadline = block.timestamp + 90 days;

        assertNotEq(deadline, expectedDeadline);
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended mitigation

The functions `InheritanceManager::removeBeneficiary` and `InheritanceManager::contractInteractions` should call the internal function `InheritanceManager::_setDeadline` to reset the 90-day timer after performing their operations.
Proposed changes:

```diff
    function contractInteractions(
        address _target,
        bytes calldata _payload,
        uint256 _value,
        bool _storeTarget
    ) external nonReentrant onlyOwner {
        (bool success, bytes memory data) = _target.call{value: _value}(
            _payload
        );
        require(success, "interaction failed");
        if (_storeTarget) {
            interactions[_target] = data;
        }
+       _setDeadline();
    }
```

```diff
    function removeBeneficiary(address _beneficiary) external onlyOwner {
        uint256 indexToRemove = _getBeneficiaryIndex(_beneficiary);
        delete beneficiaries[indexToRemove];
+       _setDeadline();
    }
```

This ensures that any owner interaction resets the timer, maintaining the contract’s intended functionality.
