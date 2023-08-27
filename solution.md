### Solution

Owner.sol

Issue: Gas optimization
Severity: Low
Recommendation
Contract looks clean generally, however, I would consider the following slight changes for gas optimization:

- replace the require checks in the mothod \_transferOwnership(address newOwner), and in the modifier onlyOwner(), by combining an if statement with a revert with the custom error.

- avoid arithemetic computations using state variables owner directly. Declare a variable in memory and assign its value to the state variable variable. Now you can make computations with a reduced gas cost.

- change the visibility of the public methods renounceOwnership() and transferOwnership(address newOwner) to external

```
contract Ownable {
    address public owner;

    event OwnershipTransferred(
        address indexed previousOwner,
        address indexed newOwner
    );

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Ownable: caller is not the owner");
        _;
    }

    function renounceOwnership() public onlyOwner {
        emit OwnershipTransferred(owner, address(0));
        owner = address(0);
    }

    function transferOwnership(address newOwner) public onlyOwner {
        _transferOwnership(newOwner);
    }

    function _transferOwnership(address newOwner) internal {
        require(
            newOwner != address(0),
            "Ownable: new owner is the zeroaddress"
        );
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
}
```

---

IERC20.sol

Issue: No problem

---

StakedWrapper.sol

Issue: Gas optimization
Severity: Low

Recommendation
I would consider the following changes for gas optimization:

- replace the state variable \_transferErrorMessage with a custom error.

- replace therequire checks in the mothods stakeFor(address forWhom, uint128 amount) and withdraw(uint128 amount), by combining an if statement with revert and custom errors.

- avoid using state variables \_balances and buyback in conditionals and arithemetic computations directly respectively. Declare a variable in memory and assign its value to the state variable. Then you can use the conditional with a reduced gas cost.

- create a private method \_handleTransfer(address who, uint128 amount) and use it in the withdraw(uint128 amount) method for the two transfer transactions for the beneficiary and msg.sender.

Issue: Insufficient check
Severity: Medium

Recommendation
The following vulnerability could potentially lead to an increased gas usage:

- the method stakeFor(address forWhom, uint128 amount) should have a check to restrict the caller from using it with 0 value sent. This would help protect the network from spam transactions.

Issue: Revenue omission
Severity: Medium

Recommendation
The following vulnerability could potentially lead to loss of buyback revenue:

- in the method withdraw(uint128 amount), for stakes made in ERC20 tokens, there is no collection of fees for buyback as made available in native currency stakes. This could be a bug or intentional

Issue: Revenue omission
Severity: Critical

Recommendation
The following vulnerabilities would lead to a DoS and loss of funds:

- the state variable beneficiary should be made payable on declaration, or should be casted to payable in the method withdraw(uint128 amount) to avoid DoS.

- msg.sender in solidity 0.8 needs to be casted to payable (payable(msg.sender)) when transferring ether in the method withdraw(uint128 amount).

```
pragma solidity ^0.8.7;

error TokenTransferErr(address caller);
error TransferErr(bytes data);
error LowValueErr();
error ValueNotAllowed();
error Insufficient(string message);

contract StakedWrapper {
    uint256 public totalSupply;
    uint128 public buyback = 2; //defined in percentage
    mapping(address => uint256) private _balances;
    IERC20 public stakedToken;
    event Staked(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    address public beneficiary =
        address(0xDbfd6dAbD2Eaf53e5dBDc5E96fFB7E5E6B201F69);

    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }

    function stakeFor(address forWhom, uint128 amount) public payable virtual {
        IERC20 st = stakedToken;
        if (st == IERC20(address(0))) {
            //eth
            if (msg.value == 0) revert LowValueErr();
            unchecked {
                totalSupply += msg.value;
                _balances[forWhom] += msg.value;
            }
        } else {
            if (msg.value > 0) revert ValueNotAllowed();
            if (amount == 0) revert Insufficient("Amount");
            if (!st.transferFrom(msg.sender, address(this), amount))
                revert TokenTransferErr(address(st));
            unchecked {
                totalSupply += amount;
                _balances[forWhom] += amount;
            }
        }
        emit Staked(forWhom, amount);
    }

    function withdraw(uint128 amount) public virtual {
        uint256 bal = _balances[msg.sender];
        if (amount > bal) revert Insufficient("Amount");
        unchecked {
            _balances[msg.sender] -= amount;
            totalSupply = totalSupply - amount;
        }
        IERC20 st = stakedToken;
        if (st == IERC20(address(0))) {
            //eth
            uint128 bback = buyback;
            uint128 val = (amount * bback) / 100;
            _handleTransfer(payable(beneficiary), val);
            _handleTransfer(payable(msg.sender), amount - val);
        } else {
            if (!stakedToken.transfer(msg.sender, amount))
                revert TokenTransferErr(address(st));
        }
        emit Withdrawn(msg.sender, amount);
    }

    function _handleTransfer(address payable who, uint128 amount) private {
        (bool success, bytes memory data) = who.call{value: amount}("");
        if (!success) revert TransferErr(data);
    }
}

```

---

RewardsETH.sol

Issue: Undeclared importation
Severity: High

Recommendation

I would consider the following change for contract compilation:

- replace the imported undeclared contract StakedTokenWrapper to StakedWrapper.

Issue: Gas optimization
Severity: low

Recommendation
I would consider the following changes for gas optimization.

- replace the value of maxStakingAmount from 2 \* 10\*_0 _ 10\*\*17 to 0.2 ether.

- create a new private method \_updateReward(address account) and put all the code in the modifier updateReward(address account) inside this private method.

- avoid using state variables rewardRate and userRewards, and rewardToken and stakeToken in arithemetic computations directly. Declare a variable in memory and assign its value to the state variable. Then you can use the conditional with a reduced gas cost.

- replace the require checks in the mothods stake(uint128 amount), getReward(), setRewardParams(uint128 reward, uint64 duration), withdrawReward(), and setMaxStakingAmount(uint256 value) by combining an if statement with a revert with the custom error.

Issue: Insufficient Events
Severity: Low

Recommendation
Consider adding events to provide offchain information for future use on the following methods:

setBuyBackAddr(address addr), setBuyback(uint128 value), setMaxStakingAmount(uint256 value), withdrawReward(), and exit().

Issue: Insufficient checks
Severity: Medium

Recommendation
The following vulnerability could potentially mess up the staking logic:

- the method stakeFor(address forWhom, uint128 amount) should have a check to restrict the caller from using it with a value greater than the maxStakingAmount as available in the method stake(uint128 amount). This would help protect the network from spam transactions. Using a modifier would be the most optimized way of dealing with this to eliminate redundancy.

- restrict users from calling the getRewards() method with zero rewards available for them.

Issue: Wrong call location
Severity: low

Recommendation
This issue would be critical if the value being sent is a native currency rather than an ERC20 standard asset.

- it is always recommended to update the smart contract info before transferring value as shown in the method withdrawReward()

```

error ExceededLimit(uint256 amount, uint256 limit);

contract RewardsETH is StakedWrapper, Ownable {
    IERC20 public rewardToken;
    uint256 public rewardRate;
    uint64 public periodFinish;
    uint64 public lastUpdateTime;
    uint128 public rewardPerTokenStored;

    struct UserRewards {
        uint128 userRewardPerTokenPaid;
        uint128 rewards;
    }

    mapping(address => UserRewards) public userRewards;
    event SetBuyBackAddr(address indexed addr);
    event SetBuyback(uint128 value);
    event SetMaxStakingAmount(uint256 value);
    event WithdrawReward(uint256 rewardSupply);
    event Exit(address indexed user);
    event RewardAdded(uint256 reward);
    event RewardPaid(address indexed user, uint256 reward);
    uint256 public maxStakingAmount = 0.2 ether; //0.2 ETH

    constructor(IERC20 _rewardToken, IERC20 _stakedToken) {
        rewardToken = _rewardToken;
        stakedToken = _stakedToken;
    }

    modifier updateReward(address account) {
        _updateReward(account);
        _;
    }

    modifier limitCheck(uint256 amount) {
        _limitCheck(amount);
        _;
    }

    function _updateReward(address account) private {
        uint128 _rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = lastTimeRewardApplicable();
        rewardPerTokenStored = _rewardPerTokenStored;
        userRewards[account].rewards = earned(account);
        userRewards[account].userRewardPerTokenPaid = _rewardPerTokenStored;
    }

    function _limitCheck(uint256 amount) private view {
        uint256 max = maxStakingAmount;
        if (amount >= max) revert ExceededLimit(amount, max);
    }

    function lastTimeRewardApplicable() public view returns (uint64) {
        uint64 blockTimestamp = uint64(block.timestamp);
        return blockTimestamp < periodFinish ? blockTimestamp : periodFinish;
    }

    function rewardPerToken() public view returns (uint128) {
        uint256 totalStakedSupply = totalSupply;
        if (totalStakedSupply == 0) {
            return rewardPerTokenStored;
        }
        unchecked {
            uint256 rewardDuration = lastTimeRewardApplicable() -
                lastUpdateTime;
            uint256 rRate = rewardRate;
            return
                uint128(
                    rewardPerTokenStored +
                        (rewardDuration * rRate * 1e18) /
                        totalStakedSupply
                );
        }
    }

    function earned(address account) public view returns (uint128) {
        UserRewards memory uRewards = userRewards[account];
        unchecked {
            return
                uint128(
                    (balanceOf(account) *
                        (rewardPerToken() - uRewards.userRewardPerTokenPaid)) /
                        1e18 +
                        uRewards.rewards
                );
        }
    }

    function stake(uint128 amount) external payable limitCheck(amount) {
        stakeFor(msg.sender, amount);
    }

    function stakeFor(address forWhom, uint128 amount)
        public
        payable
        override
        updateReward(forWhom)
        limitCheck(amount)
    {
        super.stakeFor(forWhom, amount);
    }

    function withdraw(uint128 amount) public override updateReward(msg.sender) {
        super.withdraw(amount);
    }

    function exit() external {
        getReward();
        withdraw(uint128(balanceOf(msg.sender)));
        emit Exit(msg.sender);
    }

    function getReward() public updateReward(msg.sender) {
        uint256 reward = earned(msg.sender);
        if (reward == 0) revert Insufficient("Reward");
        userRewards[msg.sender].rewards = 0;
        if (!rewardToken.transfer(msg.sender, reward))
            revert TokenTransferErr(address(rewardToken));
        emit RewardPaid(msg.sender, reward);
    }

    function setRewardParams(uint128 reward, uint64 duration)
        external
        onlyOwner
    {
        unchecked {
            if (reward == 0) revert Insufficient("Reward Param");
            rewardPerTokenStored = rewardPerToken();
            uint64 blockTimestamp = uint64(block.timestamp);
            uint256 maxRewardSupply = rewardToken.balanceOf(address(this));
            (IERC20 rToken, IERC20 sToken) = (rewardToken, stakedToken);
            if (rToken == sToken) maxRewardSupply -= totalSupply;
            uint256 leftover = 0;
            if (blockTimestamp >= periodFinish) {
                rewardRate = reward / duration;
            } else {
                uint256 remaining = periodFinish - blockTimestamp;
                leftover = remaining * rewardRate;
                rewardRate = (reward + leftover) / duration;
            }
            if (reward + leftover > maxRewardSupply)
                revert Insufficient("Tokens");
            lastUpdateTime = blockTimestamp;
            periodFinish = blockTimestamp + duration;
            emit RewardAdded(reward);
        }
    }

    function withdrawReward() external onlyOwner {
        uint256 rewardSupply = rewardToken.balanceOf(address(this));
        //ensure funds staked by users can't be transferred out
        (IERC20 rToken, IERC20 sToken) = (rewardToken, stakedToken);
        if (rToken == sToken) rewardSupply -= totalSupply;
        rewardRate = 0;
        periodFinish = uint64(block.timestamp);
        if (!rewardToken.transfer(msg.sender, rewardSupply))
            revert TokenTransferErr(address(rewardToken));
        emit WithdrawReward(rewardSupply);
    }

    function setMaxStakingAmount(uint256 value) external onlyOwner {
        require(value > 0);
        maxStakingAmount = value;
        emit SetMaxStakingAmount(value);
    }

    function setBuyback(uint128 value) external onlyOwner {
        buyback = value;
        emit SetBuyback(value);
    }

    function setBuyBackAddr(address addr) external onlyOwner {
        beneficiary = addr;
        emit SetBuyBackAddr(addr);
    }
}

```
