# TODO 2024-10-superposition
---
- Category: chose from [[protocol_categories]]
- Note Create 2025-01-22
- Platform: code4rena
- Report Url: [2024-10-superposition Reports](https://code4rena.com/reports/2024-10-superposition)
- Competitive Url: [2024-10-superposition](https://code4rena.com/audits/2024-10-superposition)
---
# High Risk Findings (03)

---
## [H-01] `createPoolD650E2D0` will not work due to mismatch in solidity and stylus function definitions
----
- **Tags**:  #context #function_parm
- Number of finders: 1
- Difficulty: Easy
---
### Proof of Concept

`SeaWaterAMM.sol` holds the `createPoolD650E2D0` which allows the admin to initialize a new pool. It calls the `create_pool_D650_E2_D0` function in the stylus.

As can be seen from the `create_pool_D650_E2_D0` function, it takes in the token address, `sqrtPriceX96` and fee.

```rust
	pub fn create_pool_D650_E2_D0(
		&mut self,
		pool: Address, // The token address
		price: U256,   // The sqrt price scaled by 2^96
		fee: u32,      // The fee amount
	) -> Result<(), Revert> {
	//...
	}
```

But `createPoolD650E2D0`’s definition takens in more, token address, `sqrtPriceX96`, fee, tick spacing and `maxLiquidityPerTick`, causing a mismatch between the function definitions of the Solidity and Stylus contracts.

```solidity
    function createPoolD650E2D0(
        address /* token */,
        uint256 /* sqrtPriceX96 */,
        uint32 /* fee */,
        uint8 /* tickSpacing */,
        uint128 /* maxLiquidityPerTick */
    ) external {
        directDelegate(_getExecutorAdmin());
    }
```

Calls to the function will always fail, breaking `SeawaterAMM.sol`’s functionality to create a pool position.
### Recommended Mitigation

Remove the unneeded parameters.

```rust
    function createPoolD650E2D0(
        address /* token */,
        uint256 /* sqrtPriceX96 */,
        uint32 /* fee */,
-       uint8 /* tickSpacing */,
-       uint128 /* maxLiquidityPerTick */
    ) external {
        directDelegate(_getExecutorAdmin());
    }
```

### Discussion

af-afk (Superposition) confirmed:

0xsomeone (judge) increased severity to High and commented:

> The Warden has correctly identified that the function definitions of the Solidity and Stylus contracts differ, resulting in the relevant functionality of the system being inaccessible.
> 
> In line with the previous audit’s rulings, I believe a high-risk rating is appropriate for this submission as pool creations are rendered inaccessible via any other functions in contrast to the original audit’s submission which permitted circumvention of this error.

DadeKuma (warden) commented:

> I believe some key pieces of information are missing to provide an accurate severity assessment, which I will address in this comment.
> 
> It is true that `createPoolD650E2D0` will not work if directly called, as it has the wrong ABI, and this finding is technically valid. However, there is a fallback function that allows the creation of new pools by using the correct ABI.
> 
> The correct ABI, like this issue points, is the following:
> 
> `createPoolD650E2D0(address,uint256,uint32)`
> 
> So the third byte is `0x80`:
```rust
function testAbi() public pure returns (bytes1) {
    return abi.encodeWithSignature("createPoolD650E2D0(address,uint256,uint32)", address(0), 0, 0)[2];
}
```
>> decoded output { “0”: “bytes1: 0x80” }
>If we look at the fallback function, the execution will fall under the executor fallback:
```rust
fallback() external {
    // swaps
    if (uint8(msg.data[2]) == EXECUTOR_SWAP_DISPATCH)
        directDelegate(_getExecutorSwap());
        // update positions
    else if (uint8(msg.data[2]) == EXECUTOR_UPDATE_POSITION_DISPATCH)
        directDelegate(_getExecutorUpdatePosition());
        // positions
    else if (uint8(msg.data[2]) == EXECUTOR_POSITION_DISPATCH)
        directDelegate(_getExecutorPosition());
        // admin
    else if (uint8(msg.data[2]) == EXECUTOR_ADMIN_DISPATCH)
        directDelegate(_getExecutorAdmin());
        // swap permit 2
    else if (uint8(msg.data[2]) == EXECUTOR_SWAP_PERMIT2_A_DISPATCH)
        directDelegate(_getExecutorSwapPermit2A());
        // quotes
    else if (uint8(msg.data[2]) == EXECUTOR_QUOTES_DISPATCH) directDelegate(_getExecutorQuote());
    else if (uint8(msg.data[2]) == EXECUTOR_ADJUST_POSITION_DISPATCH) directDelegate(_getExecutorAdjustPosition());
    else if (uint8(msg.data[2]) == EXECUTOR_SWAP_PERMIT2_B_DISPATCH) directDelegate(_getExecutorSwapPermit2B());
->  else directDelegate(_getExecutorFallback());
}
```

>Current values:
```rust
uint8 constant EXECUTOR_SWAP_DISPATCH = 0;
uint8 constant EXECUTOR_UPDATE_POSITION_DISPATCH = 1;
uint8 constant EXECUTOR_POSITION_DISPATCH = 2;
uint8 constant EXECUTOR_ADMIN_DISPATCH = 3;
uint8 constant EXECUTOR_SWAP_PERMIT2_A_DISPATCH = 4;
uint8 constant EXECUTOR_QUOTES_DISPATCH = 5;
uint8 constant EXECUTOR_ADJUST_POSITION_DISPATCH = 6;
uint8 constant EXECUTOR_SWAP_PERMIT2_B_DISPATCH = 7;
```

>Moreover, creating a pool is permissionless and [intended](https://github.com/code-423n4/2024-10-superposition-findings/issues/9#issuecomment-2461280457) by the Sponsor, it doesn’t have to be called by the executor admin; the executor fallback would be able to create new pools.
>
  Therefore, there is no loss of funds, and the functionality of the protocol is not impacted in this way; I don’t see how a High risk can be justified. I believe this issue falls under the QA umbrella, as a function does not work according to specifications.

0xsomeone (judge) commented:

> @DadeKuma - This submission’s assessment is in line with the previous audit and the presence of a fallback mechanism is not sufficient to justify the finding’s invalidation. Otherwise, any inaccessible functionality of the system could be argued as being present in the fallback function and all findings pertaining to it would have to be invalidated in the previous audit as well.
> 
> Given that wardens were aware of the judgment style of this particular submission type, I do not believe that downgrading it in the follow-up round is a fair approach.

### Notes

Function signatures must match exactly between Solidity and Rust.
- Same parameter count
- Same parameter types
- Same parameter order

### Tools
### Refine

{{ Refine to typical issues}}

---
## [H-02] Users are incorrectly refunded when liqudity is insufficient
----
- **Tags**: #context #business_logic #refund
- Number of finders: 4
- Difficulty: Medium
---
### Lines of code

[lib.rs#L287-L297](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L287-L297)
```rust
        erc20::take(from, amount_in, permit2)?;
        erc20::transfer_to_sender(to, amount_out)?;

        if original_amount > amount_in {
            erc20::transfer_to_sender(
                to,
                original_amount
                    .checked_sub(amount_in)
                    .ok_or(Error::TransferToSenderSub)?,
            )?;
        }
```
### Proof of Concept

1. Context

In `swap_2_internal`, if the first pool doesn't have enough liquidity, `amount_in`could be less than `original_amount`, and as expected, `amount_in` is taken from swapper. But the function still refunds `original_amount - amount_in` to the user if `original_amount` is more than `amount_in`.

2. Bug Location

From the function, we can see than `amount_in` is taken from swapper. Then the function checks if `original_amount` is more than `amount_in`, before which the difference is transferred back to the sender.

```rust
	// First, the function takes only the amount_in from the user
	erc20::take(from, amount_in, permit2)?;

	// Then it transfers the swap output to the user
	erc20::transfer_to_sender(to, amount_out)?;

	// The bug: It checks if original_amount > amount_in
	if original_amount > amount_in {
		// And refunds the difference!
		erc20::transfer_to_sender(
			to,
			original_amount
				.checked_sub(amount_in)
				.ok_or(Error::TransferToSenderSub)?,
		)?;
	}
```

3. Final Effect

An unnecessary refund is processed leading to loss of funds for the protocol. Malicious users can take advantage of this to "rob" the protocol of funds through the refunds.
### Recommended Mitigation

No need to process refunds since `amount_in` is already taken.

```
        erc20::take(from, amount_in, permit2)?;
        erc20::transfer_to_sender(to, amount_out)?;

-       if original_amount > amount_in {
-           erc20::transfer_to_sender(
-               to,
-               original_amount
-                   .checked_sub(amount_in)
-                   .ok_or(Error::TransferToSenderSub)?,
-           )?;
        }
```

### Discussion

### Notes

#### Notes 

Imagine a user named Alice who wants to swap tokens in a liquidity pool:

1. Alice wants to swap 1000 USDC for ETH
2. The pool has limited liquidity and can only process 200 USDC
3. Let's see what happens with the buggy code

##### What Should Happen

In a correctly functioning swap:

- Alice should send 200 USDC to the pool
- Alice should receive some ETH in return (let's say 0.1 ETH)
- No refund should happen (since Alice only sent 200 USDC)

##### What Actually Happens (With the Bug)

Here's how the buggy code executes:

1. `original_amount` = 1000 USDC (what Alice wants to swap)
2. `amount_in` = 200 USDC (what the pool can actually handle)
3. `amount_out` = 0.1 ETH (what Alice gets in return)

Now the code executes these steps:

```rust
// Alice sends 200 USDC to the contract
erc20::take(from, amount_in, permit2)?;  // Takes 200 USDC from Alice

// Alice receives 0.1 ETH from the swap
erc20::transfer_to_sender(to, amount_out)?;  // Sends 0.1 ETH to Alice

// The BUG: Check if original_amount > amount_in
if original_amount > amount_in {  // 1000 > 200, so this is true
    // INCORRECT REFUND: Alice gets 800 USDC back, but she never sent it!
    erc20::transfer_to_sender(
        to,
        original_amount.checked_sub(amount_in)?,  // 1000 - 200 = 800 USDC
    )?;
}
```

##### The End Result

Alice's balance changes:

- Initially: 1000 USDC, 0 ETH
- After the swap: 800 USDC, 0.1 ETH

The protocol's balance changes:

- Initially: 0 USDC, X ETH
- After the swap: 200 USDC, (X - 0.1) ETH

So Alice effectively:

1. Paid 200 USDC
2. Received 0.1 ETH
3. **PLUS got an additional 800 USDC "refund" from the protocol's funds**

Alice has magically gained 800 USDC from nowhere! This is because the protocol is refunding Alice for tokens she never actually sent.

#### Impressions
- Wrong refund logic

### Tools
### Refine
#business_logic 

---
## [H-03] No slippage control when withdrawing a position leads to loss of funds
----
- **Tags**:  #sandwich #withdraw #slippage #context
- Number of finders: 1
- Difficulty: Hard:
---
### Lines of code
[lib.rs#L745-L757](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L745-L757)
```rust
impl Pools {
    /// Refreshes and updates liquidity in a position, using approvals to transfer tokens.
    /// See [Self::update_position_internal].
    #[allow(non_snake_case)]
    pub fn update_position_C_7_F_1_F_740(
        &mut self, //the function can modify the state of the `Pools` struct it belongs to
        pool: Address,
        id: U256,
        delta: i128,
    ) -> Result<(I256, I256), Revert> {
        self.update_position_internal(pool, id, delta, None)
    }
}
```

```rust
pub fn update_position_internal(
        &mut self, //This function can modify the state of the `Pools` struct.
        pool_addr: Address, //the specific liquidity pool.
        id: U256, //The unique identifier of the position
        delta: i128, //The amount of liquidity to add (positive) or remove (negative).
        permit2: Option<(Permit2Args, Permit2Args)>,//Optional authorization parameters for token transfers.
    ) -> Result<(I256, I256), Revert> {
        assert_eq_or!(
            msg::sender(),
            self.position_owners.get(id),
            Error::PositionOwnerOnly
        ); // ownership check

        let mut pool = self.pools.setter(pool_addr);
        let (token_0, token_1) = pool.update_position(id, delta)?;


        if delta < 0 { // withdraw
            // if we're sending to sender, make sure that the pool is initialised.
            assert_or!(pool.initialised.get(), Error::PoolDisabled); // safety check
            erc20::transfer_to_sender(pool_addr, token_0.abs_neg()?)?; // transfer back token0
            erc20::transfer_to_sender(FUSDC_ADDR, token_1.abs_neg()?)?; // transfer back token1
        } else { // deposit 
            // if we're TAKING, make sure that the pool is enabled.
            assert_or!(pool.enabled.get(), Error::PoolDisabled);
            let (permit_0, permit_1) = match permit2 {
                Some((permit_0, permit_1)) => (Some(permit_0), Some(permit_1)),
                None => (None, None),
            };


            erc20::take(pool_addr, token_0.abs_pos()?, permit_0)?;
            erc20::take(FUSDC_ADDR, token_1.abs_pos()?, permit_1)?;
        }


        evm::log(events::UpdatePositionLiquidity {
            id,
            token0: token_0,
            token1: token_1,
        }); // logging the event


        Ok((token_0, token_1))
    }
```
### Impact

An attacker can sandwich a user withdrawing funds as there is no way to put slippage protection, which will cause a large loss of funds for the victim.
### Proof of Concept
`decr_position_09293696` function was removed entirely. Now, the only way for users to withdraw funds is by calling `update_position_C_7_F_1_F_740` with negative delta.

The issue is that in this way, users can't have any slippage protection. `decr_position` allowed users to choose an `amount_0_min` and `amount_1_min` of funds to receive, which is now zero.

This allows an attacker to sandwich their withdrawal to steal a large amount of funds.
### Recommended Mitigation

Consider reintroducing a withdrawal function that offers slippage protection to users (they should be able to choose `amount_0_min, amount_1_min, amount_0_desired`, and `amount_1_desired`).

### Discussion
**af-afk (Superposition) acknowledged**

**0xsomeone (judge) commented:**

> The submission has demonstrated that liquidity withdrawals from the system are inherently insecure due to being open to arbitrage opportunities as no slippage is enforced.
> 
> I am unsure why the Sponsor has opted to acknowledge this submission as it is a tangible vulnerability and one that merits a high-risk rating. The protocol does not expose a secure way to natively extract funds from it whilst offering this functionality for other types of interactions.

**af-afk (Superposition) commented:**

> @0xsomeone - we won’t fix this for now since Superposition has a centralised sequencer, and there’s no MEV that’s possible for a third-party to extract using the base interaction directly with our provider.

**DadeKuma (warden) commented:**

> @af-afk - I highly suggest fixing this issue, as a centralized sequencer does not prevent MEV extraction. 

### Notes

Withdrawals particularly need  Slippage Protection

### Tools
### Refine

{{ Refine to typical issues}}

---
# Medium Risk Findings (03)

---
## [M-02] Tokens are pulled from users without verifying pool status contrary to requirement
----
- **Tags**: 
- Number of finders: 2
- Difficulty: Medium
---
### Lines of code

[lib.rs#L679-L687](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L679-L687)
```rust
            // if we're TAKING, make sure that the pool is enabled.
            assert_or!(pool.enabled.get(), Error::PoolDisabled);
            let (permit_0, permit_1) = match permit2 {
                Some((permit_0, permit_1)) => (Some(permit_0), Some(permit_1)),
                None => (None, None),
            };

            erc20::take(pool_addr, token_0.abs_pos()?, permit_0)?;
            erc20::take(FUSDC_ADDR, token_1.abs_pos()?, permit_1)?;
```

[lib.rs#L716-L738](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L716-L738)
```rust
        let (amount_0, amount_1) =
            self.pools
                .setter(pool)
                .adjust_position(id, amount_0_desired, amount_1_desired)?;

        evm::log(events::UpdatePositionLiquidity {
            id,
            token0: amount_0,
            token1: amount_1,
        });

        let (amount_0, amount_1) = (amount_0.abs_pos()?, amount_1.abs_pos()?);

        assert_or!(amount_0 >= amount_0_min, Error::LiqResultTooLow);
        assert_or!(amount_1 >= amount_1_min, Error::LiqResultTooLow);

        let (permit_0, permit_1) = match permit2 {
            Some((permit_0, permit_1)) => (Some(permit_0), Some(permit_1)),
            None => (None, None),
        };

        erc20::take(pool, amount_0, permit_0)?;
        erc20::take(FUSDC_ADDR, amount_1, permit_1)?;
```
### Proof of Concept

Both the `update_position_internal()` and `adjust_position_internal()` functions are responsible for managing token positions, which involves taking tokens from users. However, there is a critical inconsistency in how each function verifies the operational status of the liquidity `pool` before performing token transfers from the user.

[`update_position_internal()`](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L679-L687)  
This function checks if the `pool` is `enabled` before taking tokens from the user.
```rust
            // if we're TAKING, make sure that the pool is enabled.
            assert_or!(pool.enabled.get(), Error::PoolDisabled);
            ---SNIP---
            erc20::take(pool_addr, token_0.abs_pos()?, permit_0)?;
            erc20::take(FUSDC_ADDR, token_1.abs_pos()?, permit_1)?;
```

[`adjust_position_internal()`](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L716-L738)  
This does not explicitly check whether the `pool` is `enabled` before proceeding.
```rust
    let (amount_0, amount_1) =
            self.pools
                .setter(pool)
                .adjust_position(id, amount_0_desired, amount_1_desired)?;
    ---SNIP---
    erc20::take(pool, amount_0, permit_0)?;
    erc20::take(FUSDC_ADDR, amount_1, permit_1)?;
```

First, it calls `self.pools.setter(pool).adjust_position(...)` which has the following [comment](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/pool.rs#L222-L225):

```rust
    // [update_position] should also ensure that we don't do this on a pool that's not currently running
    self.update_position(id, delta)
```

The comment in the `adjust_position()` function implies that a check for the pool's operational state is necessary and should be enforced in `update_position()`. However, `update_position()` function does not make such enforcement as it does not check for pool status.
### Recommended Mitigation

Modify the `adjust_position_internal()` function to include a status check before executing the position adjustment:

```rust
    // Ensure the pool is enabled before making any adjustments
+   assert_or!(pool.enabled.get(), Error::PoolDisabled);
```
### Discussion

**af-afk (Superposition) acknowledged and commented via duplicate Issue #4:**

> We made the decision that we were going to allow this functionality for now. The reason being that in a programmatic context, the pool can be controlled to be enabled and disabled depending on the broader environment. We use this for example with 9lives to prevent trading of a market that’s expired, in lieu of remembering ownership at different time points of the asset. We made the decision that allowing people to supply liquidity could be useful in the future, and for this reason we also allowed supplying liquidity to a frozen pool as well.

**0xsomeone (judge) commented via duplicate Issue #4:**

> The submission claims that an issue submitted in the original audit was not resolved properly; however, the Sponsor’s choice to acknowledge the issue does not contradict the original issue’s acceptance. As this audit is a follow-up one, it would have been helpful to discuss with the Sponsor directly about their intentions on how to resolve issue #31 of the original audit.
> 
> I believe that the issue is invalid based on the Sponsor’s intentions.

**DadeKuma (warden) commented via duplicate Issue #4:**

> @0xsomeone - The original issue was fixed, but it introduced another bug (this issue).
> 
> Code documentation `clearly states` that adding liquidity to disabled pools shouldn’t be possible:
> 
> Requires the pool to be enabled unless removing liquidity. Moreover, it’s actually `not possible` to add liquidity to disabled pools by using the following path, like the documentation suggests:
> 
> `update_position_C_7_F_1_F_740 > update_position_internal`
> 
> But it is still possible by using the path described in this issue: `incr_position_E_2437399 > adjust_position_internal > adjust_position > update_position`
> 
> Even if the documentation states that this shouldn’t be possible:
> 
> > `// [update_position]` should also ensure that we don’t do this on a pool that’s not currently `// running`
> 
> This is clearly a discrepancy, and I strongly believe this is a valid issue based on the information available during the audit.

**0xsomeone (judge) commented via duplicate Issue #4:**

> @DadeKuma - I believe the discrepancies between the documentation and the implementation are adequate to merit a proper medium-risk vulnerability rating and have re-instated it so.
### Notes & Impressions

#### Impressions
Permission or state checks should be consistently applied across ***all_code_paths*** that perform the same or similar sensitive operations.
- Simply check similar functions have the similar permission or state checks

### Tools
### Refine

{{ Refine to typical issues}}

---
## [M-03] Incorrect slippage handing in `swap_internal()`
----
- **Tags**: #invalid_validation #business_logic 
- Number of finders: 1
- Difficulty: Medium
---
### Proof of Concept

In the [`swap_internal()`](https://github.com/code-423n4/2024-10-superposition/blob/7ad51104a8514d46e5c3d756264564426f2927fe/pkg/seawater/src/lib.rs#L183-L186) function, the slippage check uses the `||` operator to validate the swap results. This can lead to a scenario where one of the amounts (`amount_0_abs` or `amount_1_abs`) is allowed to be zero, potentially resulting in unwanted `slippage`.

```rust
    assert_or!(
        amount_0_abs > U256::zero() || amount_1_abs > U256::zero(), // Problematic operator
        Error::SwapResultTooLow
    );
```

Using the `||` operator allows the swap to proceed even if one of the amounts is `zero`, which could lead to unacceptable slippage.

**Scenario**:

- Consider a user swapping `100` units of token A (`amount_0`) for token B (`amount_1`).
- Due to slippage, `token B`’s output (`amount_1_abs`) becomes `zero`, while token A’s output (`amount_0_abs`) remains positive.
- With the current `||` operator, the `swap` would still be considered `valid` since one amount is greater than zero, even though the user receives no token B (`amount_1_abs = 0`), resulting in a poor outcome for the user.

### Impact

Using the `||` operator means that one token amount can be zero while the other passes the check, leading to an imbalanced swap that might not meet user expectations.
### Recommended Mitigation

Replace the `||` operator with the `&&` operator to ensure both token amounts are greater than `zero`.

```rust
    assert_or!(
-       amount_0_abs > U256::zero() || amount_1_abs > U256::zero(),
+       amount_0_abs > U256::zero() && amount_1_abs > U256::zero(),
        Error::SwapResultTooLow
    );
```

### Discussion

### Notes & Impressions

#### Notes 
Using || (OR) here is risky because it allows the swap to go through even if one of the token amounts is zero. Imagine you’re swapping 100 units of token A to get some token B:

- **Scenario**: Due to slippage (market prices shifting), you end up with:
    - amount_0_abs = 100 (token A amount is positive),
    - amount_1_abs = 0 (token B amount is zero).
- With the || operator, the condition amount_0_abs > 0 || amount_1_abs > 0 is still true because amount_0_abs > 0 is true. The swap is allowed to proceed.
- **Result**: You give away 100 token A but get _nothing_ (0 token B) in return. That’s a terrible deal!

#### Impressions

When checking multiple conditions that all need to be true for an operation to be safe or correct (especially in financial transactions like swaps), use && (AND) instead of || (OR).

### Tools
### Refine

{{ Refine to typical issues}}

---

## Audit Summary Notes
- {{summary_notes}}

## Tags
- Category: {{tags_category}}
- Priority:{{tags_priority}}