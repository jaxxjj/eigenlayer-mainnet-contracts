## RewardsCoordinator

original doc: [RewardsCoordinator.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/core/RewardsCoordinator.md)

| File | Type | Proxy |
| -------- | -------- | -------- |
| [`RewardsCoordinator.sol`](../../src/contracts/core/RewardsCoordinator.sol) | Singleton | Transparent proxy |

AVS 通过rewards submission 发送ERC20 给RewardsCoordinator，用于对specific time range 里面对operator奖励

当前有两种rewards type:
1. reward v1: rewards submissions
2. reward v2: operator-directed rewards submissions. Oct 18, 2024 的 [Eigenlayer proposal 001](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-001.md)

- offchain: trusted rewards updater 计算 在time range 里 rewards distribution:
  - v1: 1. 每个operator stake weight 2. a default split to each operator
  - v2: 1. AVS custom rewards logic 2. per-operator splits
- onchain:
  - trusted rewards updater 发送给rewards coordinator merkle root, earners 提供 merkle proof 来claim rewards
  
workflow:
1. AVS 提交给RewardsCoordinator rewards submission (v1 or v2), specifies a time range and token 还有 each strategy's distribution.
2. trusted rewards updater 在offchain计算 rewards distribution based on 提交的 rewards submission, 发送给rewards coordinator merkle root. 在activationDelay后DistributionRoot 可以被claim
3. staker 和 operator 提供 merkle proof against `DistributionRoot` 来claim rewards

### variables

`DistributionRoot[] public distributionRoots`:
distributionRoots stores historic reward merkle tree roots submitted by the rewards updater. For each earner, the rewards merkle tree stores cumulative earnings per ERC20 reward token. For more details on merkle tree structure see Rewards Merkle Tree Structure below.

```solidity
struct DistributionRoot {
    bytes32 root;                      // Merkle树根哈希
    uint32 activatedAt;                // 激活时间
    uint32 rewardsCalculationEndTimestamp;  // 计算结束时间
    bool disabled;                     // 是否被禁用
}
```

- 存储所有历史奖励的Merkle树根
- 每个根节点包含cumulative收益信息
- 由rewards updater定期提交

`mapping(address => address) public claimerFor: earner => claimer`:
Stakers and Operators can designate a "claimer" who can claim rewards via on their behalf via processClaim. If a claimer is not set in claimerFor, the earner will have to call processClaim themselves.
Note that the claimer isn't necessarily the reward recipient, but they do have the authority to specify the recipient when calling processClaim on the earner's behalf.

- 允许质押者和运营商指定认领代理
- 代理可以代表earner调用processClaim
- 如果没设置,则只有earner本人可以认领

`mapping(address => mapping(IERC20 => uint256)) public cumulativeClaimed: earner => token => total amount claimed to date`:
Mapping for earners(Stakers/Operators) to track their total claimed earnings per reward token. This mapping is used to calculate the difference between the cumulativeEarnings stored in the merkle tree and the previous total claimed amount. This difference is then transfered to the specified destination address.

- 记录每个earner的每种代币已认领总量
- 用于计算新的可认领数量
- 防止重复认领

operator 分成: 
`uint16 public defaultOperatorSplitBips`: Used off-chain by the rewards updater to calculate an Operator's split for a specific reward.
This is expected to be a flat 10% rate for the initial rewards release. Expressed in basis points, this is 1000.

- 初始设置为10%(1000基点)
- 用于标准奖励计算
- 全局统一比例

AVS vs PI:
PI奖励是EigenLayer协议层面的程序化激励，区别于AVS提供的应用层奖励。

`mapping(address => mapping(address => OperatorSplit)) internal operatorAVSSplitBips: operator => AVS => OperatorSplit`:
Operators specify their custom split for a given AVS for each OperatorDirectedRewardsSubmission, where Stakers receive a relative proportion (by stake weight) of the remaining amount.

```solidity
struct OperatorSplit {
    uint16 splitBips;        // 分成比例(基点)
    uint32 updateTimestamp;  // 更新时间
    bool active;            // 是否激活
}
```
- operator可为每个AVS设置自定义分成
- operator获得设定比例的奖励
- 剩余奖励按质押权重分配给stakers
- 每个AVS可以有不同的分成比例

例子：
```solidity
operator A:
  AVS1: {splitBips: 1000, updateTimestamp: xxx, active: true}  // 10%分成
  AVS2: {splitBips: 1500, updateTimestamp: xxx, active: true}  // 15%分成
```

`mapping(address => OperatorSplit) internal operatorPISplitBips: operator => OperatorSplit`:
Operators may also specify their custom split for programmatic incentives, where Stakers similarly receive a relative proportion (by stake weight) of the remaining amount.

- 针对程序化激励(Programmatic Incentives)的全局分成设置
- 统一的PI奖励分配策略
- 简化的管理机制

```solidity
operator A: {splitBips: 1200, updateTimestamp: xxx, active: true}  // 所有PI奖励12%分成
奖励: 1000 EL代币
operator分成(12%): 120 EL代币
stakers分成(88%): 880 EL代币
```

### tips

`_checkClaim(RewardsMerkleClaim calldata claim, DistributionRoot memory root)`:
用来check 用于claim 的merkle在不在root里
Reverts if any of the following are true:
- mismatch input param lengths: tokenIndices, tokenTreeProofs, tokenLeaves 三长度不一致
- earner proof reverting from calling _verifyEarnerClaimProof earner不在root里
- any of the token proofs reverting from calling _verifyTokenClaimProof 某个token不在earner 子树里
  
AVS (Actively Validated Service) is mentioned, 指的是 contract entity that is submitting rewards to the RewardsCoordinator. This is assumed to be a customized ServiceManager contract of some kind that is interfacing with the EigenLayer protocol. See the ServiceManagerBase docs here: eigenlayer-middleware/docs/ServiceManagerBase.md.
A rewards submission includes, unless specified otherwise, both the v1 RewardsSubmission and the v2 OperatorDirectedRewardsSubmission types.

### Submitting Rewards Requests

通过一下functions Submitting Rewards Requests:

- `RewardsCoordinator.createAVSRewardsSubmission`
- `RewardsCoordinator.createRewardsForAllSubmission`
- `RewardsCoordinator.createRewardsForAllEarners`
- `RewardsCoordinator.createOperatorDirectedAVSRewardsSubmission`

```solidity
// 特定AVS奖励
function createAVSRewardsSubmission(RewardsSubmission[] calldata submissions) {
    // 针对特定AVS的参与者
    // 无调用者限制
    // 存储在isAVSRewardsSubmissionHash
}
```

```solidity
// 全局奖励
function createRewardsForAllSubmission(RewardsSubmission[] calldata submissions) {
    // 针对所有stakers
    // 仅白名单submitter可调用
    // 存储在isRewardsSubmissionForAllHash
}
```

```solidity
// 活跃参与者奖励
function createRewardsForAllEarners(RewardsSubmission[] calldata submissions) {
    // 针对活跃operators及其delegators
    // 仅白名单submitter可调用
    // 存储在isRewardsSubmissionForAllEarnersHash
}
```

```solidity
function createOperatorDirectedAVSRewardsSubmission(
    address avs,
    OperatorDirectedRewardsSubmission[] calldata submissions
) external {
    // AVS自定义奖励分配
    // 基于operator表现
    // 支持多token奖励
}
```

#### createAVSRewardsSubmission

AVS 提交list of 所有earners的rewards submission

```solidity
function createAVSRewardsSubmission(
    RewardsSubmission[] calldata RewardsSubmissions
)
    external
    onlyWhenNotPaused(PAUSED_AVS_REWARDS_SUBMISSION)
    nonReentrant
```

- IERC20 token: 用于reward的ERC20 token
- uint256 amount: amount of token to transfer to the RewardsCoordinator
- uint32 startTimestamp: the start of the submission time range
- uint32 duration: the duration of the submission time range, in seconds
- StrategyAndMultiplier[] strategiesAndMultipliers: an array of StrategyAndMultiplier structs that define a linear combination of EigenLayer strategies the AVS is considering eligible for rewards. Each StrategyAndMultiplier contains:
  - IStrategy strategy: address of the strategy against which a Staker/Operator's relative shares are weighted in order to determine their reward amount
  - uint96 multiplier: the relative weighting of the strategy in the linear combination. (Recommended use here is to use 1e18 as the base multiplier and adjust the relative weightings accordingly)

example: 
ETH strategy:     multiplier = 2e18  (50%)
stETH strategy:   multiplier = 1e18  (25%)
rETH strategy:    multiplier = 1e18  (25%)

这个method perform transferfrom to transfer the specified reward token and amount from the caller to the RewardsCoordinator.

对于operator 需要通过`AVSDirectory` 注册并且在time period 里面才可以eligible to claim. 如果operator uneligible 给这个operator delegation的staker 也 uneligible to claim

#### createRewardsForAllSubmission

```solidity
function createRewardsForAllSubmission(
    RewardsSubmission[] calldata RewardsSubmissions
)
    external
    onlyWhenNotPaused(PAUSED_REWARDS_FOR_ALL_SUBMISSION)
    onlyRewardsForAllSubmitter
    nonReentrant
```

#### createOperatorDirectedAVSRewardsSubmission

```solidity
struct OperatorDirectedRewardsSubmission {
    IStrategy[] strategiesAndMultipliers;  // 策略权重
    OperatorReward[] operatorRewards;      // operator奖励
    uint32 startTimestamp;                 // 开始时间
    uint32 duration;                       // 持续时间
    IERC20 token;                         // 奖励代币
}
```

```solidity
struct OperatorReward {
    address operator;    // operator地址
    uint256 amount;     // 奖励数量
}
```

### Distributing and Claiming Rewards

The rewards updater 计算并提交 claimable roots through the following function submitRoot. They can also disable the root if it has not yet been activated:

- `RewardsCoordinator.submitRoot`
- `RewardsCoordinator.disableRoot`

Earners configure and claim these rewards using the following functions:

- `RewardsCoordinator.setClaimerFor`
- `RewardsCoordinator.processClaim`
- `RewardsCoordinator.processClaims`

#### submitRoot

distribute root:

```solidity
struct DistributionRoot {
    bytes32 root;                          // merkle树根
    uint32 activatedAt;                    // 激活时间
    uint32 rewardsCalculationEndTimestamp; // 计算结束时间
    bool disabled;                         // 是否禁用
}
```
```solidity
// 1. 时间验证
require(
    rewardsCalculationEndTimestamp > currRewardsCalculationEndTimestamp,
    "new root must be for newer calculated period"
);
require(
    rewardsCalculationEndTimestamp < block.timestamp,
    "rewardsCalculationEndTimestamp cannot be in the future"
);

// 2. 创建新的分发根
uint32 rootIndex = uint32(_distributionRoots.length);
uint32 activatedAt = uint32(block.timestamp) + activationDelay;

// 3. 存储分发根
_distributionRoots.push(
    DistributionRoot({
        root: root,
        activatedAt: activatedAt,
        rewardsCalculationEndTimestamp: rewardsCalculationEndTimestamp,
        disabled: false
    })
);

// 4. 更新时间戳
currRewardsCalculationEndTimestamp = rewardsCalculationEndTimestamp;

// 5. 触发事件
emit DistributionRootSubmitted(
    rootIndex, 
    root, 
    rewardsCalculationEndTimestamp, 
    activatedAt
);
```

#### disableRoot

```solidity
function disableRoot(
    uint32 rootIndex  // 要禁用的root index
) external 
    onlyWhenNotPaused(PAUSED_SUBMIT_DISABLE_ROOTS)
    onlyRewardsUpdater 
{
    // ... 实现逻辑
}
```

```solidity
// 1. 验证索引
require(
    rootIndex < _distributionRoots.length, 
    "invalid rootIndex"
);

// 2. 获取分发根
DistributionRoot storage root = _distributionRoots[rootIndex];

// 3. 状态验证
require(!root.disabled, "root already disabled");
require(
    block.timestamp < root.activatedAt, 
    "root already activated"
);

// 4. 禁用根
root.disabled = true;

// 5. 触发事件
emit DistributionRootDisabled(rootIndex);
```

#### setClaimerFor

```solidity
function setClaimerFor(address claimer) external {
    address earner = msg.sender;
    address prevClaimer = claimerFor[earner];
    claimerFor[earner] = claimer;
    emit ClaimerForSet(earner, prevClaimer, claimer);
}
```

```solidity
// 1. 直接认领
claimerFor[earner] == address(0)  // earner自己认领

// 2. 代理认领
claimerFor[earner] == claimer     // 由指定claimer认领
```

#### processClaim

```solidity
function processClaim(
    RewardsMerkleClaim calldata claim,
    address recipient
) external 
    onlyWhenNotPaused(PAUSED_PROCESS_CLAIM)
    nonReentrant 
{
    // 1. 获取分发根
    DistributionRoot memory root = _distributionRoots[claim.rootIndex];
    
    // 2. 验证认领
    _checkClaim(claim, root);
    
    // 3. 验证调用者
    address earner = claim.earnerLeaf.earner;
    address claimer = claimerFor[earner];
    if (claimer == address(0)) {
        claimer = earner;
    }
    require(msg.sender == claimer);
    
    // 4. 处理每个token的认领
    for (uint256 i = 0; i < claim.tokenIndices.length; i++) {
        // ... token处理逻辑
    }
}
```

token claims process:

```solidity
// Token认领流程
TokenTreeMerkleLeaf calldata tokenLeaf = claim.tokenLeaves[i];

// 1. 获取已认领金额
uint256 currCumulativeClaimed = cumulativeClaimed[earner][tokenLeaf.token];

// 2. 验证新增认领
require(
    tokenLeaf.cumulativeEarnings > currCumulativeClaimed,
    "cumulativeEarnings must be gt than cumulativeClaimed"
);

// 3. 计算认领金额
uint256 claimAmount = tokenLeaf.cumulativeEarnings - currCumulativeClaimed;

// 4. 更新已认领记录
cumulativeClaimed[earner][tokenLeaf.token] = tokenLeaf.cumulativeEarnings;

// 5. 转账
tokenLeaf.token.safeTransfer(recipient, claimAmount);

// 6. 触发事件
emit RewardsClaimed(root.root, earner, claimer, recipient, tokenLeaf.token, claimAmount);
```

merkle 验证：
```solidity
function _checkClaim(RewardsMerkleClaim calldata claim, DistributionRoot memory root) internal view {
    // 1. 基础验证
    require(!root.disabled);
    require(block.timestamp >= root.activatedAt);
    
    // 2. 长度验证
    require(claim.tokenIndices.length == claim.tokenTreeProofs.length);
    require(claim.tokenTreeProofs.length == claim.tokenLeaves.length);
    
    // 3. Earner验证
    _verifyEarnerClaimProof(
        root.root,
        claim.earnerIndex,
        claim.earnerTreeProof,
        claim.earnerLeaf
    );
    
    // 4. Token验证
    for (uint256 i = 0; i < claim.tokenIndices.length; ++i) {
        _verifyTokenClaimProof(
            claim.earnerLeaf.earnerTokenRoot,
            claim.tokenIndices[i],
            claim.tokenTreeProofs[i],
            claim.tokenLeaves[i]
        );
    }
}
```

Merkle Tree Structure:

```
Distribution Root
├── Earner Leaf 1
│   ├── Token Leaf 1
│   ├── Token Leaf 2
│   └── ...
├── Earner Leaf 2
│   ├── Token Leaf 1
│   └── ...
└── ...
```

#### processClaims

```solidity
function processClaims(
    RewardsMerkleClaim[] calldata claims,
    address recipient
) external 
    onlyWhenNotPaused(PAUSED_PROCESS_CLAIM)
    nonReentrant 
{
    for (uint256 i = 0; i < claims.length; i++) {
        _processClaim(claims[i], recipient);
    }
}
```

### system config

- `RewardsCoordinator.setActivationDelay`
- `RewardsCoordinator.setDefaultOperatorSplit`
- `RewardsCoordinator.setRewardsUpdater`
- `RewardsCoordinator.setRewardsForAllSubmitter`
- `RewardsCoordinator.setOperatorAVSsplit`
- `RewardsCoordinator.setOperatorPIsplit`

#### setActivationDelay

```solidity
function setActivationDelay(uint32 _activationDelay) external onlyOwner {
    _setActivationDelay(_activationDelay);
}

function _setActivationDelay(uint32 _activationDelay) internal {
    emit ActivationDelaySet(activationDelay, _activationDelay);
    activationDelay = _activationDelay;
}
```

- 设置DistributionRoot激活延迟时间
- 允许验证DistributionRoot有效性
- 防止恶意DistributionRoot立即生效

#### setDefaultOperatorSplit

```solidity
function setDefaultOperatorSplit(uint16 split) external onlyOwner {
    emit DefaultOperatorSplitBipsSet(defaultOperatorSplitBips, split);
    defaultOperatorSplitBips = split;
}
```

- 设置默认operator分成比例
- 用于计算operator收益
- 影响delegator收益分配

#### setRewardsUpdater

```solidity
function setRewardsUpdater(address _rewardsUpdater) external onlyOwner {
    _setRewardsUpdater(_rewardsUpdater);
}

function _setRewardsUpdater(address _rewardsUpdater) internal {
    emit RewardsUpdaterSet(rewardsUpdater, _rewardsUpdater);
    rewardsUpdater = _rewardsUpdater;
}
```

- 设置rewards updater地址
- distribution root 提交权限

#### setRewardsForAllSubmitter

```solidity
function setRewardsForAllSubmitter(address _submitter, bool _newValue) external onlyOwner {
    bool prevValue = isRewardsForAllSubmitter[_submitter];
    emit RewardsForAllSubmitterSet(_submitter, prevValue, _newValue);
    isRewardsForAllSubmitter[_submitter] = _newValue;
}
```

- 设置rewards for all submitter权限
- 控制createRewardsForAllSubmission调用权限
- 管理奖励分发权限

#### setOperatorAVSsplit

```solidity
function setOperatorAVSSplit(
    address operator,
    address avs,
    uint16 split
) external onlyWhenNotPaused(PAUSED_OPERATOR_AVS_SPLIT) {
    // 1. 验证调用者
    require(msg.sender == operator);
    
    // 2. 验证分成比例
    require(split <= 10000); // 100% in bips
    
    // 3. 获取当前分成信息
    OperatorSplit storage operatorSplit = operatorAVSSplitBips[operator][avs];
    
    // 4. 验证时间限制
    require(block.timestamp >= operatorSplit.activatedAt);
    
    // 5. 更新分成信息
    operatorSplit.oldSplitBips = operatorSplit.newSplitBips == 0 
        ? defaultOperatorSplitBips 
        : operatorSplit.newSplitBips;
    operatorSplit.newSplitBips = split;
    operatorSplit.activatedAt = uint32(block.timestamp + activationDelay);
    
    // 6. 触发事件
    emit OperatorAVSSplitBipsSet(operator, avs, split);
}
```

- 设置operator与AVS的分成比例
- 支持延迟生效机制
- 保护delegator利益

#### setOperatorPIsplit

```solidity
function setOperatorPISplit(
    address operator,
    uint16 split
) external onlyWhenNotPaused(PAUSED_OPERATOR_PI_SPLIT) {
    // 类似setOperatorAVSsplit的实现
    // 但针对程序化激励(PI)分成
}
```

- 设置operator程序化激励分成
- 类似AVS分成机制
- 独立的分成比例管理

### distribution merkle tree

两层的Merkle树结构:
1. 第一层是earner层,每个叶子节点包含:
- earner地址
- earnerTokenRoot(该earner的token子树根)

2. 第二层是token层,每个叶子节点包含:
- token地址
- 累计收益金额

```
Root (时间段T)
├── Earner 1 
│   ├── AVS_A Token rewards
│   ├── AVS_B Token rewards
│   └── AVS_C Token rewards
├── Earner 2
│   ├── AVS_A Token rewards
│   └── AVS_B Token rewards
└── ...
```

root:
```solidity
struct DistributionRoot {
    bytes32 root;        // Merkle树根hash
    uint32 activatedAt;  // 激活时间
    bool disabled;       // 是否禁用
}
```

earner leaf:
```solidity
struct EarnerTreeMerkleLeaf {
    address earner;           // 收益人地址
    bytes32 earnerTokenRoot;  // token子树的根
}   
```

token leaf:
```solidity
struct TokenTreeMerkleLeaf {
    IERC20 token;          // 代币地址
    uint256 cumulativeEarnings; // 累计收益金额
}
```

workflow:
1. 收集奖励数据
2. 构建token子树
3. 构建earner层
4. 计算最终根hash