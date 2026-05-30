# Damn Vulnerable DeFi: Unstoppable (Exploit Walkthrough & ERC-4626 Invariants)

This repository contains my solution, technical walkthrough, and deep-dive execution notes for the **Unstoppable** challenge from the renowned [Damn Vulnerable DeFi](https://damnvulnerabledefi.xyz/challenges/unstoppable/) wargame.

This challenge is a brilliant demonstration of how strict equality invariants on physical ERC-20 balances inside a contract's core logic can lead to complete, permanent Denial of Service (DoS) vulnerability vectors when confronted with out-of-band token donations.

---

## The Heist Background

A secure ERC-4626 compliant tokenized lending vault contains **1,000,000 DVT** tokens, offering flash loans for free under a grace period, or for a small fee afterward.

An on-chain **`UnstoppableMonitor`** contract continuously checks the health of the flash loan feature. If the monitor detects that the vault's flash loans are failing, it pauses the vault and transfers the vault's ownership back to the protocol deployer for review and repairs.

The player starts with **10 DVT** tokens.

### The Objective

- Stop the vault from offering flash loans.
- Trigger a state transition where the monitor contract automatically pauses the vault and transfers the vault's ownership back to the deployer.

---

## Target Architecture

The system consists of three main components:

1. **`src/UnstoppableVault.sol`**: An ERC-4626 compliant tokenized vault inheriting `ERC4626`, `ReentrancyGuard`, `Owned`, and `Pausable`. It offers flash loans for `DamnValuableToken` (DVT) and implements the standard `IERC3156FlashLender` interface.
2. **`src/UnstoppableMonitor.sol`**: A permissioned on-chain monitoring contract. It implements the `IERC3156FlashBorrower` interface and acts as the vault's owner. It exposes `checkFlashLoan()` to verify that flash loans are operational. If the flash loan fails, the monitor pauses the vault and transfers its ownership.
3. **`src/DamnValuableToken.sol`**: A basic Solmate-based ERC-20 token representing the underlying asset of the vault.

---

## The Vulnerability Explained

Our heist leverages a critical desynchronization vulnerability between **physical ERC-20 balances** and **internal ERC-4626 share tracking**.

### Strict Invariant Check

Inside `UnstoppableVault.sol`, look at the strict equality check enforced at the beginning of the `flashLoan()` function:

```solidity
uint256 balanceBefore = totalAssets();
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

Let's dissect the components of this condition:

1. **`totalAssets()` reads the physical balance:**

   ```solidity
   function totalAssets() public view override nonReadReentrant returns (uint256) {
       return asset.balanceOf(address(this));
   }
   ```

   This reads the actual balance directly from the external DVT ERC-20 token ledger. It represents the _actual_ amount of tokens physically sitting inside the vault.

2. **`convertToShares(totalSupply)` represents internal accounting:**
   Solmate's implementation of `convertToShares()` is calculated as:
   $$\text{Shares} = \frac{\text{Assets} \times \text{totalSupply}}{\text{totalAssets()}}$$
   However, passing `totalSupply` (which is in shares) as the asset input introduces a type/unit mismatch. Under normal deposit conditions, the physical token balance (`totalAssets()`) is exactly equal to the total minted shares (`totalSupply`).
   If `totalAssets() == totalSupply`, then:
   $$\text{Shares} = \frac{\text{totalSupply} \times \text{totalSupply}}{\text{totalSupply}} = \text{totalSupply}$$
   And `balanceBefore` is `totalAssets() = totalSupply`.
   Consequently, the invariant check simplifies to `totalSupply == totalSupply`, which evaluates to `true`, and the flash loan proceeds.

3. **Exploit: Direct Token Donation**
   Because ERC-20 tokens are external ledger systems, the vault contract has no control over direct transfers sent via `token.transfer()`. Anyone can transfer tokens directly to the vault, bypassing the formal `deposit()` mechanism.

   If we donate even **$1$ wei** of DVT to the vault:
   - The physical token balance increases: `totalAssets() = totalSupply + 1`
   - The expected balance becomes: `balanceBefore = totalSupply + 1`
   - However, `totalSupply` (shares) remains completely unchanged.

   Let's recalculate the internal conversion:
   $$\text{Shares} = \frac{\text{totalSupply} \times \text{totalSupply}}{\text{totalSupply} + 1}$$
   Due to Solidity's integer division, this division rounds down, evaluating to `totalSupply - 1`.

   Comparing the two sides of the invariant check:
   $$\text{convertToShares(totalSupply)} \neq \text{balanceBefore}$$
   $$\text{totalSupply} - 1 \neq \text{totalSupply} + 1$$

   This inequality causes the transaction to revert with `InvalidBalance()`. The flash loan feature is permanently bricked.

---

## Exploit Design & Execution

The monitor contract periodically calls `checkFlashLoan()` to verify that flash loans are functioning:

```solidity
try vault.flashLoan(this, asset, amount, bytes("")) {
    emit FlashLoanStatus(true);
} catch {
    emit FlashLoanStatus(false);
    vault.setPause(true);
    vault.transferOwnership(owner);
}
```

By performing a simple, direct token transfer from our player wallet to the vault contract, we desynchronize the accounting equation:

```solidity
token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE);
```

When the monitor or deployer subsequently triggers `checkFlashLoan()`, the vault's internal check reverts. The monitor catches the revert, emits `FlashLoanStatus(false)`, pauses the vault, and hands ownership back to the deployer. The exploit is executed in a single atomic transaction.

---

## Repository Guide

- **PoC**: You can find the PoC inside the Foundry test suite: [test/Unstoppable.t.sol](test/Unstoppable.t.sol).
- **Study Note**: For a complete breakdown of the ERC-4626 vault standard, the privilege escalation cascade, real-world case studies (like the Parity and Euler hacks), and security gotchas, check out: [Learnings/notes.md](Learnings/notes.md).
- **Remappings**: Foundry path configuration resides in: [remappings.txt](remappings.txt).

---

## How to Run

Ensure you have Foundry installed on your machine. To build and execute the exploit test suite, run:

```bash
# Build the smart contracts
forge build

# Run the Unstoppable exploit test in verbose mode
forge test --mt test_unstoppable -vvv
```

**_DeFi is still damn vulnerable._**
