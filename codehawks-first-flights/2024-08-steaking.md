### [L-#] Incorrect staked ETH amount after withdrawal

## Summary

Withdrawing of a correspondingly smaller amount of funds than previously staked by the user results in staking an amount below the required minimum (less than 0.5 ETH).

## Vulnerability Details

The `Steaking::unstake` function allows to withdraw staked assets of the user during staking period. Users can use it to adjust their staked ETH amount, or withdraw it completely. The additional requirement is that minimum staked amount is `0.5` ether. It is possible to broke this invariant when user first stake specific amount of ether and then withdraw a little less, where (staked amount - withdrawn amount) < 0.5 ether.

## Impact

The minimum staked amount requirement could be broken.

## Proof of Concepts

1. user stakes the specific amount of assets, greater or equal to the minimum requirement.
2. user withdraws a little less than the staked amount before the staking period ends.

## Proof of code

Add the following code to the `Steaking.t.sol` file within the `SteakingTest` contract.

```solidity
    function testUnstakeBelowMinimumStakedAmount() public {
        uint256 user1StakedAmount;
        uint256 dealAmount = 1 ether;
        uint256 unstakeAmount = 0.8 ether;

        _stake(user1, dealAmount, user1);

        user1StakedAmount = steaking.usersToStakes(user1);
        assertEq(user1StakedAmount, dealAmount);

        _unstake(user1, unstakeAmount, user1);

        user1StakedAmount = steaking.usersToStakes(user1);
        assertEq(user1StakedAmount, dealAmount - unstakeAmount);

        assertLt(user1StakedAmount, 0.5 ether);
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

The `Steaking::unstake` should check that the remaining amount (staked amount - withdrawn amount) is either greater than the minimum staked value (0.5 ETH) or equal to 0.

Possible solution (changes in `Steaking.vy` file):

```diff
+STEAK__STEAK_AMOUNT_LESS_THAN_MINIMUM: constant(String[33]) = "Steak__SteakAmountLessThanMinimum"
```

```diff
def unstake(_amount: uint256, _to: address):
    """
    @notice Allows users to unstake their staked ETH before the staking period ends. Users
    can adjust their staking amounts to their liking.
    @param _amount The amount of staked ETH to withdraw.
    @param _to The address to send the withdrawn ETH to.
    """
    assert not self._hasStakingPeriodEnded(), STEAK__STAKING_PERIOD_ENDED
    assert _to != ADDRESS_ZERO, STEAK__ADDRESS_ZERO

    stakedAmount: uint256 = self.usersToStakes[msg.sender]
    assert stakedAmount > 0 and _amount > 0, STEAK__AMOUNT_ZERO

+   remainingAmount: uint256 = stakedAmount - _amount
+   assert remainingAmount == 0 or remainingAmount >= MIN_STAKE_AMOUNT, STEAK__STEAK_AMOUNT_LESS_THAN_MINIMUM

    assert _amount <= stakedAmount, STEAK__INSUFFICIENT_STAKE_AMOUNT

    self.usersToStakes[msg.sender] -= _amount
    self.totalAmountStaked -= _amount

    send(_to, _amount)

    log Unstaked(msg.sender, _amount, _to)
```

### [M-#] Incorrect amount of staked assets may be assigned when staking on behalf of the user

## Summary

Staking ETH on behalf of another user overwrites the previous value for the user receiving the staking assets.

## Vulnerability Details

The `Steaking::stake` function allows staking raw ETH on behalf of another user. When user (user2) stakes assets on behalf of another user (user1), who had previously staked assets for himself, the previously stored value in the `usersToStakes` variable for user1 will be overwritten with the new value from `msg.value` from the transaction sent by user2.

```python
  def stake(_onBehalfOf: address):
      """
      @notice Allows users to stake ETH for themselves or any other user within the staking period.
      @param _onBehalfOf The address to stake on behalf of.
      """
      assert not self._hasStakingPeriodEnded(), STEAK__STAKING_PERIOD_ENDED
      assert msg.value >= MIN_STAKE_AMOUNT, STEAK__INSUFFICIENT_STAKE_AMOUNT
      assert _onBehalfOf != ADDRESS_ZERO, STEAK__ADDRESS_ZERO

@>    self.usersToStakes[_onBehalfOf] = msg.value
      self.totalAmountStaked += msg.value

      log Staked(msg.sender, msg.value, _onBehalfOf)
```

## Impact

Assets previously staked by user1 will no longer be tracked by the `usersToStakes` variable, though they will still be included in the `totalAmountStaked` variable. As a result, user1 will be unable to withdraw their originally staked assets and will only have access to the amount staked on their behalf by user2. In an edge case, user2 could reduce allocation of user1 to 0.5 ETH, which is the minimum stake amount.

## Proof of Concept

1. user1 stakes 5 ETH, and this amount is assigned for him in the `usersToStakes` variable
2. user2 then stakes 0.5 ETH on behalf of user1, which overwrites the previous value in the `usersToStakes` variable. As a result, user1 is now only able to withdraw 0.5 ETH, instead of the originally staked 5 ETH.

## Proof of Code

Add the following code to the `Steaking.t.sol` file within the `SteakingTest` contract.

```solidity
    function testStakeOnBehalfOf() public {
        address user2 = makeAddr("user2");

        uint256 dealAmount1 = 5 ether;
        uint256 dealAmount2 = steaking.getMinimumStakingAmount(); // 0.5 ether

        _stake(user1, dealAmount1, user1);
        _stake(user2, dealAmount2, user1);

        uint256 user1StakedAmount = steaking.usersToStakes(user1);
        uint256 user2StakedAmount = steaking.usersToStakes(user2);

        uint256 totalAmountStaked = steaking.totalAmountStaked();

        assertEq(user1StakedAmount, dealAmount2);
        assertEq(user2StakedAmount, 0);
        assertEq(totalAmountStaked, dealAmount1 + dealAmount2);
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended mitigation

Instead of setting a new value of the staked ETH, the amount form the `msg.value` variable should be added to the previously staked ETH for that specific user.

```diff
  def stake(_onBehalfOf: address):
      """
      @notice Allows users to stake ETH for themselves or any other user within the staking period.
      @param _onBehalfOf The address to stake on behalf of.
      """
      assert not self._hasStakingPeriodEnded(), STEAK__STAKING_PERIOD_ENDED
      assert msg.value >= MIN_STAKE_AMOUNT, STEAK__INSUFFICIENT_STAKE_AMOUNT
      assert _onBehalfOf != ADDRESS_ZERO, STEAK__ADDRESS_ZERO

-      self.usersToStakes[_onBehalfOf] = msg.value
+      self.usersToStakes[_onBehalfOf] += msg.value
      self.totalAmountStaked += msg.value

      log Staked(msg.sender, msg.value, _onBehalfOf)
```
