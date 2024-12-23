## BeaconChainProofs.sol  
 
BeaconChainProofs是连接EigenLayer与信标链的关键库 EigenPod verification

### verify
1. verifyStateRoot
```solidity
function verifyStateRoot(
    bytes32 beaconBlockRoot,
    StateRootProof calldata proof
) internal view {
    // 验证状态根是否包含在区块根中
    require(
        Merkle.verifyInclusionSha256(
            proof.proof,
            beaconBlockRoot,
            proof.beaconStateRoot,
            STATE_ROOT_INDEX
        )
    );
}
```
2. verifyValidatorFields
```solidity
function verifyValidatorFields(
    bytes32 beaconStateRoot,
    bytes32[] calldata validatorFields,
    bytes calldata validatorFieldsProof,
    uint40 validatorIndex
) internal view {
    // 验证验证者信息是否包含在状态根中
    bytes32 validatorRoot = Merkle.merkleizeSha256(validatorFields);
    uint256 index = (VALIDATOR_CONTAINER_INDEX << (VALIDATOR_TREE_HEIGHT + 1)) | validatorIndex;
    
    require(
        Merkle.verifyInclusionSha256(
            validatorFieldsProof,
            beaconStateRoot, 
            validatorRoot,
            index
        )
    );
}
```


workflow:
```
BeaconBlockRoot
    └── StateRoot 
        ├── ValidatorContainer
        │   └── ValidatorFields
        └── BalanceContainer
            └── ValidatorBalance
```

## BytesLib.sol 

1. 核心操作：
- concat: concat bytes array
- slice: slice bytes array
- equal: compare bytes array

2. 类型转换：
- toAddress
- toUint8/16/32/64/96/128/256
- toBytes32

## EIP1271SignatureUtils.sol 

supports EIP1271 signature verification
## Endian.sol 

- 将little-endian的uint64转换为big-endian
- 主要用于处理beacon chain的数据

## Merkle.sol 

1. proof verification
2.  merkle tree construction
3. supports sha256 and keccak256

## StructuredLinkedList.sol 

在StrategyManager中定义strategy列表:

```solidity
// 用户地址 => 策略列表
mapping(address => StructuredLinkedList.List) internal strategiesLinkedList;
```

