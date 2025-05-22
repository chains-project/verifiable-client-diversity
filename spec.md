# Verifiable Client Diversity Smart Contract Specification

## 1. Overview

The VerifiableClientDiversity smart contract enables:

- Submission and on-chain verification of proofs for minority client versions.
- Configurable reward distribution to valid proof submitters.
- Owner-controlled registry of allowed client version hashes.

Key components:

- Proof Types: Supported mechanisms (SGX_ATTESTATION, RISC0).
- Allowed Versions: List of approved client version hashes.
- Reward Mechanism: Fixed payout per successful proof.

## 2. Data Structures

### 2.1 Enum: `ProofType`
```solidity
enum ProofType {
    SGX_ATTESTATION,
    RISC0
}
```

### 2.2 State Variables

| Name               | Type                          | Description                                                                                               |
| ------------------ | ----------------------------- | --------------------------------------------------------------------------------------------------------- |
| `proofType`        | `ProofType`                   | Proof mechanism. Set at contruction time.                                                                  |
| `allowedBlockDelay`| `uint8`                       | Max allowed block proof delay.                                                                            |
| `allowedVersions`  | `bytes32[]`                   | Array of approved version hashes.                                                                         |
| `versionIndex`     | `mapping(bytes32 => uint256)` | Counts the number of verified proofs for each hash. Used to track the client implementation distribution. |
| `reward`           | `uint256`                     | Reward amount (in Wei or token units).                                                                    |
| `owner`            | `address`                     | Contract owner with administrative rights.                                                                |

## 3. Access Control

Only the `owner` can:

- Call `setReward`
- Call `setsetAllowedBlockDelay`
- Call `addVersion`
- Call `removeVersion`

Ownership follows standard Ownable patterns. Transfer of ownership can be added if required.

## 4. Public Interface

### 4.1 `setReward`
```solidity
function setReward(uint256 _reward) external onlyOwner;
```
- Access: onlyOwner
- Parameters:
  - `_reward`: New reward amount.
- Emits:
    ```solidity
    RewardUpdated(uint256 oldReward, uint256 newReward)
    ```
### 4.2 `setAllowedBlockDelay`
```solidity
function setAllowedBlockDelay(uint8 _allowedDelay) external onlyOwner;
```
- Access: onlyOwner
- Parameters:
  - `_allowedDelay`: New reward amount.
- Emits:
    ```solidity
    AllowedDelayUpdated(uint8 oldDelay, uint8 newDelay)
    ```


### 4.3 `addVersion`
```solidity

function addVersion(bytes32 versionHash) external onlyOwner;
```
- Access: `onlyOwner`
- Parameters:
  - `versionHash`: Hash of the client version to enable.
- Reverts if already allowed.
- Emits:
  ```solidity
  VersionAdded(bytes32 versionHash)
  ```
  
### 4.4 `removeVersion`
```solidity
function removeVersion(bytes32 versionHash) external onlyOwner;
```
- Access: `onlyOwner`
- Parameters:
  - `versionHash`: Hash of the client version to disable.
- Reverts if not present.
- Emits:
  ```solidity
  VersionRemoved(bytes32 versionHash)
  ```
### 4.5 `getSupportedVersions`

```solidity
function getSupportedVersions() external view returns (bytes32[] memory);
```
- Access: Public
- Returns: All approved version hashes.

### 4.6 `submitProof`
```solidity
function submitProof(uint256 proofForBlock, bytes32 versionHash, bytes calldata proofData) external;
```
- Access: Public
- Parameters:
  - `proofForBlock`: The block number which the proof corresponds to.
  - `versionHash`: Approved client version hash.
  - `proofData`: Encoded proof (format per proofType).
- Workflow:
  - Check `versionHash` is allowed.
  - Check `proofForBlock`, it must be between `block.number` and `block.number + allowedBlockDelay`, inclusive. 
  - Verify proof according to proofType:
    - **SGX_ATTESTATION**: Perform attestation validation with automata-dcap-validation contract `0x95175096a9B74165BE0ac84260cc14Fc1c0EF5FF`.
    - **RISC0**: Invoke on-chain risc0 verifier contract `0x925d8331ddc0a1F0d96E68CF073DFE1d92b69187`.
  - On success, transfer reward to `msg.sender`.
- Emits:
  ```solidity
  ProofSubmitted(address indexed submitter, bytes32 versionHash, uint256 reward)
  ```
- Security: Protect against reentrancy.

### 4.7 getMinority
```solidity
function getMinority() external view returns (bytes32);
```
- Access: Public
- Returns: Version hash designated as current "minority client"
- Logic: get the allowed hash with the lowest count in the mapping.

## 5. Events

**Events are optional**

```solidity
event RewardUpdated(uint256 oldReward, uint256 newReward);
event AllowedDelayUpdated(uint8 oldDelay, uint8 newDelay);
event VersionAdded(bytes32 versionHash);
event VersionRemoved(bytes32 versionHash);
event ProofSubmitted(address indexed submitter, bytes32 versionHash, uint256 reward);
```

## 6. Error Handling

- Use require/revert with clear messages for:
  - Unauthorized calls ("Only owner").
  - Duplicate add/remove operations.
  - Invalid proof submissions.

## 7. Security Considerations
- If applicable, guard `submitProof` from reentrancy.


