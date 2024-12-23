## StrategyManager

original doc: [StrategyManager.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/core/StrategyManager.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`StrategyManager.sol`](../../src/contracts/core/StrategyManager.sol) | Singleton | Transparent proxy |
| [`StrategyFactory.sol`](../../src/contracts/core/StrategyFactory.sol) | Singleton | Transparent proxy |
| [`StrategyBaseTVLLimits.sol`](../../src/contracts/strategies/StrategyBaseTVLLimits.sol) | Instanced, one per supported token | - Strategies deployed outside the `StrategyFactory` use transparent proxies <br /> - Anything deployed via the `StrategyFactory` uses a Beacon proxy |

### general

Strategy：EigenLayer中用于管理Liquid Staking Token(LST)的标准化接口和实现。每个Strategy负责管理一种特定的token

1. 接收user存入的LST
2. 计算并分配对应的shares(份额)
3. 处理withdraw
4. control access

1. 允许staker 在对应的strategy 存入 LST
2. 允许DelegationManager remove shares when staker queue withdrawals
3. withdraw：
   1. adding shares back to staker
   2. directly withdraw from strategy

`StrategyBaseTVLLimits`: 
1. StrategyFactory为每个token部署一个Strategy deposit and withdraw
2. 自动加入白名单
3. 确保每个token只有一个Strategy

对于EIGEN/bEIGEN 使用contract `EigenStrategy`

### variables

- `mapping(address => mapping(IStrategy => uint256)) public stakerStrategyShares`: staker 在 strategy中的shares

```solidity
1. 存款时增加shares
function _addShares(address staker, IStrategy strategy, uint256 shares) internal {
    stakerStrategyShares[staker][strategy] += shares;
}

1. 提款时减少shares
function _removeShares(address staker, IStrategy strategy, uint256 shares) internal {
    stakerStrategyShares[staker][strategy] -= shares;
}
```

- `mapping(address => IStrategy[]) public stakerStrategyList`: 1 to many staker 对应的 strategies
Updated as needed when Stakers deposit and withdraw: if a Staker has a zero balance in a Strategy, it is removed from the list. Likewise, if a Staker deposits into a Strategy and did not previously have a balance, it is added to the list.
```solidity
1. 首次存款时添加策略
if (stakerStrategyShares[staker][strategy] == 0) {
    stakerStrategyList[staker].push(strategy);
}

2. 提款清零时移除策略
if (userShares == 0) {
    _removeStrategyFromStakerStrategyList(staker, strategy);
}
```
- `mapping(IStrategy => bool) public strategyIsWhitelistedForDeposit`: The strategyWhitelister is (as of M2) a permissioned role that can be changed by the contract owner. The strategyWhitelister has currently whitelisted 3 StrategyBaseTVLLimits contracts in this mapping, one for each supported LST.

- `mapping(IStrategy => bool) public thirdPartyTransfersForbidden`: The strategyWhitelister can disable third party transfers for a given strategy. If thirdPartyTransfersForbidden[strategy] == true:
    - Users cannot deposit on behalf of someone else (see depositIntoStrategyWithSignature).
Users cannot withdraw on behalf of someone else. (see DelegationManager.queueWithdrawals)
控制第三方转账权限

#### tips
stakerStrategyListLength(address staker) -> (uint):
```solidity
function stakerStrategyListLength(address staker) external view returns (uint256) {
    return stakerStrategyList[staker].length;
}
```
`DelegationManager` 使用这个函数来确定一个 Staker 是否在 StrategyManager 中拥有任何策略的 shares (如果为 0，则没有)

uint256 constant MAX_TOTAL_SHARES = 1e38 - 1
The maximum total shares a single strategy can handle. 

- 防止shares溢出
- 保护链下服务计算
- 提供合理的上限值

### Depositing Into Strategies

用于deposit LST, receive shares

#### `depositIntoStrategy` 函数用于将 LST 存入策略，并返回相应的 shares。

```solidity
function depositIntoStrategy(
    IStrategy strategy, 
    IERC20 token, 
    uint256 amount
)
    external
    onlyWhenNotPaused(PAUSED_DEPOSITS)
    onlyNotFrozen(msg.sender)
    nonReentrant
    returns (uint256 shares)
```
underlying strategy 必须是 StrategyBaseTVLLimits whitelisted instances 中的一个
shares 通过exchange rate 得出

```solidity
interface IStrategy {
    // 1. 核心功能
    function deposit(IERC20 token, uint256 amount) external returns (uint256);
    function withdraw(address recipient, IERC20 token, uint256 amount) external;
    
    // 2. 查询功能
    function sharesToUnderlying(uint256 shares) external view returns (uint256);
    function underlyingToShares(uint256 amount) external view returns (uint256);
    function totalShares() external view returns (uint256);
}
```
1. StrategyBaseTVLLimits:
   - 标准ERC20 token策略
   - 支持TVL限制
   - 通过StrategyFactory部署

2. EigenStrategy:
   - EIGEN/bEIGEN专用策略
   - 特殊的shares计算逻辑

3. 预白名单LST策略:
   - 早期支持的LST token策略
   - 在StrategyFactory前已存在
   - 
内部：
```solidity
function _depositIntoStrategy(
    address staker,
    IStrategy strategy,
    IERC20 token,
    uint256 amount
) internal returns (uint256 shares) {
    // 1. 转移token到strategy
    token.safeTransferFrom(msg.sender, address(strategy), amount);

    // 2. 调用strategy计算shares
    shares = strategy.deposit(token, amount);

    // 3. 更新staker的shares记录
    _addShares(staker, token, strategy, shares);

    // 4. 更新委托关系
    delegation.increaseDelegatedShares(staker, strategy, shares);
}
```


#### `depositIntoStrategyWithSignature` 

通过签名授权第三方进行存款操作。

```solidity
function depositIntoStrategyWithSignature(
    IStrategy strategy,
    IERC20 token,
    uint256 amount,
    bytes calldata signature
) external returns (uint256 shares) {
```

```solidity
function depositIntoStrategyWithSignature(
    IStrategy strategy,
    IERC20 token,
    uint256 amount,
    address staker,
    uint256 expiry,
    bytes memory signature
) external returns (uint256 shares) {
    // 1. 检查第三方转账权限
    require(!thirdPartyTransfersForbidden[strategy]);

    // 2. 检查签名是否过期
    require(expiry >= block.timestamp);

    // 3. 验证签名
    bytes32 digestHash = _calculateDigestHash(
        staker, strategy, token, amount, nonces[staker], expiry
    );
    EIP1271SignatureUtils.checkSignature_EIP1271(staker, digestHash, signature);

    // 4. 更新nonce
    nonces[staker]++;

    // 5. 执行存款
    shares = _depositIntoStrategy(staker, strategy, token, amount);
}
```

### Withdrawal Processing

以下所有方法 only `DelegationManager`  call

#### `removeShares`

```solidity
function removeShares(
    address staker,
    IStrategy strategy, 
    uint256 shares
) external onlyDelegationManager {
    _removeShares(staker, strategy, shares);
}

function _removeShares(
    address staker,
    IStrategy strategy,
    uint256 shares
) internal returns (bool) {
    // 1. 检查输入
    require(shares != 0, "shareAmount should not be zero!");
    
    // 2. 检查余额
    uint256 userShares = stakerStrategyShares[staker][strategy];
    require(shares <= userShares, "shareAmount too high");
    
    // 3. 更新余额
    stakerStrategyShares[staker][strategy] = userShares - shares;
    
    // 4. 如果余额为0,移除策略
    if (userShares == 0) {
        _removeStrategyFromStakerStrategyList(staker, strategy);
        return true;
    }
    return false;
}
```

当staker queue withdrawals, remove 对应strategy 的share  如果为0 移除策略。 当withdraw completed 调用addShares 或者 withdrawSharesAsTokens

`addShares`:

```solidity
function addShares(
    address staker,
    IStrategy strategy,
    uint256 shares
) 
    external 
    onlyDelegationManager
```

`withdrawSharesAsTokens`:

```solidity
function withdrawSharesAsTokens(
    address recipient,
    IStrategy strategy,
    uint shares,
    IERC20 token
)
    external
    onlyDelegationManager
```

### Strategies

only `StrategyManager` can call
`StrategyBaseTVLLimits.deposit`
`StrategyBaseTVLLimits.withdraw`

#### `StrategyBaseTVLLimits.deposit`

```solidity
function deposit(
    IERC20 token, 
    uint256 amount
)
    external
    onlyWhenNotPaused(PAUSED_DEPOSITS)
    onlyStrategyManager
    returns (uint256 newShares)
```

存入underlying token 返回shares

#### `StrategyBaseTVLLimits.withdraw`

```solidity
function withdraw(
    address recipient, 
    IERC20 token, 
    uint256 amountShares
)
    external
    onlyWhenNotPaused(PAUSED_WITHDRAWALS)
    onlyStrategyManager
```

queued withdrawals completed 时，把shares 转换为underlying token 并transfer 给recipient

### `StrategyFactory.deployNewStrategy`

external function,允许任何人为指定token部署新的策略合约:

```solidity
function deployNewStrategy(IERC20 token)
    external
    onlyWhenNotPaused(PAUSED_NEW_STRATEGIES)
    returns (IStrategy newStrategy)
```

- 为指定token部署新的Strategy代理合约
- 自动将新策略添加到StrategyManager的白名单中
- 记录token与strategy的对应关系

limitations:
- 每个token只能部署一个strategy
- 被列入黑名单的token不能部署strategy
- 系统暂停时不能部署新strategy

### StrategyFactory.blacklistTokens

```solidity
function blacklistTokens(IERC20[] calldata tokens) external onlyOwner
```
- owner可以将token加入黑名单,防止为其部署新策略:

- 将指定token加入黑名单
- 如果token已有strategy,会从StrategyManager白名单中移除
- 只有合约所有者可以调用
- 黑名单是永久性的(无法移除)
- 已经被加入黑名单的token不能再次加入

### `StrategyFactory.whitelistStrategies`

```solidity
function whitelistStrategies(
    IStrategy[] calldata strategiesToWhitelist,
    bool[] calldata thirdPartyTransfersForbiddenValues
) 
    external 
    onlyOwner
```

- owner将指定策略添加到StrategyManager的白名单
- 可以同时设置第三方转账权限
- 用于添加不是通过工厂部署的策略
- 批量管理策略白名单

### `StrategyFactory.setThirdPartyTransfersForbidden`

```solidity
function setThirdPartyTransfersForbidden(IStrategy strategy, bool value) external onlyOwner
```

- owner设置指定策略是否允许第三方转账
- 直接调用StrategyManager的对应功能

### `StrategyFactory.removeStrategiesFromWhitelist`

```solidity
function removeStrategiesFromWhitelist(IStrategy[] calldata strategiesToRemoveFromWhitelist) external onlyOwner
```

- 将指定策略从StrategyManager白名单中移除
- 批量操作能力

### system configuration

#### `setStrategyWhitelister`

```solidity
function setStrategyWhitelister(address newStrategyWhitelister) external onlyOwner
```

Whitelister:
- 管理策略白名单
- 控制第三方转账权限

#### `addStrategiesToDepositWhitelist`

```solidity
function addStrategiesToDepositWhitelist(
    IStrategy[] calldata strategiesToWhitelist,
    bool[] calldata thirdPartyTransfersForbiddenValues
) external onlyStrategyWhitelister
```

- 批量添加策略到白名单
- 设置每个策略的第三方转账权限

#### `removeStrategiesFromDepositWhitelist`

```solidity
function removeStrategiesFromDepositWhitelist(
    IStrategy[] calldata strategiesToRemoveFromWhitelist
) external onlyStrategyWhitelister
```

- 批量移除strategies from whitelist
- 自动禁用第三方转账权限

#### `setThirdPartyTransfersForbidden`

```solidity
function setThirdPartyTransfersForbidden(
    IStrategy strategy,
    bool value
) external onlyStrategyWhitelister
```

- 签名存款功能(depositIntoStrategyWithSignature)
- 提款到其他地址的功能


当第三方转账被禁用时:
1. 不能使用签名进行存款
2. 不能提款到其他地址
3. 只能直接操作自己的账户