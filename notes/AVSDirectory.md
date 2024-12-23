## AVSDirectory

original doc: [AVSDirectory.md](https://github.com/jaxxjj/eigenlayer-mainnet-contracts/blob/main/docs/core/AVSDirectory.md)


| File | Type | Proxy |
| -------- | -------- | -------- |
| [`AVSDirectory.sol`](../../src/contracts/core/AVSDirectory.sol) | Singleton | Transparent proxy |

### general

通过 `DelegationManager` 注册为Operator后，operator可以 通过AVS contracts register 一个或多个 AVS 来提供offchain services。当前仅有two methods AVS contracts 和 AVSDirectory interact. to track operator是否registered 了 AVS.

`AVSDirectory.registerOperatorToAVS`
`AVSDirectory.deregisterOperatorFromAVS`

### `registerOperatorToAVS`

```solidity
function registerOperatorToAVS(
    address operator,
    ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature
) 
    external 
    onlyWhenNotPaused(PAUSED_OPERATOR_REGISTER_DEREGISTER_TO_AVS)
```

AVS to register operator(通过valid signature)

- 将operator注册到AVS (operator's status to REGISTERED for the AVS)
- 可以随时更改,无需重新部署AVS

### `deregisterOperatorFromAVS`

```solidity
function deregisterOperatorFromAVS(
    address operator
) 
    external 
    onlyWhenNotPaused(PAUSED_OPERATOR_REGISTER_DEREGISTER_TO_AVS)
```

AVS to deregister operator

- 将operator从caller AVS中移除 (Sets the operator's status to UNREGISTERED for the AVS)

### `cancelSalt`

```solidity
function cancelSalt(bytes32 salt) external;
```

- 允许caller operator cancel salt (operatorSaltIsSpent[msg.sender][salt] to true)