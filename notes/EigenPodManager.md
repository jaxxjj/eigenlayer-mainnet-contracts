## EigenPodManager

original doc: [EigenPodManager.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/pods/EigenPodManager.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`EigenPodManager.sol`](../../src/contracts/pods/EigenPodManager.sol) | Singleton | Transparent proxy |
| [`EigenPod.sol`](../../src/contracts/pods/EigenPod.sol) | Instanced, deployed per-user | Beacon proxy |


### Depositing Into EigenLayer
1. create pod
2. stake

```solidity
function createPod() 
    external 
    onlyWhenNotPaused(PAUSED_NEW_EIGENPODS) 
    returns (address)
```

staker deploy a pod

```solidity
function stake(
    bytes calldata pubkey, 
    bytes calldata signature, 
    bytes32 depositDataRoot
) 
    external 
    payable
    onlyWhenNotPaused(PAUSED_NEW_EIGENPODS)
```

deposit 32 ETH into beacon chain deposit contract. 设置withdrawal credentials为pod地址

##### Withdrawal Processing

1. 用户申请提款
2. removeShares减少shares
3. 进入提款队列
4. 队列完成后:
   - addShares: 重新添加shares
   - withdrawSharesAsTokens: 提取为tokens

`DelegationManager` 作为entry for undelegation and withdrawals, 通知 `EigenPodManager` 处理提款请求

1. `removeShares`:

```solidity
function removeShares(
    address podOwner, 
    uint256 shares
) 
    external 
    onlyDelegationManager
```

1. 用户取消委托(undelegate)
2. 用户申请提款(queueWithdrawals)

User -> DelegationManager.undelegate/queueWithdrawals 
     -> EigenPodManager.removeShares

2. `addShares`:

```solidity
function addShares(address podOwner, uint256 shares) external onlyDelegationManager returns (uint256) {
    // 1. 基础检查
    require(podOwner != address(0), "podOwner cannot be zero address");
    require(int256(shares) >= 0, "shares cannot be negative");
    require(shares % GWEI_TO_WEI == 0, "must be whole Gwei amount");

    // 2. 更新shares
    int256 currentPodOwnerShares = podOwnerShares[podOwner];
    int256 updatedPodOwnerShares = currentPodOwnerShares + int256(shares);
    podOwnerShares[podOwner] = updatedPodOwnerShares;

    // 3. 计算可委托的shares变化
    return uint256(
        _calculateChangeInDelegatableShares({
            sharesBefore: currentPodOwnerShares,
            sharesAfter: updatedPodOwnerShares
        })
    );
}
```

use:
1. 提款队列完成时重新添加shares
2. 允许用户在不退出验证者的情况下更换operator
3. 处理share deficit(负数shares)的情况

3. `withdrawSharesAsTokens`:

```solidity
function withdrawSharesAsTokens(
    address podOwner,
    address destination,
    uint256 shares
) external onlyDelegationManager {
    // 1. 处理share deficit
    if (currentPodOwnerShares < 0) {
        uint256 currentShareDeficit = uint256(-currentPodOwnerShares);
        if (shares > currentShareDeficit) {
            // 清除deficit后继续提款
            podOwnerShares[podOwner] = 0;
            shares -= currentShareDeficit;
        } else {
            // 只清除部分deficit后返回
            podOwnerShares[podOwner] += int256(shares);
            return;
        }
    }

    // 2. 提取ETH
    ownerToPod[podOwner].withdrawRestakedBeaconChainETH(destination, shares);
}
```

提款条件：
1. ETH已经在EigenPod中
2. 金额已在withdrawableRestakedExecutionLayerGwei中确认
3. 完成checkpoint验证

##### other methods

`recordBeaconChainETHBalanceUpdate`:

```solidity
function recordBeaconChainETHBalanceUpdate(
    address podOwner,
    int256 sharesDelta
) external onlyEigenPod(podOwner) nonReentrant {
    // 1. 基础检查
    require(podOwner != address(0), "podOwner cannot be zero address");
    require(
        sharesDelta % int256(GWEI_TO_WEI) == 0,
        "sharesDelta must be whole Gwei"
    );

    // 2. 更新shares
    int256 currentPodOwnerShares = podOwnerShares[podOwner];
    int256 updatedPodOwnerShares = currentPodOwnerShares + sharesDelta;
    podOwnerShares[podOwner] = updatedPodOwnerShares;

    // 3. 通知DelegationManager
    int256 changeInDelegatableShares = _calculateChangeInDelegatableShares({
        sharesBefore: currentPodOwnerShares,
        sharesAfter: updatedPodOwnerShares
    });
}
```

以下情况调用：
1. EigenPod.verifyWithdrawalCredentials
2. EigenPod.startCheckpoint
3. EigenPod.verifyCheckpointProofs