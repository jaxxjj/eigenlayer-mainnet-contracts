## EigenPod
original doc: [EigenPod.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/pods/EigenPod.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`EigenPod.sol`](../../src/contracts/pods/EigenPod.sol) | Instanced, deployed per-user | Beacon proxy |

### notes+总结

#### general

`EigenPod` 由 staker 通过 `EigenPodManager` 部署，每个用户一个。一个pod支持多个validators

1. EigenPod serves as two roles:
- withdrawal credentials: 接收validators的本金和共识层奖励
```solidity
function _podWithdrawalCredentials() internal view returns (bytes memory) {
    // 生成ETH2.0提款凭证
    return abi.encodePacked(
        bytes1(uint8(1)),  // 版本
        bytes11(0),        // 填充
        address(this)      // Pod地址
    );
}
```

- fee recipient: EigenPod作为 接收执行层奖励和MEV收益的接收者
```solidity
// Pod可以接收执行层奖励
receive() external payable {
    // 接收执行层奖励和MEV
    emit FeeRecipientRewardsReceived(msg.value);
}
```

##### roles

- Pod Owner：部署EigenPod的Staker
- Proof Submitter：可以call 一些EigenPod的函数，日常操作的热钱包地址
- Active validator：所有把withdrawal credentials设置为EigenPod地址的validators(verified过的)

A validator enters the active validator set when their withdrawal credentials are verified
A validator leaves the active validator set when a checkpoint proof shows they have 0 balance

`uint256 activeValidatorCount`: 有validator enter +1; 有validator leave -1
`mapping(bytes32 => ValidatorInfo) _validatorPubkeyHashToInfo`: 记录validator的状态
VALIDATOR_STATUS.INACTIVE -> VALIDATOR_STATUS.ACTIVE when entering the active validator set 在`_verifyWithdrawalCredentials` 更新 after 验证credentials后
VALIDATOR_STATUS.ACTIVE -> VALIDATOR_STATUS.WITHDRAWN when leaving the active validator set 在`_verifyCheckpointProof` 验证余额如果0 WITHDRAWN


- Checkpoint: A snapshot of EigenPod and beacon chain state used to update the Pod Owner's shares based on a combination of beacon chain balance and native ETH balance. 快照 -> 更新 Checkpoints allow an EigenPod to account for validator exits, partial withdrawals of consensus rewards, or execution layer fees earned by their validators. Completing a checkpoint will account for these amounts in the EigenPod, enabling the Pod Owner to compound their restaked shares or withdraw accumulated yield.

```solidity
struct Checkpoint {
    bytes32 beaconBlockRoot;  // proofs are verified against a beacon block root
    uint24 proofsRemaining;   // number of proofs remaining before the checkpoint is completed
    uint64 podBalanceGwei;    // native ETH that will be awarded shares when the checkpoint is completed
    int128 balanceDeltasGwei; // total change in beacon chain balance tracked across submitted proofs
}
```

1. 初始化Checkpoint:
```solidity
function _startCheckpoint(bool revertIfNoBalance) internal {
    // 1. 确保没有活跃checkpoint
    require(
        currentCheckpointTimestamp == 0,
        "must finish previous checkpoint"
    );
    
    // 2. 计算可用ETH余额
    uint64 podBalanceGwei = uint64(address(this).balance / GWEI_TO_WEI) 
        - withdrawableRestakedExecutionLayerGwei;
    
    // 3. 创建新checkpoint
    Checkpoint memory checkpoint = Checkpoint({
        beaconBlockRoot: getParentBlockRoot(uint64(block.timestamp)),
        proofsRemaining: uint24(activeValidatorCount),
        podBalanceGwei: podBalanceGwei,
        balanceDeltasGwei: 0
    });
    
    // 4. 更新状态
    currentCheckpointTimestamp = uint64(block.timestamp);
    _updateCheckpoint(checkpoint);
}
```

2. 处理验证者证明

```solidity
function verifyCheckpointProofs(
    BeaconChainProofs.BalanceContainerProof calldata balanceContainerProof,
    BeaconChainProofs.BalanceProof[] calldata proofs
) external {
    // 1. 获取当前checkpoint
    Checkpoint memory checkpoint = _currentCheckpoint;
    
    // 2. 验证每个证明
    for (uint256 i = 0; i < proofs.length; i++) {
        ValidatorInfo memory validatorInfo = 
            _validatorPubkeyHashToInfo[proofs[i].pubkeyHash];
            
        // 只处理ACTIVE状态的验证者
        if (validatorInfo.status != VALIDATOR_STATUS.ACTIVE) {
            continue;
        }
        
        // 处理证明并更新余额
        (int128 balanceDelta, uint64 exitedBalance) = 
            _verifyCheckpointProof(
                validatorInfo,
                checkpointTimestamp,
                balanceContainerProof.balanceContainerRoot,
                proofs[i]
            );
            
        // 更新checkpoint状态
        checkpoint.proofsRemaining--;
        checkpoint.balanceDeltasGwei += balanceDelta;
    }
    
    // 3. 更新checkpoint
    _updateCheckpoint(checkpoint);
}
```

3. 更新状态

```solidity
function _updateCheckpoint(Checkpoint memory checkpoint) internal {
    if (checkpoint.proofsRemaining == 0) {
        // 1. 计算shares变化
        int256 totalShareDeltaWei = 
            (int128(uint128(checkpoint.podBalanceGwei)) + 
             checkpoint.balanceDeltasGwei) * 
            int256(GWEI_TO_WEI);
        
        // 2. 更新可提取余额
        withdrawableRestakedExecutionLayerGwei += 
            checkpoint.podBalanceGwei;
        
        // 3. 完成checkpoint
        lastCheckpointTimestamp = currentCheckpointTimestamp;
        delete currentCheckpointTimestamp;
        delete _currentCheckpoint;
        
        // 4. 更新shares
        eigenPodManager.recordBeaconChainETHBalanceUpdate(
            podOwner,
            totalShareDeltaWei
        );
    }
}
```

##### restaking Beacon Chain ETH

call `verifyWithdrawalCredentials` 验证validator的withdrawal credentials是否设置为EigenPod地址。如果是的话把validator加入active validator set（以后每个checkpoint proof都要加入） 并且更新这个pod的share
1. 主要目的：
- 验证withdrawal credentials
- 开始restaking ETH
- 获得可委托shares
- 加入active validator set

2. 验证流程：
- 验证beacon state root
- 验证withdrawal credentials
- 验证validator状态
- 计算shares

3. 状态变更：
- activeValidatorCount增加
- validator状态更新
- shares分配
- checkpoint记录

##### Checkpointing Validators

```solidity
function startCheckpoint(bool revertIfNoBalance)
    external
    onlyOwnerOrProofSubmitter() 
    onlyWhenNotPaused(PAUSED_START_CHECKPOINT) 
```

- 开始一个新的checkpoint周期
- 记录当前状态快照
- 准备验证validator状态

- podBalanceGwei: 
```solidity
// podBalanceGwei = 当前ETH余额 - 已计入shares的余额
uint64 podBalanceGwei = uint64(address(this).balance / GWEI_TO_WEI) - 
    withdrawableRestakedExecutionLayerGwei;

// 1. address(this).balance: pod当前的ETH余额
// 2. GWEI_TO_WEI: 转换单位(1 GWEI = 10^9 WEI)
// 3. withdrawableRestakedExecutionLayerGwei: 已经计入shares的余额


// 总ETH余额 = Beacon Chain余额 + Pod Native余额

// 例如：
// 1. Beacon Chain: 32 ETH (验证过的validator余额)
// 2. Pod Native: 2 ETH (已提取的收益)
// 总计: 34 ETH = 可以发行34个shares
```

- activeValidatorCount: 在active validator set的validators数量
- beaconBlockRoot: 上一个slot 的 beacon block root 

workflow:

```solidity
// 例如：一个pod有一个validator

// 1. 初始状态
beaconBalance = 34 ETH    // beacon chain上
podBalance = 0 ETH        // pod中
shares = 34               // 发行34个shares

// 2. 提取2 ETH到pod
beaconBalance = 32 ETH    // beacon chain剩余
podBalance = 2 ETH        // pod接收提款
shares = 34               // shares数量不变

// 3. 任何时候
totalETH = beaconBalance + podBalance
// 32 ETH   +   2 ETH    = 34 ETH
// 与shares数量完全对应
```

以上就是backed share

这里记录current balance
区别：
1. current balance  实际ETH余额 每个slot更新(12秒) 精确反映validator的实际ETH数量
1. effective balance 是当前的余额，不包括已计入shares的余额 用于权重计算的余额 每个epoch更新(6.4分钟，32个slot) 向下取整到1 ETH 最大值32 ETH

`verifyCheckpointProofs`: 
```solidity
function verifyCheckpointProofs(
    BeaconChainProofs.BalanceContainerProof calldata balanceContainerProof,
    BeaconChainProofs.BalanceProof[] calldata proofs
)
    external 
    onlyWhenNotPaused(PAUSED_EIGENPODS_VERIFY_CHECKPOINT_PROOFS) 

struct BalanceContainerProof {
    bytes32 balanceContainerRoot;
    bytes proof;
}

struct BalanceProof {
    bytes32 pubkeyHash;
    bytes32 balanceRoot;
    bytes proof;
}
```

- 验证validator的current balance(每slot更新)
- 不是验证effective balance(每epoch更新)
- 使用merkle proof验证余额

workflow: 
   1.  验证ACTIVE状态
   2.  防止重复验证
   3.  处理proof
   
validator 验证条件：
```solidity
// 条件1: validator必须是ACTIVE状态
if (validatorInfo.status != VALIDATOR_STATUS.ACTIVE) {
    continue;  // 跳过这个validator
}

// 条件2: 防止重复验证
if (validatorInfo.lastCheckpointedAt >= checkpointTimestamp) {
    continue;  // 跳过这个validator
}
```

如果验证失败直接跳过，而不是整个交易revert -> 防止攻击者frontrun并提交部分proof导致整批proof失败

##### staleness proofs

```solidity
// Staleness Proofs用于：
// 1. 允许第三方在validator被惩罚时启动checkpoint
// 2. 防止Pod Owner恶意不更新被惩罚的validator状态
// 3. 确保系统状态的及时更新

function verifyStaleBalance(
    uint64 beaconTimestamp,
    BeaconChainProofs.StateRootProof calldata stateRootProof,
    BeaconChainProofs.ValidatorProof calldata proof
) external {
    // 验证validator是否被惩罚
    require(
        proof.validatorFields.isValidatorSlashed(),
        "validator must be slashed"
    );
}
```

通常情况下 pod owner 会自己去主动执行checkpoint因为这是唯一获取 1.质押收益 2.更新shares数量 3.提取收益 的办法。
但是当pod owner 被slashed 时，他可能不会主动执行checkpoint确认损失，此时第三方可以提交staleness proof来启动checkpoint: `verifyStaleBalance` (允许任何人在validator被惩罚时强制执行checkpoint)

##### other methods

`setProofSubmitter`: 
```solidity
function setProofSubmitter(address newProofSubmitter) external onlyEigenPodOwner
```

更新日常管理地址

`stake`:
```solidity
function stake(
    bytes calldata pubkey,
    bytes calldata signature,
    bytes32 depositDataRoot
)
    external 
    payable 
    onlyEigenPodManager
```

调用ETH2.0 deposit；pod地址作为withdrawal credentials

`withdrawRestakedBeaconChainETH`:
```solidity
function withdrawRestakedBeaconChainETH(
    address recipient, 
    uint256 amountWei
)
    external 
    onlyEigenPodManager
```
由EigenPodManager调用，用于提取restaked的beacon chain ETH(NATIVE ETH)

```solidity
// 1. 用户发起提款请求到DelegationManager
DelegationManager.queueWithdrawal()

// 2. DelegationManager通知EigenPodManager
EigenPodManager.withdrawSharesAsTokens()

// 3. EigenPodManager调用EigenPod
EigenPod.withdrawRestakedBeaconChainETH()
```

- 只能由EigenPodManager调用
- 提款金额必须是整数GWEI
- 只能提取已经在checkpoint中确认的ETH
- 扣除 `withdrawableRestakedExecutionLayerGwei`

`recoverTokens`:
```solidity
function recoverTokens(
    IERC20[] memory tokenList,
    uint256[] memory amountsToWithdraw,
    address recipient
) 
    external 
    onlyEigenPodOwner 
    onlyWhenNotPaused(PAUSED_NON_PROOF_WITHDRAWALS)
```

转出误入的ERC20