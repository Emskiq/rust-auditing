# QA Report

### Risk Rating: **Low**

---

## [L-01] Not Closing the Bonding Curve Account After Migration

### **Severity**: Low  
### **Likelihood**: Low  

### **Description**
The `bonding_curve` account is not being closed after migration. This can result in a small loss of funds if the account does not meet the rent-exemption threshold. 

Without explicitly closing the account, leftover SOL used to pay for rent will remain locked in the unused account, which may accumulate over time if this process is repeated multiple times.

### **Code Reference**
```rust
#[account(
    mut,
    // add constraint that ensures the bonding curve is completed
    seeds = [BondingCurve::SEED_PREFIX.as_bytes(), token_b_mint.to_account_info().key.as_ref()],
    bump,
    close = fee_receiver, // Add this line to close the account and send SOL to the fee_receiver
)]
bonding_curve: Box<Account<'info, BondingCurve>>,
```

### **Recommendation**
- Add a `close` attribute to the `bonding_curve` account to ensure it is closed after migration.
- Ensure that the `fee_receiver` (or another suitable recipient) is designated to receive any leftover SOL.

---

## [L-02] Not Checking Whether the Bonding Curve Is Completed Before Locking the Pool

### **Severity**: Low  
### **Likelihood**: Low  

### **Description**
The `lock_pool` instruction does not validate whether the bonding curve is completed before proceeding. This could lead to inconsistencies or unexpected errors during pool locking. A partially completed bonding curve may not hold the expected reserves, resulting in incorrect behavior.

### **Code Reference**
```rust
#[account(
    mut,
    // Add a constraint to ensure the bonding curve is completed
    seeds = [BondingCurve::SEED_PREFIX.as_bytes(), token_b_mint.to_account_info().key.as_ref()],
    bump,
)]
bonding_curve: Box<Account<'info, BondingCurve>>,
```

### **Recommendation**
- Add a constraint to verify that the bonding curve is completed before allowing the `lock_pool` instruction to execute.
- For example:
  ```rust
  constraint = bonding_curve.is_completed() == true @ ContractError::BondingCurveNotCompleted,
  ```

---

## Additional Comments / Informational issues

- **Unused Accounts**: The `event_authority` account in the `LockPool` instruction is entirely unused. Consider removing it to simplify the code and reduce attack surface.
  
- **Unchecked Accounts**: Some accounts, such as `system_program`, `token_program`, and `associated_token_program`, are marked as `UncheckedAccount`. Consider replacing these with more explicit `Program<'info, X>` declarations to improve safety.

- **Use of `init_if_needed`**: For certain accounts, such as `escrow_vault`, using `init_if_needed` might prevent unnecessary failures during initialization.

- **Metadata Account Validation**: Ensure that the `metadata` account is properly validated (e.g., using seeds or a proper PDA check) to avoid misuse.

---

By addressing the issues identified above, the protocol can reduce potential vulnerabilities and ensure smoother operation.
