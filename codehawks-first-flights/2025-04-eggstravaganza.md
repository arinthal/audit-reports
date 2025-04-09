### [M-#] Error during token minting process after changing the game contract

## Summary

The `EggstravaganzaNFT` contract allows its owner to update the associated game contract used for token minting. However, if a game has already been played and tokens have been minted, updating the game contract prevents any further token minting. As a result, the game can no longer function as intended, until the original game contract is not set again.

## Vulnerability Details

As players discover more eggs, tokens are minted with sequential `tokenId` values (1, 2, 3, ...). If the `EggstravaganzaNFT` contract is updated in a new game contract that uses the same `tokenId` generation logic as the original `EggHuntGame` contract, it will attempt to mint tokens with `tokenId` values that already exist, leading to conflicts or errors.

## Impact

Minting new tokens is not possible through a newly set game contract.

## Proof of Code

Add the following code to the `EggHuntGameTest.t.sol` file within the `EggGameTest` contract.

```solidity
    function testChangingGameContractWhenGameWasPreviouslyPlayed() public {
        uint256 duration = 100;

        // start the game and win an egg
        game.startGame(duration);
        game.setEggFindThreshold(100);

        vm.prank(alice);
        game.searchForEgg();

        assertEq(nft.totalSupply(), 1);
        assertEq(nft.ownerOf(1), alice);

        // create new game contract
        EggHuntGame game2 = new EggHuntGame(address(nft), address(vault));
        game2.setEggFindThreshold(100);

        // change the game contract in NFT contract
        nft.setGameContract(address(game2));

        // start new game and try to win an egg
        game2.startGame(duration);

        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(
                IERC721Errors.ERC721InvalidSender.selector,
                address(0)
            )
        );
        game2.searchForEgg();
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

To resolve this issue, the definition of tokenId can be moved from the `EggHuntGame` contract to the `EggstravaganzaNFT` contract. The `EggstravaganzaNFT::mintEgg` function can then be modified to assign `tokenId` values based on the current total number of tokens minted. This ensures each new token receives a unique `tokenId` that increments with every minting.

File `EggstravaganzaNFT.sol`:

```diff
-   function mintEgg(address to, uint256 tokenId) external returns (bool) {
+   function mintEgg(address to) external returns (bool) {
        require(msg.sender == gameContract, "Unauthorized minter");
+       totalSupply += 1;
+       uint256 tokenId = totalSupply;
        _mint(to, tokenId);
-       totalSupply += 1;
        return true;
    }
```

File `EggHuntGame.sol`:

```diff
-       eggNFT.mintEgg(msg.sender, eggCounter);
+       eggNFT.mintEgg(msg.sender);
```

File `EggHuntGameTest.t.sol`:

line 58:

```diff
-       bool success = nft.mintEgg(alice, 1);
+       bool success = nft.mintEgg(alice);
```

line 68:

```diff
-       nft.mintEgg(bob, 2);
+       nft.mintEgg(bob);
```

line 77:

```diff
-       nft.mintEgg(address(vault), 10);
+       nft.mintEgg(address(vault));
```

line 197:

```diff
-       nft.mintEgg(alice, 20);
+       nft.mintEgg(alice);
```

line 253:

```diff
-       nft.mintEgg(alice, 30);
+       nft.mintEgg(alice);
```

This approach ensures that the NFT contract (`EggstravaganzaNFT`) retains full control over the generation of new `tokenId` values.

### [L-#] No protection against early game termination

**Note**: The documentation does not clarify whether the game owner is allowed to end the game before the scheduled end time. In the scenario shown below, the game is expected to run for a fixed duration defined at its start.

## Summary

Currently, there is no safeguard preventing the game owner from ending the game prematurely. Players should have the full, predetermined time to search for hidden eggs, and the owner should not have the ability to cut the game short.

## Impact

The game can be terminated before the originally defined end time.

## Proof of Code

Add the following code to the `EggHuntGameTest.t.sol` file within the `EggGameTest` contract.

```solidity
    function testEndGameBeforeEndOfItsDuration() public {
        // start game
        uint256 duration = 100;
        uint256 currentTime = block.timestamp;
        game.startGame(duration);
        // check game status
        assertEq(game.getGameStatus(), "Game is active");

        // check the setting of start and end times
        assertEq(game.startTime(), currentTime);
        assertEq(game.endTime(), currentTime + 100);

        // warp time to before the game duration
        vm.warp(currentTime + duration - 1);

        // end the game before game ends
        game.endGame();

        assertEq(game.getGameStatus(), "Game is not active");
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

To ensure players have the full allotted time to complete the game, a safeguard should be implemented to prevent the game from ending before the predetermined duration has passed. It is recommended to introduce below changes within the `EggHuntGame` contract.

```diff
    function endGame() external onlyOwner {
        require(gameActive, "Game not active");
+       require(block.timestamp > endTime, "Game is not over yet");
        gameActive = false;
        emit GameEnded(block.timestamp);
    }
```

### [L-#] Incorrect value of remaining game time

**Note**: The documentation does not clarify whether the game owner is allowed to end the game before the scheduled end time. In the scenario presented below, the owner of the game contract can end the game before the specified time limit.

## Summary

When the game owner ends the game before the initial time limit, calling the `EggHuntGame::getTimeRemaining` function still returns the remaining time based on the initial duration. This is incorrect, as the game is no longer active. The function should reflect that the game has ended and return zero instead.

## Vulnerability Details

When the owner starts the game, he sets its duration but can also choose to end it early. Any user can call the `EggHuntGame::getTimeRemaining` function to check how much time is left. However, if the owner ends the game early, this function still returns the time remaining based on the original duration, which is incorrect.

## Impact

The possibility that the `EggHuntGame::getTimeRemaining` function may return an invalid value.

## Proof of Code

Add the following code to the `EggHuntGameTest.t.sol` file within the `EggGameTest` contract.

```solidity
    function testTimeRemaining() public {
        uint256 duration = 150;
        uint256 currentTime = block.timestamp;

        game.startGame(duration);
        vm.warp(currentTime + 50);
        game.endGame();
        assertNotEq(game.getTimeRemaining(), 0);
    }
```

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

To ensure the function always returns the correct value, it should also check whether the game is still active. Recommended changes in the `EggHuntGame` contract.

```diff
    function getTimeRemaining() external view returns (uint256) {
-       return block.timestamp >= endTime ? 0 : endTime - block.timestamp;
+       return
+           block.timestamp >= endTime || !gameActive
+               ? 0
+               : endTime - block.timestamp;
    }
```

### [I-#] The `EggHuntGame::getGameStatus` function includes unused code block

## Summary

The first condition in the `EggHuntGame::getGameStatus` function’s conditional statement can never be true.

## Vulnerability Details

```solidity
    function getGameStatus() external view returns (string memory) {
        if (gameActive) {
@>          if (block.timestamp < startTime) {
@>              return "Game not started yet";
            } else if (
                block.timestamp >= startTime && block.timestamp <= endTime
            ) {
                return "Game is active";
            } else {
                return "Game time elapsed";
            }
        } else {
            return "Game is not active";
        }
    }
```

This condition can only be met when the game is active. The `startTime` variable is set by the game owner when he call the `EggHuntGame::startGame` function, which assigns it the current block's timestamp. This value is never decreased during the game, so the current block's timestamp can never be less than value of `startTime` variable.

## Impact

The `EggHuntGame::getGameStatus` function performs unnecessary calculations during execution.

## Tools Used

- Manual Review
- Foundry

## Recommended Mitigation

Unused code should be removed from the code base.

File `EggHuntGame.sol`:

```diff
    function getGameStatus() external view returns (string memory) {
        if (gameActive) {
-           if (block.timestamp < startTime) {
-               return "Game not started yet";
-           } else if (
-               block.timestamp >= startTime && block.timestamp <= endTime
            if (block.timestamp >= startTime && block.timestamp <= endTime) {
                return "Game is active";
            } else {
                return "Game time elapsed";
            }
        } else {
            return "Game is not active";
        }
    }
```
