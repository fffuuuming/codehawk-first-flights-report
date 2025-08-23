## [2025-07-last-man-standing](https://codehawks.cyfrin.io/c/2025-07-last-man-standing)

### [✅ Root + Impact: Missing Grace Period Check in `claimThrone()` Allows Post-Expiry Claims](https://codehawks.cyfrin.io/c/2025-07-last-man-standing/s/114)

Judgement:
- `Validated`
- Severity: `High`
- Assigned Finding Tags: `Game::claimThrone can still be called regardless of the grace period`

#### Description
- According to the documentation, once the grace period expires, no other player should be able to claim the throne, and the current king should be finalized as the winner.
- However, the `claimThrone()` function does not verify whether the grace period has expired. Since the `gameNotEnded` modifier only checks that `gameEnded` is false, players can continue to claim the throne even after the round should have ended — as long as `declareWinner()` has not yet been called.

```solidity
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
@>  // lack the check of whether the period has expired
    uint256 sentAmount = msg.value;
    uint256 previousKingPayout = 0;
    uint256 currentPlatformFee = 0;
    uint256 amountToPot = 0;

    // Calculate platform fee
    currentPlatformFee = (sentAmount * platformFeePercentage) / 100;

    // Defensive check to ensure platformFee doesn't exceed available amount after previousKingPayout
    if (currentPlatformFee > (sentAmount - previousKingPayout)) {
        currentPlatformFee = sentAmount - previousKingPayout;
    }
    platformFeesBalance = platformFeesBalance + currentPlatformFee;

    // Remaining amount goes to the pot
    amountToPot = sentAmount - currentPlatformFee;
    pot = pot + amountToPot;

    // Update game state
    currentKing = msg.sender;
    lastClaimTime = block.timestamp;
    playerClaimCount[msg.sender] = playerClaimCount[msg.sender] + 1;
    totalClaims = totalClaims + 1;

    // Increase the claim fee for the next player
    claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100;

    emit ThroneClaimed(
        msg.sender,
        sentAmount,
        claimFee,
        pot,
        block.timestamp
    );
}
```
#### Risk

**Likelihood**: High
- This absence of proper state management within the contract allows every attempt to claim the throne to succeed after the period expires, as long as `declareWinner()` hasn't been called yet

**Impact**: High
- **Violation of Promised Rewards and Game Logic**: Players who should rightfully win after the grace period may be overtaken, breaking the protocol’s core assumption that the last king within the grace period is the winner.
- **Player Financial Loss**: The rightful winner loses access to the accumulated pot, resulting in tangible economic harm and undermining trust in the fairness of the game.


#### Proof of Concept
Add the following test, then run this command: `forge test -vv --match-test testClaimThroneAlthoughExpired`

```solidity
function testClaimThroneAlthoughExpired() public {
    console2.log("player1 claims the throne");
    uint256 claimFee = game.claimFee();
    vm.prank(player1);
    game.claimThrone{value: claimFee}();

    console2.log("Wait for grace period to expire");
    skip(GRACE_PERIOD + 1);
    console2.log("Remaining time: ", game.getRemainingTime());

    claimFee = game.claimFee();
    vm.prank(player2);
    game.claimThrone{value: claimFee}();
    console2.log("player2 successfully claims the throne although grace period has expired");
}
```

PoC Results:
```
forge test -vv --match-test testClaimThroneAlthoughExpired
[⠊] Compiling...
[⠔] Compiling 1 files with Solc 0.8.29
[⠒] Solc 0.8.29 finished in 473.13ms
Compiler run successful!

Ran 1 test for test/Game.t.sol:GameTest
[PASS] testClaimThroneAlthoughExpired() (gas: 206162)
Logs:
  player1 claims the throne
  Wait for grace period to expire
  Remaining time:  0
  player2 successfully claims the throne although grace period has expired

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.07ms (1.52ms CPU time)

Ran 1 test suite in 233.18ms (6.07ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

#### Recommended Mitigation
Add the check that whether the grace period expires

```diff
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
+   require(
+       block.timestamp <= lastClaimTime + gracePeriod,
+       "Game: Grace period has expired."
+   );
    uint256 sentAmount = msg.value;
    uint256 previousKingPayout = 0;
    uint256 currentPlatformFee = 0;
    uint256 amountToPot = 0;

    // Calculate platform fee
    currentPlatformFee = (sentAmount * platformFeePercentage) / 100;

    // Defensive check to ensure platformFee doesn't exceed available amount after previousKingPayout
    if (currentPlatformFee > (sentAmount - previousKingPayout)) {
        currentPlatformFee = sentAmount - previousKingPayout;
    }
    platformFeesBalance = platformFeesBalance + currentPlatformFee;

    // Remaining amount goes to the pot
    amountToPot = sentAmount - currentPlatformFee;
    pot = pot + amountToPot;

    // Update game state
    currentKing = msg.sender;
    lastClaimTime = block.timestamp;
    playerClaimCount[msg.sender] = playerClaimCount[msg.sender] + 1;
    totalClaims = totalClaims + 1;

    // Increase the claim fee for the next player
    claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100;

    emit ThroneClaimed(
        msg.sender,
        sentAmount,
        claimFee,
        pot,
        block.timestamp
    );
}
```

---

### [✅ Root + Impact: Inverted currentKing Check Blocks All Players from Claiming the Throne](https://codehawks.cyfrin.io/c/2025-07-last-man-standing/s/76)

Judgement:
- `Validated`
- Severity: `Medium`
- Assigned Finding Tags: ```Game::claimThrone `msg.sender == currentKing` check is busted```
#### Description
- The function `claimThrone()` has access control which it declares that current king can't re-claim.
- However, the actual implementation logic is inverted, it allows only the current king to call `claimThrone()`, while blocking all other players from participating. 

```solidity
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    @> // below check blocks all player from claiming throne
    require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");

    ...
}
```

#### Risk

**Likelihood**: High
- The incorrect logic is already inside the contract and would immediately block all other players from interacting.

**Impact**: High
- It prevents the primary functionality of the game — no one can claim the throne since initial current king is `address(0)`

#### Proof of Concept
Add the following test, then run the command: `forge test --match-test testclaimThroneFail`

```solidity
function testclaimThroneFail() public {
    vm.prank(player1);
    vm.expectRevert("Game: You are already the king. No need to re-claim.");
    game.claimThrone{value: INITIAL_CLAIM_FEE}();
}
```

#### Recommended Mitigation
Check if `msg.sender != currentKing` in `claimThrone()` to implement the correct access control

```diff
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
+   require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
-   require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");

    ...
}
```

### [✅ Root + Impact: Missing Previous King Payout Breaks Incentives and Cause Player Financial Loss](https://codehawks.cyfrin.io/c/2025-07-last-man-standing/s/95)

Judgement:
- `Validated`
- Severity: `Medium`
- Assigned Finding Tags: `Missing Previous King Payout Functionality`

#### Description
- According to the comment of `claimThrone()`, previous king will receive next king's small portion of the claim fee
- However, although `previousKingPayout` is declared and initialized to `0`, it is never set and transfered to previous king

> Note that below code has fixed the incorrect access control `require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");`, in order to make the core basic functionality work
```solidity
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");

    uint256 sentAmount = msg.value;
    uint256 previousKingPayout = 0;
    uint256 currentPlatformFee = 0;
    uint256 amountToPot = 0;

    // Calculate platform fee
    currentPlatformFee = (sentAmount * platformFeePercentage) / 100;
@>  // No calculation for previous king's payout
    // Defensive check to ensure platformFee doesn't exceed available amount after previousKingPayout
    if (currentPlatformFee > (sentAmount - previousKingPayout)) {
        currentPlatformFee = sentAmount - previousKingPayout;
    }
    platformFeesBalance = platformFeesBalance + currentPlatformFee;
@>  // No payout send to previous king
    // Remaining amount goes to the pot
    amountToPot = sentAmount - currentPlatformFee;
    pot = pot + amountToPot;

    // Update game state
    currentKing = msg.sender;
    lastClaimTime = block.timestamp;
    playerClaimCount[msg.sender] = playerClaimCount[msg.sender] + 1;
    totalClaims = totalClaims + 1;

    // Increase the claim fee for the next player
    claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100;

    emit ThroneClaimed(
        msg.sender,
        sentAmount,
        claimFee,
        pot,
        block.timestamp
    );
}
```

#### Risk

**Likelihood**: High
- This logic error is already inside the contract and will happen each time a new king claims the throne
 
**Impact**: High
- **Financial Loss for Players**: Dethroned kings incurs a cost when claiming the throne but receive no compensation, resulting in an unfair financial disadvantage.
- **Credibility Degradation**: The mismatch between documented game mechanics and actual implementation undermines user trust and damages the project’s reputation.
- **Broken Incentive Structure**: Without rewards for holding the throne, players have no motivation to participate early or repeatedly, leading to poor engagement and a shift toward last-minute sniping strategies


#### Proof of Concept
Add the following test, then run the command: `forge test -vv --match-test testNoRecevivePayoutFromNextPlayer`
```solidity
function testNoRecevivePayoutFromNextPlayer() public {
    console2.log("player1 claims the throne");
    uint256 claimFee = game.claimFee();
    vm.prank(player1);
    game.claimThrone{value: claimFee}();

    console2.log("player1 balance: ", address(this).balance);

    console2.log("player2 claims the throne");
    claimFee = game.claimFee();
    vm.prank(player2);
    game.claimThrone{value: game.claimFee()}();

    console2.log("player1 balance: ", address(this).balance);
    console2.log("player1 doesn't receive the payout from player2");
}
```

PoC Results:
```
forge test -vv --match-test testNoRecevivePayoutFromNextPlayer
[⠊] Compiling...
[⠑] Compiling 1 files with Solc 0.8.29
[⠘] Solc 0.8.29 finished in 668.41ms
Compiler run successful!

Ran 1 test for test/Game.t.sol:GameTest
[PASS] testNoRecevivePayoutFromNextPlayer() (gas: 200452)
Logs:
  player1 claims the throne
  player1 balance:  79228162514264337593543950335
  player2 claims the throne
  player1 balance:  79228162514154337593543950335
  player1 doesn't receive the payout from player2

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.37ms (1.50ms CPU time)

Ran 1 test suite in 324.91ms (7.37ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

#### Recommended Mitigation
- Add a state variable `previousKingPayoutPercentage` and initialize it inside constrcutor, ensure that `_previousKingRewardPercentage + _platformFeePercentage <= 100`
- insert the logic of payout for previsou king inisde `claimThrone()`
```diff
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");

    uint256 sentAmount = msg.value;
    uint256 previousKingPayout = 0;
    uint256 currentPlatformFee = 0;
    uint256 amountToPot = 0;

+   address previousKing = currentKing;
    // Calculate platform fee
    currentPlatformFee = (sentAmount * platformFeePercentage) / 100;
+   previousKingPayout = (sentAmount * previousKingPayoutPercentage) / 100;

    // Defensive check to ensure platformFee doesn't exceed available amount after previousKingPayout
    if (currentPlatformFee > (sentAmount - previousKingPayout)) {
        currentPlatformFee = sentAmount - previousKingPayout;
    }
    platformFeesBalance = platformFeesBalance + currentPlatformFee;

+   (bool success, ) = payable(previousKing).call{value: previousKingPayout}("");
+       require(success, "Game: Failed to pay for previous king.");
        
    // Remaining amount goes to the pot
+   amountToPot = sentAmount - currentPlatformFee - previousKingPayout;
-   amountToPot = sentAmount - currentPlatformFee;
    pot = pot + amountToPot;

    // Update game state
    currentKing = msg.sender;
    lastClaimTime = block.timestamp;
    playerClaimCount[msg.sender] = playerClaimCount[msg.sender] + 1;
    totalClaims = totalClaims + 1;

    // Increase the claim fee for the next player
    claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100;

    emit ThroneClaimed(
        msg.sender,
        sentAmount,
        claimFee,
        pot,
        block.timestamp
    );
}
```

---

### [❌ Root + Impact: Checks-Effects-Interactions Violation in withdrawWinnings() Enables Read-Only Reentrancy and Stale State Exposure](https://codehawks.cyfrin.io/c/2025-07-last-man-standing/s/122)

Judgement:
- `Invalid` Reason: Non-acceptable severity
- Assigned Finding Tags: `Game::withdrawWinnings reentrancy`
- We only audit the current code in scope. We cannot make speculation with respect to how this codebase will evolve in the future. For now there is a nonReentrant modifier which mitigates any reentrancy. CEI is a good practice, but it's not mandatory. Informational

#### Description
- In the `withdrawWinnings()` function, a manual `nonReentrant` modifier is used to block standard reentrancy attacks.
- However, the function violates the **Checks-Effects-Interactions pattern**. Specifically, it transfers ETH before clearing `pendingWinnings[msg.sender]`, which allows a read-only reentrancy vector.
- Any external observer (e.g., oracles or smart contracts) querying `pendingWinnings[msg.sender]` during the call may observe outdated values and mistakenly believe the user still has withdrawable balance.

```solidity
function withdrawWinnings() external nonReentrant {
    uint256 amount = pendingWinnings[msg.sender];
    require(amount > 0, "Game: No winnings to withdraw.");

    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Game: Failed to withdraw winnings.");
@>  // violates Checks-Effects-Interactions pattern: update pendingWinnings after transfering ETH
    pendingWinnings[msg.sender] = 0;

    emit WinningsWithdrawn(msg.sender, amount);
}
```
#### Risk

**Likelihood**: Medium
- It requires external protocols to naively use `pendingWinnings` as real-time collateral or creditworthiness data

**Impact**: High
- External contracts can be misled into granting users more credit, assets, or privileges than they deserve, based on stale `pendingWinnings` values


#### Proof of Concept
Add the following exploit contract and test, then run this command: `forge test -vv --match-test testWithdrawWinningsReentrancy`

```solidity
// test
function testWithdrawWinningsReentrancy() public {
    GameWithdrawWinningsReentrancy exploiter = new GameWithdrawWinningsReentrancy{value: 1 ether}(game);
    exploiter._gameClaimThrone();
    skip(GRACE_PERIOD + 1);
    game.declareWinner();

    console2.log("player1 penddingWinning before withdraw: ", game.pendingWinnings(address(exploiter)));
    console2.log("player1 withdraw winnings");
    exploiter.exploit();
    console2.log("player1 penddingWinning druing callback:", exploiter.pendingWinningDruingCallback());
    console2.log("player1 penddingWinning after withdraw: ", game.pendingWinnings(address(exploiter)));
}


// exploit contract
contract GameWithdrawWinningsReentrancy is Test {
    Game private _game;
    uint256 public pendingWinningDruingCallback;

    constructor(Game game) payable {
        _game = game;
    }

    function _gameClaimThrone() public {
        _game.claimThrone{value: _game.claimFee()}();
    }

    function exploit() public {
        _game.withdrawWinnings();
    }

    receive () external payable {
        pendingWinningDruingCallback = _game.pendingWinnings(address(this));
    }
}
```

PoC Results:
```
forge test -vv --match-test testWithdrawWinningsReentrancy
[⠊] Compiling...
[⠔] Compiling 1 files with Solc 0.8.29
[⠒] Solc 0.8.29 finished in 488.84ms
Compiler run successful!

Ran 1 test for test/Game.t.sol:GameTest
[PASS] testWithdrawWinningsReentrancy() (gas: 1568155)
Logs:
  player1 penddingWinning before withdraw:  95000000000000000
  player1 withdraw winnings
  player1 penddingWinning druing callback: 95000000000000000
  player1 penddingWinning after withdraw:  0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.34ms (1.07ms CPU time)

Ran 1 test suite in 218.18ms (5.34ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

#### Recommended Mitigation
Follow the Checks-Effects-Interactions , update `pendingWinnings[msg.sender]` before transferring ETH
```diff
function withdrawWinnings() external nonReentrant {
    uint256 amount = pendingWinnings[msg.sender];
    require(amount > 0, "Game: No winnings to withdraw.");
+   pendingWinnings[msg.sender] = 0;
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Game: Failed to withdraw winnings.");

-   pendingWinnings[msg.sender] = 0;

    emit WinningsWithdrawn(msg.sender, amount);
}
```
