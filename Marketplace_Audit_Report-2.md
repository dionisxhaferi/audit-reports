
# FlyBlox Marketplace Smart Contract Audit (v2.1.3)

**Audit Date**: 2025-05-22  
**Auditor**: Smart Contract Auditor (GPT)  
**Target**: MarketplacePayments.sol

---

## Findings Summary

| ID | Title                                  | Severity   |
|----|----------------------------------------|------------|
| 1  | Reentrancy Vulnerability in Transfers  | High       |
| 2  | Missing `nonReentrant` Guards          | Medium     |
| 3  | Unbounded Gas Consumption in Loop      | Low        |
| 4  | No Pause Mechanism for Emergency       | Low        |
| 5  | Lack of Event for Token List Updates   | Informational |
| 6  | Fee Misalignment Between Order & System | Informational |

---

## 1. Reentrancy Vulnerability in Transfers
**Description**:  
Functions like `markCompleteAndreleaseFundsToSeller`, `claimFundsFromBuyer`, and `returnFundsToBuyer` transfer ETH before updating the order state.

**Impact**:  
High. A malicious seller or buyer contract could re-enter and manipulate contract state before it’s updated.

**Proof of Code**:
```solidity
// Line in markCompleteAndreleaseFundsToSeller
payable(order.seller).transfer(order.amount - feeValue); // vulnerable position
order.state = State.Completed; // should be before transfer
```

**Recommended Mitigation**:
Move state updates before external calls and use a reentrancy guard.
```solidity
order.state = State.Completed;
payable(order.seller).transfer(order.amount - feeValue);
```

---

## 2. Missing `nonReentrant` Guards
**Description**:  
Public functions handling ETH and token transfers are not protected with reentrancy guards.

**Impact**:  
Medium. Increases risk of reentrancy exploit.

**Proof of Code**:
All functions like:
```solidity
function markCompleteAndreleaseFundsToSeller(...) public payable
```

**Recommended Mitigation**:
Inherit and apply OpenZeppelin’s `ReentrancyGuard`:
```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
contract MarketplacePayments is Ownable, ReentrancyGuard {
...
function markCompleteAndreleaseFundsToSeller(...) public payable nonReentrant {
```

---

## 3. Unbounded Gas Consumption in Loops
**Description**:  
While current implementation doesn’t explicitly loop through mappings, future extensions (e.g., refunds to multiple addresses) might hit gas limits.

**Impact**:  
Low. Latent issue; not currently exploitable.

**Proof of Code**: _N/A_

**Recommended Mitigation**:
Avoid unbounded loops. Use pagination or off-chain tracking.

---

## 4. No Pause Mechanism for Emergency
**Description**:  
No way to halt the contract in case of an emergency.

**Impact**:  
Low. Can't freeze operations if critical bug/exploit found.

**Proof of Code**:
```solidity
// No use of Pausable or emergency halt mechanism
```

**Recommended Mitigation**:
Use OpenZeppelin's `Pausable` contract.
```solidity
import "@openzeppelin/contracts/security/Pausable.sol";
contract MarketplacePayments is Ownable, Pausable {
    function createAndDeposit(...) public whenNotPaused { ... }
}
```

---

## 5. Lack of Event for Token List Updates
**Description**:  
`updateTokensList()` does not emit any event.

**Impact**:  
Informational. Harder for off-chain indexers to track changes.

**Proof of Code**:
```solidity
function updateTokensList(...) public onlyOwner returns (bool)
```

**Recommended Mitigation**:
Emit event:
```solidity
event TokenListUpdated(address token, bool allowed);
emit TokenListUpdated(_tokenAddress, allowed);
```

---

## 6. Fee Misalignment Between Order & System
**Description**:  
Order stores `fee` at creation, but it’s never used in later logic – contract uses the global fee.

**Impact**:  
Informational. Potential inconsistency if `fee` is changed post-order.

**Proof of Code**:
```solidity
order.fee = fee; // stored but unused
```

**Recommended Mitigation**:
Either remove from order or use it in fee calculations.

---

**End of Report**

## 7. ERC-20 Transfer Return Value Not Checked
**Description**:  
The contract assumes all ERC-20 `transfer` and `transferFrom` calls return `true`, but some tokens (e.g., USDT) do not comply strictly and may revert or return nothing.

**Impact**:  
Medium. Transfers may silently fail, causing deposits or payouts to fail unexpectedly.

**Proof of Code**:
```solidity
IERC20(order.paymentToken).transfer(order.seller, order.amount - feeValue);
```

**Recommended Mitigation**:
Use OpenZeppelin's SafeERC20 library:
```solidity
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
using SafeERC20 for IERC20;
// Then use:
IERC20(order.paymentToken).safeTransfer(order.seller, order.amount - feeValue);
```

---

## 8. Unprotected Access to createAndDeposit
**Description**:  
There's no restriction preventing contracts from calling `createAndDeposit`, potentially allowing spam via contract automation.

**Impact**:  
Low. May result in spam or denial-of-service scenarios.

**Proof of Code**:
```solidity
// No restriction to prevent contract calls
```

**Recommended Mitigation**:
Restrict to EOAs if desired:
```solidity
require(tx.origin == msg.sender, "Contracts not allowed");
```

---

## 9. Fee Calculation May Truncate to Zero
**Description**:  
For small order values or fees, the calculated fee may round to 0 due to integer division.

**Impact**:  
Low. Could result in no fees being collected on small orders.

**Proof of Code**:
```solidity
uint256 feeValue = (order.amount * fee) / 10_000;
```

**Recommended Mitigation**:
Ensure a minimum fee or warn:
```solidity
require(feeValue > 0, "Fee too small");
```

---

## 10. Lack of Order Existence Check
**Description**:  
There’s no check to ensure that an order actually exists before interacting with it. `orders[orderId]` may return an empty struct.

**Impact**:  
Medium. May allow invalid actions on non-existent orders.

**Proof of Code**:
```solidity
// No check like:
require(orderId > 0 && orderId <= totalOrders);
```

**Recommended Mitigation**:
Add an existence check:
```solidity
require(orderId > 0 && orderId <= totalOrders, "Invalid order ID");
```

---

## 11. Timestamp Manipulation by Miners
**Description**:  
Using `block.timestamp` makes the logic slightly manipulable by miners.

**Impact**:  
Low. Miners could manipulate timing of dispute or due dates by a few seconds.

**Proof of Code**:
```solidity
condition(dueDateTimestamp > (block.timestamp + 13 days), "Error: Invalid Due date")
```

**Recommended Mitigation**:
Add time buffers or consider using `block.number`.

---

## 12. Duplicate Payment Logic
**Description**:  
Token/ETH payout logic is duplicated across multiple functions.

**Impact**:  
Informational. Duplication increases maintenance and audit complexity.

**Recommended Mitigation**:
Refactor into a shared internal function:
```solidity
function _payout(Order storage order) internal { ... }
```

---
