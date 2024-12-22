## DelegationManager

original doc: [DelegationManager.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/core/DelegationManager.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`DelegationManager.sol`](../../src/contracts/core/DelegationManager.sol) | Singleton | Transparent proxy |


### notes+总结

`DelegationManager` 是 EigenLayer 的核心合约之一，负责处理 Staker 的委托和撤回操作，以及与 `StrategyManager` 和 `EigenPodManager` 的交互:
1. 允许staker delegate to an operator
2. undelegate
3. 处理 withdrawals 和 withdrawal processing for shares in both the `StrategyManager` and `EigenPodManager`

#### DelegationManager StrategyManager and EigenPodManager

1. DelegationManager:
   - 负责管理委托关系,连接Staker和Operator
   - 追踪每个Operator被委托的份额
   - 处理委托和取消委托的逻辑
   - 协调StrategyManager和EigenPodManager的份额变动

2. StrategyManager:
   - 管理LST的存入和提取
   - 为每个Staker记录其在不同策略中的份额
   - 处理LST相关的份额计算和转移

3. EigenPodManager:
   - 管理原生ETH的质押
   - 为每个Staker创建和管理EigenPod
   - 追踪每个Pod所有者的ETH份额
   - 处理ETH的deposit和withdrawal
  
##### workflow:

当Staker委托时:
1. DelegationManager记录Staker委托给某个Operator
2. 从StrategyManager和EigenPodManager获取Staker的份额信息
3. 将这些份额记录为Operator的委托份额

当份额发生变化时:
1. StrategyManager/EigenPodManager通知DelegationManager
2. DelegationManager相应更新Operator的委托份额记录

##### 例子：
1. 初始委托：
Alice -> DelegationManager.delegateTo(Bob)
- 检查Alice的LST份额（StrategyManager）
- 检查Alice的ETH份额（EigenPodManager）
- 记录委托关系
- 更新Bob的委托份额总量

2. 后续份额变动：
当Alice增加质押时：
- StrategyManager/EigenPodManager处理质押
- 通知DelegationManager
- DelegationManager增加Bob的委托份额

当Alice提取时：
- 通过DelegationManager发起提取请求
- 相应减少Bob的委托份额
- StrategyManager/EigenPodManager执行实际提取

##### variables

1. mapping(address => address) public delegatedTo: Staker => Operator.
   - If a Staker is not delegated to anyone, delegatedTo is unset.
   - Operators are delegated to themselves - delegatedTo[operator] == operator
2. mapping(address => mapping(IStrategy => uint256)) public operatorShares: Tracks the current balance of shares an Operator is delegated according to each strategy. Updated by both the StrategyManager and EigenPodManager when a Staker's delegatable balance changes.
   - Because Operators are delegated to themselves, an Operator's own restaked assets are reflected in these balances.
   - A similar mapping exists in the StrategyManager, but the DelegationManager additionally tracks beacon chain ETH delegated via the EigenPodManager. The "beacon chain ETH" strategy gets its own special address for this mapping: 0xbeaC0eeEeeeeEEeEeEEEEeeEEeEeeeEeeEEBEaC0.
   - workflow:
    - EigenPodManager 调用 recordBeaconChainETHBalanceUpdate

    ```solidity
   function recordBeaconChainETHBalanceUpdate(address podOwner, int256 sharesDelta) {
    // 计算份额变化
    int256 changeInDelegatableShares = _calculateChangeInDelegatableShares({
        sharesBefore: currentPodOwnerShares,
        sharesAfter: updatedPodOwnerShares
    });

    if (changeInDelegatableShares < 0) {
        // 减少份额
        delegationManager.decreaseDelegatedShares({
            staker: podOwner,
            strategy: beaconChainETHStrategy,
            shares: uint256(-changeInDelegatableShares)
        });
    } else {
        // 增加份额
        delegationManager.increaseDelegatedShares({
            staker: podOwner,
            strategy: beaconChainETHStrategy,
            shares: uint256(changeInDelegatableShares)
        });
    }
    ```
    - StrategyManager 直接份额记录:
    ```solidity
    function _addShares(
        address staker,
        IStrategy strategy,
        uint256 shares
    ) internal {
        // 直接增加份额
        stakerStrategyShares[staker][strategy] += shares;
        
        // 通知DelegationManager
        delegation.increaseDelegatedShares(staker, strategy, shares);
    }

    function _removeShares(
        address staker,
        IStrategy strategy,
        uint256 shareAmount
    ) internal {
        // 直接减少份额
        stakerStrategyShares[staker][strategy] -= shareAmount;
        
        // 通知DelegationManager
        delegation.decreaseDelegatedShares(staker, strategy, shareAmount);
    }
    ```
    - DelegationManager:
    ```solidity
    function increaseDelegatedShares(address staker, IStrategy strategy, uint256 shares) {
        if (isDelegated(staker)) {
            address operator = delegatedTo[staker];
            _increaseOperatorShares({
            operator: operator,
            staker: staker,
            strategy: strategy,
            shares: shares
        });
    }
    }

    function _increaseOperatorShares(address operator, address staker, IStrategy strategy, uint256 shares) {
        operatorShares[operator][strategy] += shares;
        emit OperatorSharesIncreased(operator, staker, strategy, shares);
    }
    ```
3. uint256 public minWithdrawalDelayBlocks 最少withdrawal delay:
   - As of M2, this is 50400 (roughly 1 week)
   - For all strategies including native beacon chain ETH, Stakers at minimum must wait this amount of time before a withdrawal can be completed. To withdraw a specific strategy, it may require additional time depending on the strategy's withdrawal delay. See strategyWithdrawalDelayBlocks below.
4. mapping(IStrategy => uint256) public strategyWithdrawalDelayBlocks:
   - This mapping tracks the withdrawal delay for each strategy. This mapping value only comes into affect if strategyWithdrawalDelayBlocks[strategy] > minWithdrawalDelayBlocks. Otherwise, minWithdrawalDelayBlocks is used.
5. mapping(bytes32 => bool) public pendingWithdrawals:
   - bytes32 通过calculateWithdrawalRoot函数计算出的withdrawal hash,包含了withdrawal结构体的所有信息(keccak256)
   - 当发起提款时，生成withdrawal hash并设置为true/完成提款时，将hash设置为false
   - nonce确保相同参数的提款可以区分
   - A per-staker nonce provides a way to distinguish multiple otherwise-identical withdrawals.

##### 常用判断

1. isDelegated(address staker) -> (bool) -> 判断staker是否被委托: `delegateTo`时一个staker只能委托给一个operator/`undelegate`时将delegatedTo[staker]设为address(0)
   - True if delegatedTo[staker] != address(0)

2. isOperator(address operator) -> (bool) -> 判断address是否是operator: `registerAsOperator`时operator自己委托给自己
   - True if delegatedTo[operator] == operator

##### 成为operator

1. call `registerAsOperator`
   1. new operator 需要提供一个struct:`OperatorDetails`, 包含：
      - address __deprecated_earningsReceiver(unused) 暂时未启用
      - address delegationApprover: delegationApprover是operator可以设置的一个审批者地址, 用于控制谁可以委托给这个operator。 如果设置为address(0)，则任何人都可以委托, 如果设置了地址，则需要该地址的signed批准。在调用 function `_delegate`时，传入approverSignatureAndExpiry, 会检查delegationApprover的签名是否有效。
      - uint32 stakerOptOutWindowBlocks 暂时未启用 
2. 后续可以调用 function `_setOperatorDetails`(address operator, OperatorDetails calldata newOperatorDetails) internal 来更新operator的details
3. 特点：
   - 不可逆性：
      - 一旦注册成为operator无法取消
      - 只能通过提款退出系统
   - 暂停机制：
      - PAUSED_NEW_DELEGATION用于紧急情况
      - 可以暂停新的委托注册
   - 自动委托：
      - operator自动委托给自己（一个地址注册operator, 实际上作为两个角色：1. 如果operator本来有EigenPodManager shares或者StrategyManager shares,那么他也是一个staker持有自己share 2. 作为operator管理他人的shares）
      - If the Operator has shares in the EigenPodManager, the DelegationManager adds these shares to the Operator's shares for the beacon chain ETH strategy.
      - For each of the strategies in the StrategyManager, if the Operator holds shares in that strategy they are added to the Operator's shares under the corresponding strategy.

事件：
    - emit OperatorRegistered(msg.sender, registeringOperatorDetails);
    - emit OperatorMetadataURIUpdated(msg.sender, metadataURI);

##### modify operator details

```solidity
function modifyOperatorDetails(OperatorDetails calldata newOperatorDetails) external
```

call `_setOperatorDetails(msg.sender, newOperatorDetails);`
emit OperatorDetailsModified(msg.sender, newOperatorDetails);

##### updateOperatorMetadataURI

```solidity
function updateOperatorMetadataURI(string calldata metadataURI) external
```

通过emit event 更新不修改链上状态
emit OperatorMetadataURIUpdated(msg.sender, metadataURI);

##### Delegating to an Operator

1. 委托方式：
- delegateTo：直接委托
  ```solidity
  function delegateTo(
    address operator,
    SignatureWithExpiry memory approverSignatureAndExpiry,
    bytes32 approverSalt
    ) external {
        // 1. 检查staker未被委托
        require(!isDelegated(msg.sender), "already delegated");
        
        // 2. 检查operator有效性
        require(isOperator(operator), "not registered operator");
        
        // 3. 执行委托
        _delegate(msg.sender, operator, approverSignatureAndExpiry, approverSalt);
    }
  ```
- delegateToBySignature：通过签名委托
  ```solidity
  function delegateToBySignature(
    address staker,
    address operator,
    SignatureWithExpiry memory stakerSignatureAndExpiry,
    SignatureWithExpiry memory approverSignatureAndExpiry,
    bytes32 approverSalt
    ) external {
        // 1. 检查签名未过期
        require(
            stakerSignatureAndExpiry.expiry >= block.timestamp,
            "staker signature expired"
        );
        
        // 2. 基本检查
        require(!isDelegated(staker), "already delegated");
        require(isOperator(operator), "not registered operator");

        // 3. 验证staker签名
        uint256 currentStakerNonce = stakerNonce[staker];
        bytes32 stakerDigestHash = calculateStakerDelegationDigestHash(
            staker, 
            currentStakerNonce, 
            operator, 
            stakerSignatureAndExpiry.expiry
        );
        
        // 4. 增加nonce防重放
        stakerNonce[staker] = currentStakerNonce + 1;
        
        // 5. 检查签名
        EIP1271SignatureUtils.checkSignature_EIP1271(
            staker,
            stakerDigestHash, 
            stakerSignatureAndExpiry.signature
        );
        
        // 6. 执行委托
        _delegate(staker, operator, approverSignatureAndExpiry, approverSalt);
    }
  ```
- 都是全部委托，不能部分委托
- 内部委托逻辑：`_delegate`:

  ```solidity
  function _delegate(
    address staker,
    address operator,
    SignatureWithExpiry memory approverSignatureAndExpiry,
    bytes32 approverSalt
    ) internal {
        // 1. 获取delegationApprover
        address _delegationApprover = _operatorDetails[operator].delegationApprover;
        
        // 2. 验证approver签名(如果需要)
        if (_delegationApprover != address(0) && 
            msg.sender != _delegationApprover && 
            msg.sender != operator) 
        {
            // 检查签名有效性...
        }
        
        // 3. 记录委托关系
        delegatedTo[staker] = operator;
        emit StakerDelegated(staker, operator);
        
        // 4. 更新shares
        (IStrategy[] memory strategies, uint256[] memory shares) = 
            getDelegatableShares(staker);
            
        for (uint256 i = 0; i < strategies.length;) {
            _increaseOperatorShares(
                operator,
                staker,
                strategies[i],
                shares[i]
            );
            unchecked { ++i; }
        }
    }
    ```

1. 签名验证：
- staker签名：证明staker同意委托
- approver签名：如果operator设置了delegationApprover
- salt机制：防止重放攻击

##### Undelegating and Withdrawing

DelegationManager.`undelegate`
DelegationManager.`queueWithdrawals`
DelegationManager.`completeQueuedWithdrawal`
DelegationManager.`completeQueuedWithdrawals`

1. 取消委托(undelegate):
- 可以由staker自己调用
- 也可以由operator或delegationApprover调用
- 会自动为所有shares排队提款
- operator的shares会相应减少

```solidity
function undelegate(
    address staker
) 
    external 
    onlyWhenNotPaused(PAUSED_ENTER_WITHDRAWAL_QUEUE)
    returns (bytes32[] memory withdrawalRoots)
```

```solidity
function undelegate(address staker) external returns (bytes32[] memory) {
    // ... 其他检查 ...
    
    // 1. 获取所有可委托的shares
    (IStrategy[] memory strategies, uint256[] memory shares) = 
        getDelegatableShares(staker);
        
    // 2. 为每个strategy创建单独的提款请求
    bytes32[] memory withdrawalRoots = new bytes32[](strategies.length);
    for (uint256 i = 0; i < strategies.length; i++) {
        // 每个strategy单独处理
        withdrawalRoots[i] = _removeSharesAndQueueWithdrawal(
            staker,
            operator,
            staker,
            [strategies[i]],  // 单个strategy
            [shares[i]]       // 对应的shares数量
        );
    }
}
```

1. 提款队列(queueWithdrawals):
- 移除shares并加入提款队列
- 生成withdrawalRoot
- 记录pending状态
- 更新nonce

```solidity
function _removeSharesAndQueueWithdrawal(
    address staker,
    address operator,
    address withdrawer,
    IStrategy[] memory strategies,
    uint256[] memory shares
) internal returns (bytes32) {
    // 1. 从operator的delegatedShares中移除
    if (operator != address(0)) {
        _decreaseOperatorShares(
            operator,
            staker,
            strategies[0],  // 单个strategy
            shares[0]       // 对应shares
        );
    }
    
    // 2. 根据strategy类型移除shares
    if (strategies[0] == beaconChainETHStrategy) {
        // 从EigenPodManager移除ETH shares
        eigenPodManager.removeShares(staker, shares[0]);
    } else {
        // 从StrategyManager移除LST shares
        strategyManager.removeShares(
            staker, 
            strategies[0], 
            shares[0]
        );
    }
    
    // 3. 创建提款记录
    Withdrawal memory withdrawal = Withdrawal({
        staker: staker,
        delegatedTo: operator,
        withdrawer: withdrawer,
        nonce: cumulativeWithdrawalsQueued[staker]++,
        startBlock: uint32(block.number),
        strategies: strategies,
        shares: shares
    });
    
    // 4. 记录提款状态
    bytes32 withdrawalRoot = calculateWithdrawalRoot(withdrawal);
    pendingWithdrawals[withdrawalRoot] = true;
    
    emit WithdrawalQueued(withdrawalRoot, withdrawal);
    return withdrawalRoot;
}
```

假设一个staker有以下shares：
10 ETH在EigenPodManager
100 stETH在StrategyManager
50 rETH在StrategyManager
当取消委托时 会创建三个独立提款请求

1. 完成提款(completeQueuedWithdrawal):
- 检查延迟时间
- 验证withdrawalRoot
- 可以选择提取token或重新获得shares
- 如果重新获得shares且已委托,会增加operator的shares

##### queueWithdrawals

```solidity
function queueWithdrawals(
    QueuedWithdrawalParams[] calldata queuedWithdrawalParams
) 
    external 
    onlyWhenNotPaused(PAUSED_ENTER_WITHDRAWAL_QUEUE) 
    returns (bytes32[] memory)
```

1. 与undelegate的区别：
- undelegate会取消委托关系
- queueWithdrawals保持委托关系
- queueWithdrawals可以选择性提取部分shares

2. 提款流程：
- 立即从operator的delegatedShares中移除
- 从EigenPodManager或StrategyManager中移除
- 进入提款队列等待期
- 可以选择提取为token（直接获得底层token）或重新获得shares （保持系统中 可以重新分配 会单独处理beaconChainETH）

##### completeQueuedWithdrawals

多个withdrawal 请求 循环处理

##### accounting (increase/decrease delegated shares)

increase
```solidity
function increaseDelegatedShares(
    address staker,
    IStrategy strategy,
    uint256 shares
) external onlyStrategyManagerOrEigenPodManager {
    // 1. 检查是否已委托
    if (isDelegated(staker)) {
        address operator = delegatedTo[staker];
        
        // 2. 增加operator的shares
        _increaseOperatorShares({
            operator: operator,
            staker: staker,
            strategy: strategy,
            shares: shares
        });
    }
}

function _increaseOperatorShares(
    address operator,
    address staker,
    IStrategy strategy,
    uint256 shares
) internal {
    // 更新shares数量
    operatorShares[operator][strategy] += shares;
    // 发出事件
    emit OperatorSharesIncreased(operator, staker, strategy, shares);
}
```

decrease
```solidity
function decreaseDelegatedShares(
    address staker,
    IStrategy strategy,
    uint256 shares
) external onlyStrategyManagerOrEigenPodManager {
    // 1. 检查是否已委托
    if (isDelegated(staker)) {
        address operator = delegatedTo[staker];
        
        // 2. 减少operator的shares
        _decreaseOperatorShares({
            operator: operator,
            staker: staker,
            strategy: strategy,
            shares: shares
        });
    }
}

function _decreaseOperatorShares(
    address operator,
    address staker,
    IStrategy strategy,
    uint256 shares
) internal {
    // 更新shares数量(会自动检查underflow)
    operatorShares[operator][strategy] -= shares;
    // 发出事件
    emit OperatorSharesDecreased(operator, staker, strategy, shares);
}
```

validator 收益
```solidity
// 验证者收益增加shares
eigenPod.verifyBalanceUpdates() {
    // ... 其他逻辑 ...
    delegationManager.increaseDelegatedShares(podOwner, beaconChainETHStrategy, increase);
}
```

validator 惩罚
```solidity
// 验证者惩罚shares
eigenPod.verifyBalanceUpdates() {
    // ... 其他逻辑 ...
    delegationManager.decreaseDelegatedShares(podOwner, beaconChainETHStrategy, decrease);
}
```

##### config

1. 全局最小延迟设置
```solidity
function setMinWithdrawalDelayBlocks(uint256 newMinWithdrawalDelayBlocks) 
    external 
    onlyOwner 
{
    _setMinWithdrawalDelayBlocks(newMinWithdrawalDelayBlocks);
}

function _setMinWithdrawalDelayBlocks(uint256 _minWithdrawalDelayBlocks) internal {
    // 1. 验证不超过最大限制
    require(
        _minWithdrawalDelayBlocks <= MAX_WITHDRAWAL_DELAY_BLOCKS,
        "DelegationManager._setMinWithdrawalDelayBlocks: _minWithdrawalDelayBlocks cannot be > MAX_WITHDRAWAL_DELAY_BLOCKS"
    );
    
    // 2. 发出事件
    emit MinWithdrawalDelayBlocksSet(minWithdrawalDelayBlocks, _minWithdrawalDelayBlocks);
    
    // 3. 更新状态
    minWithdrawalDelayBlocks = _minWithdrawalDelayBlocks;
}
```

2. strategy delay:

```solidity
function setStrategyWithdrawalDelayBlocks(
    IStrategy[] calldata strategies,
    uint256[] calldata withdrawalDelayBlocks
) external onlyOwner {
    _setStrategyWithdrawalDelayBlocks(strategies, withdrawalDelayBlocks);
}

function _setStrategyWithdrawalDelayBlocks(
    IStrategy[] calldata _strategies,
    uint256[] calldata _withdrawalDelayBlocks
) internal {
    // 1. 验证输入长度匹配
    require(
        _strategies.length == _withdrawalDelayBlocks.length,
        "input length mismatch"
    );
    
    // 2. 遍历设置每个策略的延迟
    for (uint256 i = 0; i < _strategies.length; ++i) {
        IStrategy strategy = _strategies[i];
        uint256 prevDelay = strategyWithdrawalDelayBlocks[strategy];
        uint256 newDelay = _withdrawalDelayBlocks[i];
        
        // 验证不超过最大限制
        require(
            newDelay <= MAX_WITHDRAWAL_DELAY_BLOCKS,
            "delay cannot be > MAX_WITHDRAWAL_DELAY_BLOCKS"
        );
        
        // 更新延迟
        strategyWithdrawalDelayBlocks[strategy] = newDelay;
        
        // 发出事件
        emit StrategyWithdrawalDelayBlocksSet(
            strategy, 
            prevDelay, 
            newDelay
        );
    }
}
```

3. 获取实际延迟：

```solidity
function getWithdrawalDelay(IStrategy[] calldata strategies) 
    public view 
    returns (uint256) 
{
    // 1. 从全局最小延迟开始
    uint256 withdrawalDelay = minWithdrawalDelayBlocks;
    
    // 2. 遍历所有策略找最大延迟
    for (uint256 i = 0; i < strategies.length; ++i) {
        uint256 currDelay = strategyWithdrawalDelayBlocks[strategies[i]];
        if (currDelay > withdrawalDelay) {
            withdrawalDelay = currDelay;
        }
    }
    
    return withdrawalDelay;
}
```