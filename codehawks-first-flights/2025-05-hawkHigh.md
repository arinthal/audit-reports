### [H-#] The `LevelOne::graduateAndUpgrade` function does not upgrade the proxy to a new implementation

## Summary

The `LevelOne::graduateAndUpgrade` function is intended to upgrade the proxy to a newer implementation. According to the OpenZeppelin documentation (https://docs.openzeppelin.com/contracts/5.x/api/proxy#UUPSUpgradeable), upgrading a UUPS proxy requires calling the `upgradeToAndCall` function imported from the `UUPSUpgradeable` contract. In this case, that call should be made from within the `LevelOne` contract. However, as demonstrated in the PoC below, the proxy's implementation is not actually upgraded.

Additionally, the `LevelTwo` contract, which contains the new implementation, should comply with the EIP-1967 standard — but unfortunately, it does not.

## Impact

The proxy implementation cannot be upgraded using the `LevelOne::graduateAndUpgrade` function.

## Proof of Code

Add the following code to the `LevelOneAndGraduateTest.t.sol` file within the `LevelOneAndGraduateTest` contract.

```solidity
    function test_confirm_can_upgrade() public schoolInSession {
        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        bytes32 IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

        bytes32 oldImpl = vm.load(proxyAddress, IMPLEMENTATION_SLOT);
        address oldImplAddr = address(uint160(uint256(oldImpl)));

        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);

        bytes32 newImpl = vm.load(proxyAddress, IMPLEMENTATION_SLOT);
        address newImplAddr = address(uint160(uint256(newImpl)));

        // equal implementation addresses proves, that implementation hasn't upgraded properly
        assertEq(newImplAddr, oldImplAddr);

        //new implementation address is not equal to deployed LevelTwo contract address
        assertNotEq(newImplAddr, levelTwoImplementationAddress);
    }
```

According to the docs (https://docs.openzeppelin.com/contracts/5.x/api/proxy#UUPSUpgradeable), the `bytes32 IMPLEMENTATION_SLOT` holds the address of the current implementation contract. This value should change when the implementation is upgraded. However, in this case, the value in the slot remains unchanged, indicating that the upgrade did not occur.

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

To upgrade the current implementation of a `UUPSUpgradeable` contract to a newer version, the `upgradeToAndCall` function should be called from the current implementation contract, passing the new implementation address and, optionally, additional initialization data as parameters. This can be done as demonstrated in the code below.

We need to update the code below in the `LevelOne.sol` file.

```diff
    function graduateAndUpgrade(
        address _levelTwo,
-        bytes memory
+        bytes memory data
    ) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }

        uint256 totalTeachers = listOfTeachers.length;

        uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo);

+       upgradeToAndCall(_levelTwo, data);

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }
```

The new implementation contract should follow the EIP-1967 standard, as shown below.

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.26;

+ import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

- contract LevelTwo is Initializable {
+ contract LevelTwo is Initializable, UUPSUpgradeable {
    using SafeERC20 for IERC20;

    address principal;
    bool inSession;
    uint256 public sessionEnd;
    uint256 public bursary;
    uint256 public cutOffScore;
    mapping(address => bool) public isTeacher;
    mapping(address => bool) public isStudent;
    mapping(address => uint256) public studentScore;
    address[] listOfStudents;
    address[] listOfTeachers;

    uint256 public constant TEACHER_WAGE_L2 = 40;
    uint256 public constant PRINCIPAL_WAGE_L2 = 5;
    uint256 public constant PRECISION = 100;

    IERC20 usdc;

+    error HH__NotPrincipal();

+    modifier onlyPrincipal() {
+        if (msg.sender != principal) {
+            revert HH__NotPrincipal();
+        }
+        _;
+    }


    // here will be the contract functions


+    function _authorizeUpgrade(
+        address newImplementation
+    ) internal override onlyPrincipal {}
}

```

This ensures that the contract can be upgraded to a new version.

### [H-#] Storage collision issue after upgrading to the `LevelTwo` contract implementation

## Summary

When upgrading a proxy to a new implementation, the storage layouts of both contracts must be compatible. The existing storage layout cannot be modified. For example, if the first implementation stores an `address principal` variable in the first storage slot, then the new implementation must also store the same variable in the same slot. Subsequent storage slots must follow the same order in both contracts. The only allowed change is appending new state variables after the existing ones.

Violating this rule can lead to unintended changes in variable values due to storage collisions.

## Impact

Upgrading the implementation to `LevelTwo` will result in a storage collision.

## Proof of Concept

Inspired by the OpenZeppelin documentation (https://docs.openzeppelin.com/upgrades-plugins/proxies), we can illustrate example where potential storage collision issues might occur.

| LevelOne           | LevelTwo                  |                       |
| ------------------ | ------------------------- | --------------------- |
| address principal  | address principal         |                       |
| bool inSession     | bool inSession            |                       |
| uint256 schoolFees | uint256 public sessionEnd | <== Storage collision |
| ...                | ...                       |                       |

A problem occurs during the upgrade to `LevelTwo` if the `sessionEnd` variable occupies the same storage slot as `schoolFees` in the proxy contract, leading to unintended value changes.

## Proof of Code

**Note** The provided contracts are not upgradeable in their original form, so the code shown below has been modified from the competition version to demonstrate the storage collision vulnerability.

In the `LevelOne.sol` file, the `graduateAndUpgrade` function needs to be modified.

```diff
    function graduateAndUpgrade(
        address _levelTwo,
        bytes memory data
    ) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }

        uint256 totalTeachers = listOfTeachers.length;

        uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo);

+       upgradeToAndCall(_levelTwo, data);

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }
```

In the `LevelTwo.sol` file, we need to add the following code to make the contract compatible with the EIP-1967 standard.

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.26;

+ import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

- contract LevelTwo is Initializable {
+ contract LevelTwo is Initializable, UUPSUpgradeable {
    using SafeERC20 for IERC20;

    address principal;
    bool inSession;
    uint256 public sessionEnd;
    uint256 public bursary;
    uint256 public cutOffScore;
    mapping(address => bool) public isTeacher;
    mapping(address => bool) public isStudent;
    mapping(address => uint256) public studentScore;
    address[] listOfStudents;
    address[] listOfTeachers;

    uint256 public constant TEACHER_WAGE_L2 = 40;
    uint256 public constant PRINCIPAL_WAGE_L2 = 5;
    uint256 public constant PRECISION = 100;

    IERC20 usdc;

+    error HH__NotPrincipal();

+    modifier onlyPrincipal() {
+        if (msg.sender != principal) {
+            revert HH__NotPrincipal();
+        }
+        _;
+    }


    // here will be the contract functions


+    function _authorizeUpgrade(
+        address newImplementation
+    ) internal override onlyPrincipal {}
}
```

Add the following code to the `LevelOneAndGraduateTest.t.sol` file, inside the `LevelOneAndGraduateTest` contract.

```solidity
    function test_confirm_can_upgrade_with_storage_collision()
        public
        schoolInSession
    {
        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        bytes32 IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

        bytes32 oldImpl = vm.load(proxyAddress, IMPLEMENTATION_SLOT);
        address oldImplAddr = address(uint160(uint256(oldImpl)));

        uint256 levelOneSchoolFees = LevelOne(proxyAddress).getSchoolFeesCost();

        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);

        bytes32 newImpl = vm.load(proxyAddress, IMPLEMENTATION_SLOT);
        address newImplAddr = address(uint160(uint256(newImpl)));

        uint256 levelTwoSessionEnd = LevelTwo(proxyAddress).sessionEnd();

        // different implementation addresses prove that the upgrade to the new implementation has been performed correctly
        assertNotEq(newImplAddr, oldImplAddr);

        // levelOneSchoolFees and levelTwoSessionEnd variables have the same values
        assertEq(levelOneSchoolFees, levelTwoSessionEnd);
    }
```

The `schoolFees` variable from the `LevelOne` contract and the `sessionEnd` variable from the `LevelTwo` contract are stored in the same slot. This results in a storage collision, where the `sessionEnd` variable in the `LevelTwo` contract overwrites the value of the `sessionEnd` variable from the `LevelOne` contract.

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

The order and number of storage variables in the `LevelTwo` contract must match those in the `LevelOne` contract. This ensures that all variables from the first implementation are stored in the same slots in the second implementation.
Any additional variables in the second contract should be appended after the variables already declared in the first implementation.

### [L-#] No protection against premature system updates

## Summary

The `LevelOne` contract lacks a mechanism to prevent the director from performing a system upgrade before the session end date, which is defined as 4 weeks from the session start.

## Impact

Violation of invariant rules: "System upgrade cannot take place unless school's `sessionEnd` has reached".

## Proof of Code

Add the following code to the `LevelOneAndGraduateTest.t.sol` file within the `LevelOneAndGraduateTest` contract.

```solidity
    function test_confirm_can_graduate_and_upgrade_before_session_end()
        public
        schoolInSession
    {
        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        // this call is possible, despite the fact that the end of the session has not yet passed
        assert(block.timestamp < levelOneProxy.getSessionEnd());
        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

Ensure that the system cannot be updated before the end of the session. One of the possible ways we see below.

We need to update the code below in the `LevelOne.sol` file.

```diff
+    error HH__SessionNotEnded();
```

```diff
    function graduateAndUpgrade(
        address _levelTwo,
        bytes memory
    ) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }
+        if (block.timestamp < sessionEnd) {
+            revert HH__SessionNotEnded();
+        }

        uint256 totalTeachers = listOfTeachers.length;

        uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo);

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }
```

The changes above ensure that the function can only be executed after the session period has ended.
