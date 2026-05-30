# EVM Mechanics & Vuln Patterns: Unstoppable Study Notes

I am writing these down while the concepts are still fresh in my head. Bricking Unstoppable was a masterclass in how strict equality invariants on physical ERC-20 balances can create Denial of Service (DoS) vectors. I need to make sure I completely internalize these EVM execution concepts, ERC-4626 accounting rules, and security gotchas so I can recognize them instantly in future exploits and code audits.

---

## 1. ERC-4626: The Tokenized Vault Standard

The ERC-4626 standard defines a unified API for yield-bearing vaults. Before this standard, protocols built custom, bespoke yield structures (like cTokens in Compound or yTokens in Yearn), leading to fragmentation and dangerous integration bugs.

### Assets vs. Shares

Under the hood, an ERC-4626 vault manages two primary layers:

1. **The Underlying Asset (A):** The base ERC-20 token deposited into the vault (e.g., DAI, WETH, USDC).
2. **The Vault Shares (S):** The vault's own ERC-20 tokens minted to depositors, representing their fractional ownership of the underlying assets.

As the vault earns yield, the amount of underlying assets increases, but the number of outstanding shares remains the same. Thus, the exchange rate of shares to assets grows over time:

```text
Share Price = Total Assets in Vault / Total Supply of Shares
```

### Core API Flow

- **Inflows (Depositing):**
  - `deposit(uint256 assets, address receiver) -> uint256 shares` (rounds shares down)
  - `mint(uint256 shares, address receiver) -> uint256 assets` (rounds assets up)
- **Outflows (Withdrawing):**
  - `withdraw(uint256 assets, address receiver, address owner) -> uint256 shares` (rounds shares up)
  - `redeem(uint256 shares, address receiver, address owner) -> uint256 assets` (rounds assets down)
- **Calculations & Projections:**
  - `totalAssets()`: returns the total underlying assets held/managed by the vault.
  - `convertToShares(uint256 assets)` / `convertToAssets(uint256 shares)`: pure mathematical rate projections.

---

## 2. The Invariant Collision: Physical Balances vs. Internal Accounting

In Solidity, there is a massive architectural difference between **internal contract accounting** (how much the contract thinks it has) and **physical token balances** (how much the ERC-20 token contract says this address has).

### The Vulnerability Mechanics

In `UnstoppableVault.sol`, look at the strict invariant check in the `flashLoan` function:

```solidity
uint256 balanceBefore = totalAssets();
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

Let's analyze why this equation is extremely fragile:

1. **`totalAssets()` reads the physical balance:**

   ```solidity
   function totalAssets() public view override nonReadReentrant returns (uint256) {
       return asset.balanceOf(address(this));
   }
   ```

   This reads `asset.balanceOf` directly from the external ERC-20 contract. It represents the _actual_ amount of tokens physically resting in the vault.

2. **`convertToShares(totalSupply)` represents internal accounting:**
   Solmate's implementation of `convertToShares` calculates:
   ```text
   Shares = (Assets * totalSupply) / totalAssets()
   ```
   When the input asset amount is passed as `totalSupply` (a type/unit mismatch, as it passes shares instead of assets), the equation simplifies to:
   ```text
   Shares = (totalSupply * totalSupply) / totalAssets()
   ```
   Under normal conditions, where the vault has never received direct donations and only accepts tokens via `deposit()`, the physical balance of tokens (`totalAssets()`) is exactly equal to `totalSupply`.

   If `totalAssets() == totalSupply`, then:
   ```text
   Shares = (totalSupply * totalSupply) / totalSupply = totalSupply
   ```
   And `balanceBefore` is `totalAssets() = totalSupply`.

   Therefore, the check passes: `totalSupply == totalSupply`.

3. **The Silent Killer: Direct Token Donation**
   If an external attacker bypasses the formal `deposit()` mechanism and transfers even **1 wei** of the underlying asset directly to the vault using a raw `transfer` call, the physical token balance (`totalAssets()`) increases, but the `totalSupply` of shares remains completely unchanged.

   Let's do the math if 1 wei is donated:
   - `totalAssets() = totalSupply + 1`
   - `balanceBefore = totalSupply + 1`
   - Now, calculate `convertToShares(totalSupply)`:
     ```text
     Shares = (totalSupply * totalSupply) / (totalSupply + 1)
     ```
     Because of Solidity's integer division, this calculation rounds down to **strictly less than `totalSupply`** (specifically, `totalSupply - 1` when `totalSupply` is large).

   Now, evaluating the invariant check:
   ```text
   convertToShares(totalSupply) != balanceBefore
   (totalSupply - 1) != (totalSupply + 1)
   ```

   The equation fails, the transaction reverts, and the flashloan functionality is completely bricked forever.

---

## 3. The Attacker's Mindset: Hunting for Asymmetric Invariants

How does a security auditor or attacker systematically uncover this vulnerability without relying on luck? It comes down to a repeatable mental framework designed to locate and break **Asymmetric State Invariants**.

When I look at any contract, my brain executes a 4-step threat-modeling sequence:

### Step 1: Mapping the Gatekeepers (Locate the Reverts)

Every exploit needs an entry point that behaves differently than intended. I look for all conditional reverts (`revert()`, `require()`, `assert()`). In `UnstoppableVault`, I zoom in on:

```solidity
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

This is my target. If I want to trigger a Denial of Service (DoS) and stop flashloans completely, my objective is: **Force this specific conditional check to always evaluate to `true` (and thus revert).**

### Step 2: Classifying the Variables (The Asymmetric Split)

Next, I map out where the variables in the check get their values:

- **Left Side (`convertToShares(totalSupply)`):** This represents the contract's **internal state** (tracked shares). It can only be changed by calling functions defined _inside_ this contract (like `deposit()` or `withdraw()`).
- **Right Side (`balanceBefore` / `totalAssets()`):** This represents the contract's **external state** (tracked by the external ERC-20 token contract).

This is a classic **Asymmetric Invariant**: a comparison between an internal state variable and an external physical balance.

### Step 3: Finding the Out-of-Band Vector (The Asymmetric Injection)

Now, I ask the golden question:

> _"Is there a way to update the external state (Right Side) without triggering the internal state transition logic that updates the Left Side?"_

In Ethereum, ERC-20 tokens are external contracts. When someone calls `ERC20.transfer(vault, amount)`, the token contract updates its ledger. The vault contract itself has absolutely no control or awareness of this incoming transfer unless it implements a custom callback (which standard ERC-20s do not trigger).

Therefore: **Yes! I can unilaterally inflate the Right Side (`balanceBefore`) by directly calling `transfer()` on the token contract.**

### Step 4: Assessing Irreversibility (The Permanent Brick)

A good exploit shouldn't be easy to recover from. Once I donate the 1 wei and break the equation, can the owner or a user fix it?

- To restore the balance equation, the Left Side would need to increase (by minting more shares without increasing assets—impossible) OR the Right Side would need to decrease (by withdrawing or transferring out the donated assets).
- In `UnstoppableVault`, there are no functions to withdraw assets without burning shares, nor is there a "sweep" function for undocumented tokens.
- The donated tokens are permanently trapped, meaning **the desynchronization is permanent, and the vault is bricked forever.**

By breaking down comparison checks into **internal vs. external state** and looking for out-of-band manipulation vectors, these types of DoS vectors instantly pop out of the code.

---

## 4. Designing Secure Vaults & Preventing Inflation

### Rule 1: Never Use Strict Equality for External Token Balances

Because anyone can send ERC-20 tokens or Ether directly to a contract address, you must never assume that the contract's physical balance strictly equals its internal state variables.

Instead of checking strict equality, either:

- Allow surplus balances to simply act as yield (accumulating to existing share value).
- Verify that balances are _at least_ a threshold, rather than exactly equal:
  ```solidity
  require(totalAssets() >= balanceBefore + fee, "Repayment failed");
  ```

### Rule 2: Implement Internal Bookkeeping (State Tracking)

If your contract needs to track exactly how much it expects to hold, use an internal state variable (e.g., `uint256 private totalManagedAssets`) that is only modified during formal contract calls (like `deposit`, `withdraw`, `collectFee`), ignoring the raw ERC-20 `balanceOf(address(this))` directly.

### Rule 3: Guard Against the First Deposit Bug (Inflation Attack)

When `totalSupply == 0`, an attacker can manipulate the exchange rate by depositing 1 wei, and donating 10,000 tokens to steal subsequent depositors' funds.

- To fix this, mint **virtual shares and assets** (a small offset in the calculation, like OpenZeppelin's ERC-4626 virtual offsets) to ensure the division never rounds down to zero.
- Alternatively, burn/lock a small amount of the very first minted shares (e.g., 1,000 or 1,000,000,000 wei of shares) to a dead address (`address(0)`) so the exchange rate can never be radically skewed.

---

## 5. Privilege Escalation Cascade: How DoS Invariants Lead to Total System Ruin

While this specific challenge only requires us to trigger a Denial of Service (DoS) by bricking the flashloan capability, we must realize the catastrophic "Butterfly Effect" of this bug in a real-world production deployment.

In a real DeFi system, a minor desynchronization DoS does not happen in a vacuum. It triggers secondary fallback states that can lead to complete, irreversible protocol compromise.

### 1. The Trapdoor: `UnstoppableVault.execute()`

Take a close look at this function in [UnstoppableVault.sol](src/UnstoppableVault.sol#L124-L127):

```solidity
function execute(address target, bytes memory data) external onlyOwner whenPaused {
    (bool success,) = target.delegatecall(data);
    require(success);
}
```

This is a standard admin escape hatch. It allows the owner of the vault to execute arbitrary changes via a low-level `delegatecall` while the vault is paused.
However, **`delegatecall` in the context of the vault means the caller can execute ANY logic on behalf of the vault, rewrite its storage, or sweep all of its ERC-20 assets.**

### 2. The Dangerous Fallback Loop

When we brick the vault's flashloans using our 1 wei donation exploit, how does the protocol react?

1. The **`UnstoppableMonitor`** contract detects the flashloan failure.
2. It automatically calls `vault.setPause(true)`.
3. It then calls `vault.transferOwnership(owner)`.

Here, the ownership is transferred back to the safe `deployer` address. But in a slightly different architecture—for instance, if the monitor's ownership logic had a privilege escalation vulnerability, or if the `transferOwnership` function lacked strict access controls—an attacker could hijack the ownership during this fallback state.

**Once an attacker gains ownership of the vault, the vault is immediately dead.** The attacker simply calls `execute()` with a malicious `delegatecall` payload, approving their own address to transfer out all 1,000,000 DVT tokens, resulting in a total asset drain.

### 3. Real-World Lessons: The Danger of Cascading State Shifts

In smart contract security, we frequently see seemingly "harmless" bugs (like DoS or rounding errors) acting as the keys that unlock catastrophic exploits.

#### Anecdote A: The Parity Multi-Sig Hack (2017) — 150M Frozen

- The Vulnerability: The Parity multi-sig library contract had an uninitialized owner state.
- The Trigger: A user called `initWallet()` on the library contract, successfully claiming ownership of it.
- The Cascade: In a panic or out of malice, the new "owner" called the `kill()` function, which triggered `selfdestruct`.
- The Ruin: Because all Parity multi-sig wallets on-chain relied on this library's code via `delegatecall`, destroying the library instantly froze over 150,000,000 worth of Ether forever. A simple lack of initialization access control led to permanent system death.

#### Anecdote B: The Euler Finance Donation Attack (2023) — 197M Stolen

- The Vulnerability: Euler allowed users to directly "donate" their eTokens (collateral) to the reserve pool.
- The Exploit: An attacker minted a massive amount of collateral and debt, and then called `donateToReserves()`.
- The Cascade: By intentionally donating their collateral, their health score fell below 1 (liquidation threshold), but they still held the un-liquidated debt.
- The Ruin: This undercollateralized state allowed them to liquidate themselves using a secondary account at a massive discount, draining 197,000,000 from the protocol. This is a direct real-life example of using **deliberate token donation** to break accounting invariants.

---

## 6. Audit Red Flags & Pattern Recognition

When auditing vaults or flash loan systems, look out for:

- **Strict Equality (`==`, `!=`) Checks on physical balances:** Any check comparing `balanceOf(address(this))` directly with state variables or expectations is a critical DoS candidate.
- **Unit / Type Mismatches in Conversions:** Watch out for calculations where a developer passes a share amount into a function expecting an asset amount (e.g. `convertToShares(shares)` or `convertToAssets(assets)`).
- **First Deposit Unprotected:** If a vault does not lock initial shares or implement virtual assets/shares, it is vulnerable to exchange rate skewing via early donation.
- **Dangerous Admin `delegatecall` Escape Hatches:** Any function allowing arbitrary low-level calls (`delegatecall`, `call`) under admin ownership must be audited with extreme skepticism. If ownership is compromised, the entire pool is lost.
