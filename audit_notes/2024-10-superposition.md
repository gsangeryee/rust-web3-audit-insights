# TODO 2024-10-superposition
---
- Category: chose from [[protocol_categories]]
- Note Create 2025-01-22
- Platform: code4rena
- Report Url: [2024-10-superposition](https://code4rena.com/reports/2024-10-superposition)
---
# High Risk Findings (xx)

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

# Medium Risk Findings (xx)

---

{{Copy from Medium Risk Finding Template.md}}

---

## Audit Summary Notes
- {{summary_notes}}

## Tags
- Category: {{tags_category}}
- Priority:{{tags_priority}}