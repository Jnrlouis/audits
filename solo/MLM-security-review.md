# Introduction

A time-boxed security review of the **MLM** protocol was done by **jnrlouis**, with a focus on the security aspects of the application's smart contracts implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **jnrlouis**

**jnrlouis**, is an independent junior smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Check his previous work [here](https://github.com/jnrlouis/audits) or reach out on Twitter [@Lo0_0u](https://twitter.com/Lo0_0u).

# About **MLM**

MLM is a Multilevel Marketing protocol where users can refer other users and earn rewards via referrals. It uses the MJC token, which is a regular ERC20 token. It is expected to be deployed on the Binance Smart Chain mainnet.

## Observations

MLM offers and rewards only the users 9 *parents*

## Privileged Roles & Actors

*owner* - The contract deployer is at the top of the hierachy and their address is a *parent* to every user

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**_review commit hash_ - [95783d43ea2bbfba51c03ca1bc2259497d2a7aac](https://github.com/Maitrayee94/MLM_Contract/commit/95783d43ea2bbfba51c03ca1bc2259497d2a7aac)**


### Scope

The following smart contracts were in scope of the audit:

- `MJC_Token.sol`
- `StakeandSale.sol`

---

# Findings Summary

| ID            | Title                                                     | Severity | Status |
| ------        | -----------------------                                   | -------- | ------ |
| [C-01](#C-01) | Users can lose funds when staking with same ID            | Critical | TBD    |
| [C-02](#C-02) | Wrong decimal conversion can lead to loss of funds        | Critical | TBD    |
| [C-03](#C-03) | Users cannot unstake tokens                               | Critical | TBD    |
| [H-01](#H-01) | Unstaking with a particular `stakeId` deletes all stake   | High     | TBD    |
| [H-02](#H-02) | Variables updated in `memory`                             | High     | TBD    |
| [H-03](#H-03) | Lack of input validation                                  | High     | TBD    |
| [M-01](#M-01) | Token Integer design                                      | Medium   | TBD    |
| [M-02](#M-02) | `Owner` and `Fee` addresses cannot be changed             | Medium   | TBD    |
| [L-01](#L-01) | Empty Function Body - Consider commenting why             | Low      | 1      |
| [L-02](#L-02) | Unsafe ERC20 operation(s)                                 | Low      | 10     |
| [NC-01](#NC-01) | Return values of `approve()` not checked                | NC       | 6      |
| [NC-02](#NC-02) | Event is missing `indexed` fields                       | NC       | 8      |
| [NC-03](#NC-03) | Functions not used internally could be marked external  | NC       | 4      |


## Gas Optimizations

| |Issue|Instances|
|-|:-|:-:|
| [GAS-1](#GAS-1) | Use `selfbalance()` instead of `address(this).balance` | 2 |
| [GAS-2](#GAS-2) | Use assembly to check for `address(0)` | 7 |
| [GAS-3](#GAS-3) | Using bools for storage incurs overhead | 2 |
| [GAS-4](#GAS-4) | Use calldata instead of memory for function arguments that do not get mutated | 7 |
| [GAS-5](#GAS-5) | For Operations that will not overflow, you could use unchecked | 83 |
| [GAS-6](#GAS-6) | Use Custom Errors | 44 |
| [GAS-7](#GAS-7) | Don't initialize variables with default value | 6 |
| [GAS-8](#GAS-8) | Long revert strings | 23 |
| [GAS-9](#GAS-9) | Functions guaranteed to revert when called by normal users can be marked `payable` | 5 |
| [GAS-10](#GAS-10) | `++i` costs less gas than `i++`, especially when it's used in `for`-loops (`--i`/`i--` too) | 6 |
| [GAS-11](#GAS-11) | Using `private` rather than `public` for constants, saves gas | 7 |
| [GAS-12](#GAS-12) | Use != 0 instead of > 0 for unsigned integer comparison | 4 |
| [GAS-13](#GAS-13) | `internal` functions not called by the contract should be removed | 3 |


___

# Detailed Findings

# <a name="C-01"></a>[C-01] Users can lose funds when staking with same ID

## Description

`stakeTokens` in `StakeandSale.sol` is a public function used to stake funds. This function takes the following parameters: `tokenAmount_` , `stakingDuration_` , `teamSize_` and `id`.

The issue here is that the `id` is not dynamically generated by the contract, but an expected input from the user. 

```solidity

    users[msg.sender][id] = User({
    stakedAmount: requiredAmount,
    stakingEndTime: stakingEndTime,
    StartDate: StartDate,
    teamSize: teamSize_  // Corrected field name to match the struct
});
```
If a user calls this function with the same `id`, his previous stake would be overwritten and lost forever.

## Recommendations
The `id` should be generated dynamically at the smart contract level, as it is accessible to users.

# <a name="C-02"></a>[C-02] Wrong decimal conversion can lead to loss of funds

## Description

`stakeTokens` in `StakeandSale.sol` is a public function used to stake funds. This function takes the following parameters: `tokenAmount_` , `stakingDuration_` , `teamSize_` and `id`.

The `tokenAmount_` is expected to represent 1 unit of the token, hence it is multiplied by `1 ether` (or 10 ^ 18).

```solidity

135    uint256 requiredAmount = tokenAmount_ * 1 ether;

154    totalStaked += requiredAmount * 1 ether;
```
`requiredAmount` is the converted form of `tokenAmount_` as it has already been multiplied by `1 ether`, but when calculating the `totalStaked`, the `requiredAmount` was once again multiplied by `1 ether`, increasing the value of the token by aan additional `10^18`.

This is also done in the `TotalTokenStaked` function as the `totalStakedByUser` would have the same error.

## Recommendations
The `totalStaked` and `totalStakedByUser` should not be multiplied by `1 ether`.

```solidity

-- 154    totalStaked += requiredAmount * 1 ether;

++ 154    totalStaked += requiredAmount;

-- 193        totalStakedByUser += user.stakedAmount * 1 ether;

++ 193        totalStakedByUser += user.stakedAmount;
```

# <a name="C-03"></a>[C-03] Users cannot unstake tokens

## Description

`stakeTokens` in `StakeandSale.sol` stores the stake information in a `users` mapping.

```solidity

64    mapping(address => mapping(uint256 => User)) public users;

145    users[msg.sender][id] = User({
146        hstakedAmount: requiredAmount,
147        stakingEndTime: stakingEndTime,
148        StartDate: StartDate,
149        teamSize: teamSize_  // Corrected field name to match the struct
150    });
```

The `unstakeTokens` function attempts to get the `stakeAmount` from a different mapping that wasn't used in the `stakeTokens` function.

```solidity

170        User storage user = userStaking[msg.sender][stakeId_];
171        uint256 stakedAmount = user.stakedAmount;
```
Since the `userStaking` was not updated in the `stakeTokens` function, the `stakedAmount` would always return 0. Users wouldn't be able to unstake and they'll have their funds stuck forever.


## Recommendations
Either use the `users` mapping in the `unstakeTokens` function, or update the `userStaking` mapping in the `stakeTokens` function.

# <a name="H-01"></a>[H-01] Unstaking with a particular `stakeId` deletes all stake

## Description

`stakeTokens` in `StakeandSale.sol` is designed in such a way that users can have multiple stakes with different `stakeId_`'s, this isn't considered when unstaking, as attempting to unstake just a particular Id would cause the user to lose all other stakes.

```solidity

179        delete userStaking[msg.sender];
```

The `unstakeTokens` function deletes the users entire array which may contain information from multiple stakeIds.

```solidity
62    mapping(address => User[]) public userStaking;
```

## Recommendations
Other than deleting (resetting) the entire array, only the index of the `stakeId_` should be deleted.

```solidity

-- 179        delete userStaking[msg.sender];
++ 179        delete userStaking[msg.sender][stakeId_];
```

# <a name="H-02"></a>[H-02] Variables updated in `memory`

## Description

The `DirectStakeJoining` in `StakeandSale.sol` implements a subscription model. 

```solidity

226         StakeSubscription memory subscription = stakeSubscription[msg.sender];
227    require(subscription.tokenAmount == 0, "User already has a subscription");
228    uint256 amount = _tokenAmount * 1 ether;
229    subscription.tokenAmount = amount;
230    subscription.parent = _referreladdress; 
```

The `subscription` uses `memory`, so all the variables updated are lost after the function call. This means after a user transfers token to subscribe, record of the subscription and subscription amount would be lost.


## Recommendations
Use `storage` instead of `memory`

```solidity
-- 226         StakeSubscription memory subscription = stakeSubscription[msg.sender];
++ 226         StakeSubscription storage subscription = stakeSubscription[msg.sender];
```

# <a name="H-03"></a>[H-03] Lack of input validation

## Description

The `DirectStakeJoining` and `buyTokens` in `StakeandSale.sol` do not have enough input validation, thereby allowing users to subscribe without paying anf fees or any token. Since both functions are user facing functions with no restriction, users can call the function and parse in `0` as the `tokenAmount_` and as the `fee` in the `buyTokens` function.

Since the `MJC_Token.sol` allow 0 token transfer, this would pass all the checks without error, parents on the top of the ladder would receive no reward, and the protocol would receive no fees.


## Recommendations
The `_fees` should not be a parameter on the `buyTokens` function, rather, it should be a, access controlled value set in the contract by the `owner`. The `_tokenAmount` should be validated to ensure the value is greater than 0. 

```solidity

   function buyTokens(address _referrer, uint256 _tokenAmount, uint256 _tier, uint256 _fees) external {
    require(
        _tier == ZeroUSD || _tier == FiftyUSD || _tier == HundreadUSD || _tier == TwoHundreadUSD
            || _tier == FiveHundreadUSD || _tier == ThousandUSD,
        "Invalid tier value"
    );
++  require(_tokenAmount > 0, "Invalid Amount");
    uint256 amount = _tokenAmount * 1 ether;
```

# <a name="M-01"></a>[M-01] Token Integer design

## Description

`StakeandSale.sol` is designed in such a way that only token integers can be used, this means, you can only work with whole numbers and decimals are not allowed.

Solidity does not support floating numbers, so as a standard, tokens are designed with a number of the decimals, and the contract operates with the smallest unit. For a token with 18 decimals, the contract numbers are represented in `wei` and then converted to the respective decimal on the frontend.

This contract does the conversion at the contract level and it can be really problematic for users that prefer to operate directly with the contract.

```solidity
    uint256 requiredAmount = tokenAmount_ * 1 ether;
```
Also, this conversion means that only integers can be used. This can be confusing as it is against the norm for most users.

## Recommendations
Working with the smallest unit of the tokens at the contract level is advised. Not only would it remove confusion for users, but it would also remove the integer restriction, thus improving user experience.


# <a name="M-02"></a>[M-02] `Owner` and `Fee` addresses cannot be changed

## Description

The `owner` and `fee` addresses in `StakeandSale.sol` are fixed and cannot be changed. In a scenerio where the account of either addresses are compromised, there is no current way to change these addresses, and since these are critical parameters in the protocol, it can cause the users to lose trust and the protocol to lose funds.

## Recommendations
Add an access controlled function where these addresses can be set. It is recommended to implement a two-step ownership transfer where separate functions are used for a two-step address change:

- First, you approve a new address as a "pendingOwner". This step allows you to set the stage for changing the ownership.

- Next, a transaction from the "pendingOwner" address is needed to claim the pending ownership change. This step is crucial as it ensures that only the correct address can complete the process.

The benefit of this approach is that it mitigates risk. If an incorrect address is mistakenly used in step (1), you can easily fix it by re-approving the correct address. This flexibility gives you the opportunity to rectify any errors before they become permanent.

Additionally, it's worth considering adding a time-delay for such sensitive actions. By introducing a timer or waiting period, you can provide an extra layer of security and give yourself time to double-check the details before finalizing the process.

Furthermore, it is advisable to use a multisig owner address instead of an Externally Owned Account (EOA) as the minimum requirement. This adds an extra level of protection by involving multiple authorized parties in the ownership process, ensuring better control and security.

Implementing these measures enhances the safety and integrity of the ownership change process.


# <a name="L-01"></a>[L-01] Empty Function Body - Consider commenting why

*Instances (1)*:
```solidity
File: mlm/MJC_Token.sol

197:     function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual { }

```

# <a name="L-02"></a>[L-02] Unsafe ERC20 operation(s)

*Instances (10)*:
```solidity
File: mlm/MJC_Token.sol

551:         BEP20(tokenAddress).transfer(owner(), tokenAmount);

```

```solidity
File: mlm/StakeandSale.sol

159:     token.transferFrom(msg.sender, address(this), requiredAmount);

182:         token.transfer(msg.sender, stakedAmount);

240:         token.approve(address(this), amount);

241:         token.transferFrom(msg.sender, address(this), amount);

253:             token.transferFrom(address(this), parent_addr, reward_amount);

309:     token.approve(address(this), amount);

318:     require(token.transferFrom(msg.sender, address(this), amount), "Token transfer failed");

321:     require(token.transfer(fees_address, fee), "Fee transfer failed");

336:        require(token.transfer(parent_addr, reward_amount), "Reward transfer failed");

```
# <a name="NC-01"></a>[NC-01] Return values of `approve()` not checked
Not all IERC20 implementations `revert()` when there's a failure in `approve()`. The function signature has a boolean return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually approving anything

*Instances (6)*:
```solidity
File: mlm/MJC_Token.sol

121:         _approve(_msgSender(), spender, amount);

131:         _approve(sender, _msgSender(), currentAllowance - amount);

137:         _approve(_msgSender(), spender, _allowances[_msgSender()][spender] + addedValue);

144:         _approve(_msgSender(), spender, currentAllowance - subtractedValue);

212:         _approve(account, _msgSender(), currentAllowance - amount);

487:         approve(spender, amount);

```

# <a name="NC-02"></a>[NC-02] Event is missing `indexed` fields
Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

*Instances (8)*:
```solidity
File: mlm/MJC_Token.sol

17:     event Transfer(address indexed from, address indexed to, uint256 value);

19:     event Approval(address indexed owner, address indexed spender, uint256 value);

20:     event botAddedToBlacklist(address account);

21:     event botRemovedFromBlacklist(address account);

```

```solidity
File: mlm/StakeandSale.sol

88:     event TokensStaked(address indexed user, uint256 amount, uint256 stakingEndTime, uint256 id);

91:     event TokensUnstaked(address indexed user, uint256 amount);

97:     event TokenBought(address indexed buyer, uint256 amount, uint256 tier);

99:     event DirectEntry(address indexed buyer, uint256 amount);

```

# <a name="NC-03"></a>[NC-03] Functions not used internally could be marked external

*Instances (4)*:
```solidity
File: mlm/StakeandSale.sol

133:     function stakeTokens(uint256 tokenAmount_, uint256 stakingDuration_, uint256 teamSize_, uint256 id) public {

168:     function unstakeTokens(uint256 stakeId_) public {

188:     function TotalTokenStaked(address userAddress) public view returns (uint256) {

377: function totalRewardsReceived(address userAddress) public view returns (uint256) {

```

# <a name="GAS-1"></a>[GAS-1] Use `selfbalance()` instead of `address(this).balance`
Use assembly when getting a contract's balance of ETH.

You can use `selfbalance()` instead of `address(this).balance` when getting your contract's balance of ETH to save gas.
Additionally, you can use `balance(address)` instead of `address.balance()` when getting an external contract's balance of ETH.

*Saves 15 gas when checking internal balance, 6 for external*

*Instances (2)*:
```solidity
File: mlm/MJC_Token.sol

288:         require(address(this).balance >= amount, "Address: insufficient balance");

308:         require(address(this).balance >= value, "Address: insufficient balance for call");

```

# <a name="GAS-2"></a>[GAS-2] Use assembly to check for `address(0)`
*Saves 6 gas per instance*

*Instances (7)*:
```solidity
File: mlm/MJC_Token.sol

151:         require(sender != address(0), "ERC20: transfer from the zero address");

152:         require(recipient != address(0), "ERC20: transfer to the zero address");

166:         require(account != address(0), "ERC20: mint to the zero address");

176:         require(account != address(0), "ERC20: burn from the zero address");

190:         require(owner != address(0), "ERC20: approve from the zero address");

191:         require(spender != address(0), "ERC20: approve to the zero address");

540:         require(newOwner != address(0), "Ownable: new owner is the zero address");

```

# <a name="GAS-3"></a>[GAS-3] Using bools for storage incurs overhead
Use uint256(1) and uint256(2) for true/false to avoid a Gwarmaccess (100 gas), and to avoid Gsset (20000 gas) when changing from ‘false’ to ‘true’, after having been ‘true’ in the past. See [source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27).

*Instances (2)*:
```solidity
File: mlm/MJC_Token.sol

79:     mapping(address => bool) public _isBlackListedBot;

```

```solidity
File: mlm/StakeandSale.sol

77:     mapping(address => bool) public ownerReferred;

```

# <a name="GAS-4"></a>[GAS-4] Use calldata instead of memory for function arguments that do not get mutated
Mark data types as `calldata` instead of `memory` where possible. This makes it so that the data is not automatically loaded into memory. If the data passed into the function does not need to be changed (like updating values in an array), it can be passed in as `calldata`. The one exception to this is if the argument must later be passed into another function that takes an argument that specifies `memory` storage.

*Instances (7)*:
```solidity
File: mlm/MJC_Token.sol

86:     constructor (string memory name_, string memory symbol_) {

86:     constructor (string memory name_, string memory symbol_) {

451:         bytes memory data

471:         bytes memory data

485:         bytes memory data

574:         string memory name_,

575:         string memory symbol_,

```

# <a name="GAS-5"></a>[GAS-5] For Operations that will not overflow, you could use unchecked

*Instances (83)*:
```solidity
File: mlm/MJC_Token.sol

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

68:         this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691

131:         _approve(sender, _msgSender(), currentAllowance - amount);

137:         _approve(_msgSender(), spender, _allowances[_msgSender()][spender] + addedValue);

144:         _approve(_msgSender(), spender, currentAllowance - subtractedValue);

158:         _balances[sender] = senderBalance - amount;

159:         _balances[recipient] += amount;

170:         _totalSupply += amount;

171:         _balances[account] += amount;

182:         _balances[account] = accountBalance - amount;

183:         _totalSupply -= amount;

212:         _approve(account, _msgSender(), currentAllowance - amount);

224:             uint256 c = a + b;

236:             uint256 c = a - b;

246:             uint256 c = a * b;

247:             require(c / a == b, "SafeMath: multiplication overflow");

258:             uint256 c = a / b;

296:       return functionCall(target, data, "Address: low-level call failed");

304:         return functionCallWithValue(target, data, value, "Address: low-level call with value failed");

309:         require(isContract(target), "Address: call to non-contract");

317:         return functionStaticCall(target, data, "Address: low-level static call failed");

321:         require(isContract(target), "Address: static call to non-contract");

329:         return functionDelegateCall(target, data, "Address: low-level delegate call failed");

333:         require(isContract(target), "Address: delegate call to non-contract");

581:         _mint(tokenOwner, initialBalance_*10**uint256(decimals_));

581:         _mint(tokenOwner, initialBalance_*10**uint256(decimals_));

581:         _mint(tokenOwner, initialBalance_*10**uint256(decimals_));

```

```solidity
File: mlm/StakeandSale.sol

4: import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

4: import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

4: import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

4: import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

11:     address immutable owner; // The address of the contract owner

11:     address immutable owner; // The address of the contract owner

13:     ERC20 immutable token; // Address of the ERC20 token being staked

13:     ERC20 immutable token; // Address of the ERC20 token being staked

31:         uint256 stakedAmount; // Amount of tokens staked

31:         uint256 stakedAmount; // Amount of tokens staked

32:         uint256 stakingEndTime; // Time when staking ends

32:         uint256 stakingEndTime; // Time when staking ends

34:         uint256 teamSize; // Size of the staking team

34:         uint256 teamSize; // Size of the staking team

66:     mapping(address => uint256) public userCount; // Count of stakes per user

66:     mapping(address => uint256) public userCount; // Count of stakes per user

135:     uint256 requiredAmount = tokenAmount_ * 1 ether;

141:         uint256 stakingEndTime = block.timestamp + stakingDuration_ * 1 days;

141:         uint256 stakingEndTime = block.timestamp + stakingDuration_ * 1 days;

149:     teamSize: teamSize_  // Corrected field name to match the struct

149:     teamSize: teamSize_  // Corrected field name to match the struct

151:     userCount[msg.sender]++;

151:     userCount[msg.sender]++;

154:     totalStaked += requiredAmount * 1 ether;

154:     totalStaked += requiredAmount * 1 ether;

191:     for (uint256 id = 101; id <= 100 + userCount[userAddress]; id++) {

191:     for (uint256 id = 101; id <= 100 + userCount[userAddress]; id++) {

191:     for (uint256 id = 101; id <= 100 + userCount[userAddress]; id++) {

193:         totalStakedByUser += user.stakedAmount * 1 ether;

193:         totalStakedByUser += user.stakedAmount * 1 ether;

228:     uint256 amount = _tokenAmount * 1 ether;

245:         for(uint256 i=0; i< 9; i++){

245:         for(uint256 i=0; i< 9; i++){

251:            uint256 reward_amount = RewardPercentage[i] * amount / 100;

251:            uint256 reward_amount = RewardPercentage[i] * amount / 100;

278:     uint256 amount = _tokenAmount * 1 ether;

279:     uint256 fee = _fees * 1 ether;

294:         maxTierReferralCounts[_referrer]++;

294:         maxTierReferralCounts[_referrer]++;

325:     for(uint256 i=0; i< 9; i++){

325:     for(uint256 i=0; i< 9; i++){

330:        uint256 reward_amount = RewardPercentage[i] * amount / 100;

330:        uint256 reward_amount = RewardPercentage[i] * amount / 100;

346:         address[] memory parent = new address[](9); // Initialize an array with a fixed size of 9

346:         address[] memory parent = new address[](9); // Initialize an array with a fixed size of 9

349:         for (uint256 i = 0; i < 9; i++) {

349:         for (uint256 i = 0; i < 9; i++) {

382:     totalRewards += userRewards[directStake.parent].totalrewards;

386:     totalRewards += userRewards[referral.parent].totalrewards;

```

# <a name="GAS-6"></a>[GAS-6] Use Custom Errors
[Source](https://blog.soliditylang.org/2021/04/21/custom-errors/)
Instead of using error strings, to reduce deployment and runtime cost, you should use Custom Errors. This would save both deployment and runtime cost.

*Instances (44)*:
```solidity
File: mlm/MJC_Token.sol

38:         require(_status != _ENTERED, "ReentrancyGuard: reentrant call");

129:         require(!_isBlackListedBot[sender], "Account is blacklisted");

130:         require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");

143:         require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");

150:         require(!_isBlackListedBot[sender], "Account is blacklisted");

151:         require(sender != address(0), "ERC20: transfer from the zero address");

152:         require(recipient != address(0), "ERC20: transfer to the zero address");

157:         require(senderBalance >= amount, "ERC20: transfer amount exceeds balance");

166:         require(account != address(0), "ERC20: mint to the zero address");

167:         require(!_isBlackListedBot[account], "Account is blacklisted");

176:         require(account != address(0), "ERC20: burn from the zero address");

177:         require(!_isBlackListedBot[account], "Account is blacklisted");

181:         require(accountBalance >= amount, "ERC20: burn amount exceeds balance");

190:         require(owner != address(0), "ERC20: approve from the zero address");

191:         require(spender != address(0), "ERC20: approve to the zero address");

192:         require(!_isBlackListedBot[spender], "Account is blacklisted");

211:         require(currentAllowance >= amount, "ERC20: burn amount exceeds allowance");

225:             require(c >= a, "SafeMath: addition overflow");

247:             require(c / a == b, "SafeMath: multiplication overflow");

288:         require(address(this).balance >= amount, "Address: insufficient balance");

292:         require(success, "Address: unable to send value, recipient may have reverted");

308:         require(address(this).balance >= value, "Address: insufficient balance for call");

309:         require(isContract(target), "Address: call to non-contract");

321:         require(isContract(target), "Address: static call to non-contract");

333:         require(isContract(target), "Address: delegate call to non-contract");

454:         require(_checkAndCallTransfer(_msgSender(), recipient, amount, data), "ERC1363: _checkAndCallTransfer reverts");

474:         require(_checkAndCallTransfer(sender, recipient, amount, data), "ERC1363: _checkAndCallTransfer reverts");

488:         require(_checkAndCallApprove(spender, amount, data), "ERC1363: _checkAndCallApprove reverts");

530:         require(owner() == _msgSender(), "Ownable: caller is not the owner");

540:         require(newOwner != address(0), "Ownable: new owner is the zero address");

587:                 require(!_isBlackListedBot[account], "Account is already blacklisted");

594:                 require(_isBlackListedBot[account], "Account is not blacklisted");

```

```solidity
File: mlm/StakeandSale.sol

105:         require(msg.sender == owner, "Only the owner can perform this action");

138:         require(stakingDuration_ == 90 || stakingDuration_ == 180 || stakingDuration_ == 365, "Invalid staking duration");

174:         require(stakedAmount > 0, "No staked tokens");

175:         require(block.timestamp >= user.stakingEndTime, "Staking period not ended yet");

227:     require(subscription.tokenAmount == 0, "User already has a subscription");

293:         require(maxTierReferralCounts[_referrer] <= maxRefferalLimit, "Already referred to maximum users.");

312:     require(token.balanceOf(address(this)) >= amount, "Not enough tokens in the contract");

315:     require(token.allowance(msg.sender, address(this)) >= amount, "Not enough allowance");

318:     require(token.transferFrom(msg.sender, address(this), amount), "Token transfer failed");

321:     require(token.transfer(fees_address, fee), "Fee transfer failed");

333:        require(token.balanceOf(address(this)) >= reward_amount, "Not enough tokens in the contract");

336:        require(token.transfer(parent_addr, reward_amount), "Reward transfer failed");

```

# <a name="GAS-7"></a>[GAS-7] Don't initialize variables with default value

*Instances (6)*:
```solidity
File: mlm/StakeandSale.sol

17:     uint256 public constant ZeroUSD = 0;

189:     uint256 totalStakedByUser = 0;

245:         for(uint256 i=0; i< 9; i++){

325:     for(uint256 i=0; i< 9; i++){

349:         for (uint256 i = 0; i < 9; i++) {

378:     uint256 totalRewards = 0;

```

# <a name="GAS-8"></a>[GAS-8] Long revert strings

*Instances (23)*:
```solidity
File: mlm/MJC_Token.sol

130:         require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");

143:         require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");

151:         require(sender != address(0), "ERC20: transfer from the zero address");

152:         require(recipient != address(0), "ERC20: transfer to the zero address");

157:         require(senderBalance >= amount, "ERC20: transfer amount exceeds balance");

176:         require(account != address(0), "ERC20: burn from the zero address");

181:         require(accountBalance >= amount, "ERC20: burn amount exceeds balance");

190:         require(owner != address(0), "ERC20: approve from the zero address");

191:         require(spender != address(0), "ERC20: approve to the zero address");

211:         require(currentAllowance >= amount, "ERC20: burn amount exceeds allowance");

247:             require(c / a == b, "SafeMath: multiplication overflow");

292:         require(success, "Address: unable to send value, recipient may have reverted");

308:         require(address(this).balance >= value, "Address: insufficient balance for call");

321:         require(isContract(target), "Address: static call to non-contract");

333:         require(isContract(target), "Address: delegate call to non-contract");

454:         require(_checkAndCallTransfer(_msgSender(), recipient, amount, data), "ERC1363: _checkAndCallTransfer reverts");

474:         require(_checkAndCallTransfer(sender, recipient, amount, data), "ERC1363: _checkAndCallTransfer reverts");

488:         require(_checkAndCallApprove(spender, amount, data), "ERC1363: _checkAndCallApprove reverts");

540:         require(newOwner != address(0), "Ownable: new owner is the zero address");

```

```solidity
File: mlm/StakeandSale.sol

105:         require(msg.sender == owner, "Only the owner can perform this action");

293:         require(maxTierReferralCounts[_referrer] <= maxRefferalLimit, "Already referred to maximum users.");

312:     require(token.balanceOf(address(this)) >= amount, "Not enough tokens in the contract");

333:        require(token.balanceOf(address(this)) >= reward_amount, "Not enough tokens in the contract");

```

# <a name="GAS-9"></a>[GAS-9] Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided.

*Instances (5)*:
```solidity
File: mlm/MJC_Token.sol

534:     function renounceOwnership() public virtual onlyOwner {

539:     function transferOwnership(address newOwner) public virtual onlyOwner {

550:     function recoverERC20(address tokenAddress, uint256 tokenAmount) public virtual onlyOwner {

586:             function addUserToBlacklist(address account) external onlyOwner {

593:             function removeUserFromBlacklist(address account) external onlyOwner {

```

# <a name="GAS-10"></a>[GAS-10] `++i` costs less gas than `i++`, especially when it's used in `for`-loops (`--i`/`i--` too)
*Saves 5 gas per loop*

*Instances (6)*:
```solidity
File: mlm/StakeandSale.sol

151:     userCount[msg.sender]++;

191:     for (uint256 id = 101; id <= 100 + userCount[userAddress]; id++) {

245:         for(uint256 i=0; i< 9; i++){

294:         maxTierReferralCounts[_referrer]++;

325:     for(uint256 i=0; i< 9; i++){

349:         for (uint256 i = 0; i < 9; i++) {

```

# <a name="GAS-11"></a>[GAS-11] Using `private` rather than `public` for constants, saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606 gas** in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

*Instances (7)*:
```solidity
File: mlm/StakeandSale.sol

16:     uint256 public constant maxRefferalLimit = 10;

17:     uint256 public constant ZeroUSD = 0;

18:     uint256 public constant FiftyUSD = 50;

19:     uint256 public constant HundreadUSD = 100;

20:     uint256 public constant TwoHundreadUSD = 200;

21:     uint256 public constant FiveHundreadUSD = 500;

22:     uint256 public constant ThousandUSD = 1000;

```

# <a name="GAS-12"></a>[GAS-12] Use != 0 instead of > 0 for unsigned integer comparison

*Instances (4)*:
```solidity
File: mlm/MJC_Token.sol

257:             require(b > 0, errorMessage);

284:         return size > 0;

345:             if (returndata.length > 0) {

```

```solidity
File: mlm/StakeandSale.sol

174:         require(stakedAmount > 0, "No staked tokens");

```

# <a name="GAS-13"></a>[GAS-13] `internal` functions not called by the contract should be removed
If the functions are required by an interface, the contract should inherit from that interface and use the `override` keyword

*Instances (3)*:
```solidity
File: mlm/MJC_Token.sol

223:         function add(uint256 a, uint256 b) internal pure returns (uint256) {

241:         function mul(uint256 a, uint256 b) internal pure returns (uint256) {

287:     function sendValue(address payable recipient, uint256 amount) internal {

```




