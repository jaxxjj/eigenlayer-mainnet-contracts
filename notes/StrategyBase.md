## StrategyBase.sol

类似vault的contract，用户deposit underlyingToken，获得shares，shares可以withdraw为underlyingToken
### variables

```solidity
contract StrategyBase is Initializable, Pausable, IStrategy {
    // pause flags
    uint8 internal constant PAUSED_DEPOSITS = 0;
    uint8 internal constant PAUSED_WITHDRAWALS = 1;

    // variables prevent inflation attack
    uint256 internal constant SHARES_OFFSET = 1e3;
    uint256 internal constant BALANCE_OFFSET = 1e3;
    
    // max shares limit
    uint256 internal constant MAX_TOTAL_SHARES = 1e38 - 1;

    // core state
    IStrategyManager public immutable strategyManager;
    IERC20 public underlyingToken;
    uint256 public totalShares;
}
```

### functions

#### deposit

```solidity
function deposit(
    IERC20 token,
    uint256 amount
) external virtual onlyWhenNotPaused(PAUSED_DEPOSITS) onlyStrategyManager returns (uint256 newShares) {
    // 预检查
    _beforeDeposit(token, amount);

    // 计算shares
    uint256 priorTotalShares = totalShares;
    uint256 virtualShareAmount = priorTotalShares + SHARES_OFFSET;
    uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
    uint256 virtualPriorTokenBalance = virtualTokenBalance - amount;
    newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

    // 验证和更新
    require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");
    totalShares = (priorTotalShares + newShares);
    require(totalShares <= MAX_TOTAL_SHARES, "StrategyBase.deposit: totalShares exceeds MAX_TOTAL_SHARES");

    // 发出事件
    _emitExchangeRate(virtualTokenBalance, totalShares + SHARES_OFFSET);

    return newShares;
}
```

#### withdraw

```solidity
function withdraw(
    address recipient,
    IERC20 token,
    uint256 amountShares
) external virtual onlyWhenNotPaused(PAUSED_WITHDRAWALS) onlyStrategyManager {
    // 预检查
    _beforeWithdrawal(recipient, token, amountShares);

    // 计算提款金额
    uint256 priorTotalShares = totalShares;
    require(
        amountShares <= priorTotalShares,
        "StrategyBase.withdraw: amountShares must be <= totalShares"
    );

    uint256 virtualPriorTotalShares = priorTotalShares + SHARES_OFFSET;
    uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
    uint256 amountToSend = (virtualTokenBalance * amountShares) / virtualPriorTotalShares;

    // 更新状态
    totalShares = priorTotalShares - amountShares;

    // 发出事件
    _emitExchangeRate(
        virtualTokenBalance - amountToSend, 
        totalShares + SHARES_OFFSET
    );

    // 执行转账
    _afterWithdrawal(recipient, token, amountToSend);
}
```

#### shares calculation

```solidity
// 查看shares对应的underlying数量
function sharesToUnderlyingView(
    uint256 amountShares
) public view virtual returns (uint256) {
    uint256 virtualTotalShares = totalShares + SHARES_OFFSET;
    uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
    return (virtualTokenBalance * amountShares) / virtualTotalShares;
}

// 查看underlying对应的shares数量
function underlyingToShares(
    uint256 amount
) public view virtual returns (uint256) {
    uint256 virtualTotalShares = totalShares + SHARES_OFFSET;
    uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
    return (amount * virtualTotalShares) / virtualTokenBalance;
}
```

