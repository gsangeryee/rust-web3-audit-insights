# 2024-08-superposition
---
- Category: chose from [[protocol_categories]]
- Note Create 2025-01-22
- Platform: code4rena
- Report Url: [2024-08-superposition Reports](https://code4rena.com/reports/2024-08-superposition)
- Competitive Url:[2024-08-superposition](https://code4rena.com/audits/2024-08-superposition)
---
# High Risk Findings (7)

---
## [H-01] `update_emergency_council_7_D_0_C_1_C_58()` updates nft manager instead of emergency council
----
- **Tags**:  #programming_error
- Number of finders: 15
- Difficulty: Easy
---
### Impact

Inside of `lib.rs`, there is a function `update_emergency_council_7_D_0_C_1_C_58()` that is needed to update the emergency council that can disable the pools. However, in the current implementation, `nft_manager` is updated instead.
### Proof of Concept

This is the current functionality of `update_emergency_council_7_D_0_C_1_C_58()`:

[lib.rs#L1111-1124](https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/lib.rs#L1111-1124)
```rust
pub fn update_emergency_council_7_D_0_C_1_C_58(
        &mut self,
        manager: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(
            msg::sender(),
            self.seawater_admin.get(),
            Error::SeawaterAdminOnly
        );

        self.nft_manager.set(manager);  // Bug is here!

        Ok(())
    }
```

As you can see, the function updates `nft_manager` contract instead of `emergency_council` that is needed to be updated. Above this function there is another function that updates `nft_manager`:

[lib.rs#L1097-1107](https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/lib.rs#L1097-1107)
```rust
  pub fn update_nft_manager_9_B_D_F_41_F_6(&mut self, manager: Address) -> Result<(), Revert> {
        assert_eq_or!(
            msg::sender(),
            self.seawater_admin.get(),
            Error::SeawaterAdminOnly
        );

        self.nft_manager.set(manager);

        Ok(())
    }
```

As you can see here, in both of the functions `nft_manager` is updated which is an unexpected behavior and the contract cannot update the `emergency_council` that handles emergency situations:

[lib.rs#L117-118](https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/lib.rs#L117-118)
```rust
 // address that's able to activate and disable emergency mode functionality
    emergency_council: StorageAddress,
```
### Recommended Mitigation

Change `update_emergency_council_7_D_0_C_1_C_58()` to update `emergency_council`
### Discussion

**0xsomeone (judge) commented:**

> The Warden and its duplicates have correctly identified that the mechanism exposed for updating the `emergency_council` will incorrectly update the `nft_manager` instead.
> 
> I initially wished to retain a medium risk severity rating for this vulnerability due to how the `emergency_council` is configured during the contract’s initialization and its value changing being considered a rare event; however, a different highly sensitive variable is altered instead incorrectly (`nft_manager`) which would have significant consequences to the system temporarily.
> 
> Based on the above, I believe that a high-risk rating is appropriate due to the unexpected effects invocation of the function would result in.

### Notes

#### Notes 
The contract has two distinct administrative components:

- `emergency_council`: An address with authority to activate/disable emergency mode
- `nft_manager`: An address that presumably manages NFT-related functionality
#### Impressions

Looks like programming error

### Tools

- [[Semantic_Consistency_Principle]]
### Refine

{{ Refine to typical issues}}

---
## [H-02] Unrevoked approvals allow NFT recovery by previous owner
----
- **Tags**:  #token_transfer #approval_reset_or_revoke #semantic_consistency_principle 
- Number of finders: 10
- Difficulty: Easy
---
### Impact

The vulnerability arises from the fact that after a token transfer, the approval status for the token is not revoked. Specifically, the `getApproved[_tokenId]` is not updated on transfer.  
This allows the previous owner (or any approved address) to reclaim the NFT by using the approval mechanism to re-transfer the token back to themselves. This is critical because the new owner of the NFT may lose their asset without realizing it, leading to potential exploitation, loss of assets, and decreased trust in the platform.
### Proof of Concept

In the provided `approve` function, any user can approve themselves or another address for a specific token ID:

```rust
/// @inheritdoc IERC721Metadata
function approve(address _approved, uint256 _tokenId) external payable {
    _requireAuthorised(msg.sender, _tokenId);
    getApproved[_tokenId] = _approved;
}
```

Since the approval is not revoked upon transfer, the previous owner retains the ability to re-transfer the NFT. The `_requireAuthorised` function is the only check on transfer permission:

```rust
function _requireAuthorised(address _from, uint256 _tokenId) internal view {
    // revert if the sender is not authorised or the owner
    bool isAllowed =
        msg.sender == _from ||
        isApprovedForAll[_from][msg.sender] ||
        msg.sender == getApproved[_tokenId];

    require(isAllowed, "not allowed");
    require(ownerOf(_tokenId) == _from, "_from is not the owner!");
}
```

### Step-by-Step PoC:

1. **Initial Approval**: The owner of a token (`owner1`) approves an address (`addr2`) to transfer their token.
2. **Token Transfer**: The token is transferred from `owner1` to `newOwner`.
3. **Approval Not Revoked**: The approval for `addr2` is not revoked after the transfer.
4. **NFT Recovery**: `addr2` can still use their approval to transfer the NFT back to themselves, effectively recovering the NFT from `newOwner`.

Note to the judge: there is no existing tests for this specific smart contract (because it is in solidity). A coded POC for this easy-to-understand vulnerability would involve to create all deployment logic.
### Recommended Mitigation

To prevent this vulnerability, any existing approvals should be revoked when a token is transferred. This can be achieved by adding a line in the transfer function to clear the approval:

```rust
getApproved[_tokenId] = address(0);
```

This line should be added to the token transfer function to ensure that any previously approved addresses cannot transfer the NFT after it has been sold or transferred to a new owner.

### Discussion

### Notes

#### Notes 
In ERC-721 tokens (the standard for NFTs), there are two permission mechanisms:

1. **Direct ownership** - The owner of an NFT has full control
2. **Approval system** - The owner can grant permission to specific addresses to transfer a specific token

The vulnerability occurs because when a token is transferred to a new owner, the contract fails to clear previous approvals. This creates a serious security hole:
- Alice owns Token #123
- Alice approves Bob to transfer Token #123
- Alice sells Token #123 to Charlie (or Bob transfers it to Charlie)
- **The approval for Bob remains active even after Charlie becomes the new owner**
- Bob can now use his still-active approval to take Token #123 back from Charlie without permission
#### Impressions

**Approval Reset or Revoke**

### Tools
### Refine

{{ Refine to typical issues}}

---
## [H-03] Missing `lower<upper` check in `mint_position`
----
- **Tags**: #invalid_validation #lower_upper #edge_case #uniswap 
- Number of finders: 17
- Difficulty: Easy
---
### Detail

The current implementation does not perform a check for `lower < upper` during `mint_position`, while many other functions assume `lower < upper`. This discrepancy allows malicious actors to exploit inconsistencies in handling undefined behavior in other parts of the code for their benefit.
1. **Case when `lower = upper`:**
   In the `StorageTicks::update` function, liquidity can be added without issue because only one boundary and the current tick need to be passed, meaning it should function correctly even if the current tick equals the boundary. However, when calculating the amount of tokens required from the user, due to `lower = upper`, the calculation can only fall into the first and third branches. Here, `sqrt_ratio_a_x_96 = sqrt_ratio_a_x_96`, and the difference between these two values multiplies the result, leading to a token amount of 0 regardless of liquidity. A malicious user can exploit this implementation discrepancy to open a position with arbitrary liquidity value without consuming any tokens.

```rust
/* pkg/seawater/src/pool.rs */
170 | // calculate liquidity change and the amount of each token we need
171 | if delta != 0 {
172 |     let (amount_0, amount_1) = if self.cur_tick.get().sys() < lower {
...
183 | 
184 |         // we're below the range, we need to move right, we'll need more token0
185 |         (
186 |             sqrt_price_math::get_amount_0_delta(
187 |                 tick_math::get_sqrt_ratio_at_tick(lower)?,
188 |                 tick_math::get_sqrt_ratio_at_tick(upper)?,
189 |                 delta,
190 |             )?,
191 |             I256::zero(),
192 |         )
193 |     } else if self.cur_tick.get().sys() < upper {
194 |         // we're inside the range, the liquidity is active and we need both tokens
...
224 |     } else {
...
236 |         // we're above the range, we need to move left, we'll need token1
237 |         (
238 |             I256::zero(),
239 |             sqrt_price_math::get_amount_1_delta(
240 |                 tick_math::get_sqrt_ratio_at_tick(lower)?,
241 |                 tick_math::get_sqrt_ratio_at_tick(upper)?,
242 |                 delta,
243 |             )?,
244 |         )
245 |     };
```

While the attacker cannot directly profit by closing the position, these positions affect fee calculations. For example, in the provided PoC, the attacker can create a large number of empty positions within their liquidity range to collect significant fees from swapping users, inflating what would normally be a fee of 6 to 1605.

2. **Case when `lower > upper`:**
   Unlike the previous case, when `lower > upper`, it is possible to create a position with both liquidity and balance. However, the potential issue arises when calling the `get_fee_growth_inside` function to calculate fees. This function also assumes `lower < upper` in its calculations, and passing `lower > upper` results in incorrect fee calculations, allowing the attacker to steal fees from other liquidity providers. However, the current implementation of `get_fee_growth_inside` has a bug where it does not correctly handle overflow as per the Uniswap V3 specification, rendering this exploit temporarily unusable.
### Proof of Concept

```rust
#[test]
fn lower_upper_eq_poc() {
    test_utils::with_storage::<_, Pools, _>(
        Some(address!("3f1Eae7D46d88F08fc2F8ed27FCb2AB183EB2d0E").into_array()), // sender
        None,
        None,
        None,
        |contract| -> Result<(), Vec<u8>> {
            let token0 = address!("9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0");
            contract.ctor(msg::sender(), Address::ZERO, Address::ZERO)?;
            contract.create_pool_D650_E2_D0(
                token0,
                U256::from_limbs([0, 42942782960, 0, 0]), 
                500,                                      // fee
                10,                                       // tick spacing
                u128::MAX,
            )?;
            contract.enable_pool_579_D_A658(token0, true)?;
            contract.mint_position_B_C5_B086_D(token0, 30000, 50000)?; // Init a normal position
            println!("Create normal position: {:?}", contract.update_position_C_7_F_1_F_740(token0, U256::from(0), 50000).unwrap()); 

            println!("Tick before swap {}", contract.cur_tick181_C6_F_D9(token0).unwrap());
            // Create 500 malicious empty positions
            for i in 0..2000 {
                contract.mint_position_B_C5_B086_D(token0, 30000+i*10, 30000+i*10)?;
                let (amount0, amount1) = contract.update_position_C_7_F_1_F_740(token0, U256::from(1+i), 100000000000).unwrap();
                assert!(amount0 == I256::ZERO && amount1 == I256::ZERO); // Attacker don't need any token
            }
            println!("Victim swap: {:?}", contract.swap_904369_B_E(
                token0,
                true,
                I256::try_from(10000_i32).unwrap(),
                U256::MAX,
            ).unwrap());
            println!("Tick after swap {}", contract.cur_tick181_C6_F_D9(token0).unwrap());
            println!("Refresh position {:?}", contract.update_position_C_7_F_1_F_740(token0, U256::from(0), 0).unwrap());
            println!("Fee owed: {:?}", contract.fees_owed_22_F28_D_B_D(token0, U256::from(0)).unwrap());
            Ok(())
        },
    )
    .unwrap()
}
```
### Recommended Mitigation

Add `assert_or!(lower < upper, Error::InvalidTick)` check.
### Discussion
**0xsomeone (judge) commented:**

> The Warden has properly identified that the creation of a position does not sufficiently validate its lower and upper tick values, permitting positions whereby the ticks are inverted or equal to be created with significant consequences.
> 
> I consider a high-risk severity rating appropriate and have penalized submissions that simply identified the vulnerability (i.e., as a medium, or simply noted it with no impact justification) by 25% (i.e., rewarded with a partial reward of 75%).
### Notes

#### Notes 
In Uniswap-style liquidity pools, liquidity is provided within a price range defined by two tick values:

- **Lower tick**: The lower price boundary
- **Upper tick**: The upper price boundary

Normally, we expect: `lower tick < upper tick`

Now, an attacker named Bob discovers the vulnerability:

1. Bob creates a position with `lower tick = upper tick = 30000` (both representing the same price point)
2. When the contract calculates how many tokens Bob needs to provide, it does:
    
    Copy
    
    `amount = sqrt_ratio_at(upper) - sqrt_ratio_at(lower)`
    
3. Since `upper = lower`, this calculation becomes:
    
    Copy
    
    `amount = sqrt_ratio_at(30000) - sqrt_ratio_at(30000) = 0`
    
4. Bob can now set his liquidity to any large value (like 100,000,000,000) without depositing any tokens!
### Tools
### Refine

{{ Refine to typical issues}}

---
## [H-04] Position's owed fees should allow underflow but it reverts instead, resulting in locked funds
----
- **Tags**: #uniswap
- Number of finders: 3
- Difficulty: Medium
---
### Detail

The math used to calculate how many fees are owed to the user does not allow underflows, but this is a necessary due to how the Uniswap logic works as fees can be negative. This impact adding/removing funds to a position, resulting in permanently locked funds due to a revert.
### Context

Similar to `Position's fee growth can revert resulting in funds permanently locked` but with a different function/root cause, both issues must be fixed separately.
### Proof of Concept

In `position.update` underflows are not allowed:

```rust
    pub fn update(
        &mut self,
        id: U256,
        delta: i128,
        fee_growth_inside_0: U256,
        fee_growth_inside_1: U256,
    ) -> Result<(), Error> {
        let mut info = self.positions.setter(id);
        let owed_fees_0 = full_math::mul_div(
            fee_growth_inside_0
->              .checked_sub(info.fee_growth_inside_0.get())
                .ok_or(Error::FeeGrowthSubPos)?,
            U256::from(info.liquidity.get()),
            full_math::Q128,
        )?;
        let owed_fees_1 = full_math::mul_div(
            fee_growth_inside_1
->              .checked_sub(info.fee_growth_inside_1.get())
                .ok_or(Error::FeeGrowthSubPos)?,
            U256::from(info.liquidity.get()),
            full_math::Q128,
        )?;
        let liquidity_next = liquidity_math::add_delta(info.liquidity.get().sys(), delta)?;

        if delta != 0 { 
            info.liquidity.set(U128::lib(&liquidity_next));
        }
        info.fee_growth_inside_0.set(fee_growth_inside_0);
        info.fee_growth_inside_1.set(fee_growth_inside_1);
        if !owed_fees_0.is_zero() {
            // overflow is the user's problem, they should withdraw earlier
            let new_fees_0 = info
                .token_owed_0
                .get()
                .wrapping_add(U128::wrapping_from(owed_fees_0));
            info.token_owed_0.set(new_fees_0);
        }
        if !owed_fees_1.is_zero() {
            let new_fees_1 = info
                .token_owed_1
                .get()
                .wrapping_add(U128::wrapping_from(owed_fees_1));
            info.token_owed_1.set(new_fees_1);
        }
        Ok(())
    }
```

While the original Uniswap version allows underflows:

```rust
    uint128 tokensOwed0 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0X128 - _self.feeGrowthInside0LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );
    uint128 tokensOwed1 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1X128 - _self.feeGrowthInside1LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );
```

The issue is that negative fees are expected due to how the formula works. It is explained in detail in this Uniswap's issue:[Uniswap/v3-core#573](https://github.com/Uniswap/v3-core/issues/573)

This function is used every time a position is updated, so it will be impossible to remove funds from it when the underflow happens because it will revert, resulting in locked funds.
### Recommended Mitigation

Consider using `-` to let Rust underflow without errors in release mode:

```rust
        let owed_fees_0 = full_math::mul_div(
            fee_growth_inside_0
-               .checked_sub(info.fee_growth_inside_0.get())
-               .ok_or(Error::FeeGrowthSubPos)?,
+				- info.fee_growth_inside_0.get()
            U256::from(info.liquidity.get()),
            full_math::Q128,
        )?;

        let owed_fees_1 = full_math::mul_div(
            fee_growth_inside_1
-              .checked_sub(info.fee_growth_inside_1.get())
-              .ok_or(Error::FeeGrowthSubPos)?,
+			   - info.fee_growth_inside_1.get()
            U256::from(info.liquidity.get()),
            full_math::Q128,
        )?;
```

Alternatively, consider using `wrapping_sub`.
### Discussion

### Notes

#### Notes 
1. In Uniswap, `fee_growth_inside` can be negative.
2. `fn checked_sub(&self, v: &Self) -> Option<Self>` Subtracts two numbers, checking for underflow. If underflow happens, `None` is returned.
### Tools
### Refine

{{ Refine to typical issues}}

## [H-05] Parameter Misordering in Fee Collection Function Causes Denial of Service and Fee Loss
----
- **Tags**:  #DOS #programming_error 
- Number of finders: 7
- Difficulty: Easy
---
### Impact

A critical bug has been identified in the `collect_protocol_7540_F_A_9_F` function of the Seawater Automated Market Maker (AMM) protocol. This vulnerability affects the core functionality of fee collection, rendering the system unable to transfer protocol fees from liquidity pools to the Seawater Admin or any designated recipient. The bug stems from incorrect parameter handling during the invocation of the `transfer_to_addr` function, which results in a complete failure of the fee withdrawal process.

The impact of this issue is severe:

1. **Denial of Service (DoS)**: The system's inability to transfer protocol fees halts the entire fee collection process, effectively disabling this core feature. Without this function, the Seawater Admin cannot retrieve fees collected from liquidity pools, meaning the protocol cannot generate or distribute earnings from the token swapping activity in these pools.
    
2. **Financial Loss**: Since the protocol relies on fees for its economic model, this bug causes a direct financial loss. The protocol's earnings from fees generated by liquidity pools are locked and inaccessible, leading to loss of funds. Over time, the fees accumulate in the pools, but they cannot be withdrawn by the Seawater Admin or any authorized recipient.
    
3. **Erosion of User Trust**: Protocol participants expect proper fee collection and distribution. If this feature fails, users may lose confidence in the protocol's reliability, leading to reduced engagement or participation. For liquidity providers and investors, this issue directly affects expected returns, as fees cannot be withdrawn.
    
4. **Long-term Protocol Viability**: If not addressed, this bug could harm the protocol’s long-term viability. Since the protocol is unable to collect revenue from its own pools, its ability to cover operational costs or distribute rewards is compromised. This could lead to a shutdown or significant degradation of the protocol’s functionality.
    

In summary, the failure to collect protocol fees undermines the core value proposition of the AMM protocol, resulting in a loss of funds, DoS of essential functionality, and the potential collapse of the protocol’s economic model if left unresolved.
### Proof of Concept

The bug exists in the fee collection logic of the `collect_protocol_7540_F_A_9_F` function located in `lib.rs`. This function is responsible for transferring collected protocol fees (in the form of tokens) from an AMM pool to the Seawater Admin or another recipient. However, incorrect parameter handling within the `transfer_to_addr` function causes the entire process to fail.

Here’s a detailed breakdown of the problematic code:

```rust
pub fn collect_protocol_7540_F_A_9_F(
    &mut self,
    pool: Address,
    amount_0: u128,
    amount_1: u128,
    recipient: Address,
) -> Result<(u128, u128), Revert> {
    assert_eq_or!(
        msg::sender(),
        self.seawater_admin.get(),
        Error::SeawaterAdminOnly
    );
    let (token_0, token_1) = self
        .pools
        .setter(pool)
        .collect_protocol(amount_0, amount_1)?;
    // Incorrect transfer logic
    erc20::transfer_to_addr(recipient, pool, U256::from(token_0))?;
    erc20::transfer_to_addr(recipient, FUSDC_ADDR, U256::from(token_1))?;
    #[cfg(feature = "log-events")]
    evm::log(events::CollectProtocolFees {
        pool,
        to: recipient,
        amount0: token_0,
        amount1: token_1,
    });
    // transfer tokens
    Ok((token_0, token_1))
}
```

#### **Problematic Code:**

The issue arises in the following lines:

```rust
erc20::transfer_to_addr(recipient, pool, U256::from(token_0))?;
erc20::transfer_to_addr(recipient, FUSDC_ADDR, U256::from(token_1))?;
```

In this block:

- **`recipient`** is incorrectly passed as the token address.
- **`pool` or `FUSDC_ADDR`** is incorrectly passed as the recipient.

This order causes the `transfer_to_addr` function to fail because it expects the **first parameter** to be the token address and the **second parameter** to be the recipient address. Here's the correct signature of the `transfer_to_addr` function from `wasm_erc20.rs`:

```rust
pub fn transfer_to_addr(token: Address, recipient: Address, amount: U256) -> Result<(), Error> {
    safe_transfer(token, recipient, amount)
}
```

Since the parameters are passed in the wrong order, the `transfer_to_addr` function fails, leading to a complete halt of the fee transfer process. As a result, the collected fees cannot be moved from the pools to the Seawater Admin or recipient, effectively blocking the protocol’s ability to distribute or utilize these funds.

#### **Steps to Reproduce the Issue**:

1. Call the `collect_protocol_7540_F_A_9_F` function with a valid pool address, token amounts, and recipient address.
2. The function will attempt to transfer the protocol fees using `erc20::transfer_to_addr`.
3. Since the arguments are incorrectly passed (with the recipient and token address swapped), the transaction will fail, and no tokens will be transferred.

#### **Failed Transaction Log**:

```
[18881] collect_protocol_7540_F_A_9_F::collect_fees()
    ├─ [11555] erc20::transfer_to_addr(token: [recipient], pool: [token], 1)
    │   └─ ← [Revert] 
    └─ ← [Revert] 
```

This log highlights the issue: The `transfer_to_addr` function is attempting to use the recipient address as the token address, and the pool address as the recipient. This mismatch causes the transfer to fail, preventing the collected protocol fees from being withdrawn.

### Recommended Mitigation

To fix this issue, the arguments passed to the `erc20::transfer_to_addr` function need to be correctly ordered, ensuring that the token address is provided first, followed by the recipient address. This correction will allow the transfer of protocol fees from the liquidity pools to the designated recipient.

#### **Corrected Code**:

```rust
erc20::transfer_to_addr(pool, recipient, U256::from(token_0))?;
erc20::transfer_to_addr(FUSDC_ADDR, recipient, U256::from(token_1))?;
```

This update reverses the order of parameters, passing the **token address as the first parameter** and the **recipient address as the second parameter**, as expected by the `transfer_to_addr` function. By implementing this fix, the protocol will correctly transfer collected fees, ensuring that the Seawater Admin or any authorized recipient can successfully withdraw fees from the AMM pools.

#### **Additional Steps to Consider**:

1. **Add Unit Tests**: To prevent this issue from recurring, add specific unit tests to validate the correct behavior of the `collect_protocol_7540_F_A_9_F` function. These tests should confirm that fees can be collected and transferred without errors.
    
2. **Error Handling Improvements**: Consider adding more explicit error messages to the `transfer_to_addr` function to help identify issues during fee transfers. This could include checking whether the token address and recipient address are valid before proceeding with the transfer.
    
3. **Gas Efficiency**: After resolving this issue, it’s advisable to audit the overall gas usage of the fee collection process to ensure that no unnecessary operations are being performed during the transfer of protocol fees.
### Discussion

**0xsomeone (judge) commented:**

> The Warden has identified that an incorrect order of arguments in the EIP-20 transfer calls within the `lib::collect_protocol_7540_F_A_9_F` function will cause a token transfer to be attempted with the `recipient` address as the “token” thereby failing on each invocation.
> 
> I believe a high severity rating is appropriate given that funds are directly lost and are irrecoverable due to an improper implementation of the protocol fee claim mechanism.

### Notes

#programming_error  - incorrect order of arguments

### Tools
### Refine

{{ Refine to typical issues}}

---
## [H-06] `get_fee_growth_inside` in `tick.rs` should allow for `underflow`/`overflow` but doesn't
----
- **Tags**:  #under_overflow
- Number of finders: 8
- Difficulty: Easy
---
### Impact

When operations need to calculate the `Superposition` position's fee growth, it uses a similar function implemented by [uniswap v3](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/PositionValue.sol#L145-L166).

However, according to this known issue : [Uniswap/v3-core#573](https://github.com/Uniswap/v3-core/issues/573). The contract implicitly relies on underflow/overflow when calculating the fee growth, if underflow is prevented, some operations that depend on fee growth will revert.
### Proof of Concept

It can be observed that the current implementation of `tick::get_fee_growth_inside` does not allow underflow/overflow to happen when calculating `fee_growth_inside_0` and `fee_growth_inside_1` because the contract used [checked math](https://docs.rs/num/latest/num/trait.CheckedSub.html) with additional overflow/underflow checks, which will `Performs subtraction that returns None instead of wrapping around on underflow`:

`tick.rs`
```rust
pub fn get_fee_growth_inside(
    &mut self,
    lower_tick: i32,
    upper_tick: i32,
    cur_tick: i32,
    fee_growth_global_0: &U256,
    fee_growth_global_1: &U256,
) -> Result<(U256, U256), Error> {
    // the fee growth inside this tick is the total fee
    // growth, minus the fee growth outside this tick
    let lower = self.ticks.get(lower_tick);
    let upper = self.ticks.get(upper_tick);

    let (fee_growth_below_0, fee_growth_below_1) = if cur_tick >= lower_tick {
        #[cfg(feature = "testing-dbg")]
        dbg!((
            "cur_tick >= lower_tick",
            current_test!(),
            lower.fee_growth_outside_0.get().to_string(),
            lower.fee_growth_outside_1.get().to_string()
        ));
        (
            lower.fee_growth_outside_0.get(),
            lower.fee_growth_outside_1.get(),
        )
    } else {
        #[cfg(feature = "testing-dbg")]
        dbg!((
            "cur_tick < lower_tick",
            current_test!(),
            fee_growth_global_0,
            fee_growth_global_1,
            lower.fee_growth_outside_0.get().to_string(),
            lower.fee_growth_outside_1.get().to_string()
        ));
        (
            fee_growth_global_0
                .checked_sub(lower.fee_growth_outside_0.get())
                .ok_or(Error::FeeGrowthSubTick)?,
            fee_growth_global_1
                .checked_sub(lower.fee_growth_outside_1.get())
                .ok_or(Error::FeeGrowthSubTick)?,
        )
    };

    let (fee_growth_above_0, fee_growth_above_1) = if cur_tick < upper_tick {
        #[cfg(feature = "testing-dbg")]
        dbg!((
            "cur_tick < upper_tick",
            current_test!(),
            upper.fee_growth_outside_0.get().to_string(),
            upper.fee_growth_outside_1.get().to_string()
        ));
        (
            upper.fee_growth_outside_0.get(),
            upper.fee_growth_outside_1.get(),
      )
    } else {
        #[cfg(feature = "testing-dbg")]
        dbg!((
            "cur_tick >= upper_tick",
            current_test!(),
            fee_growth_global_0,
            fee_growth_global_1,
            upper.fee_growth_outside_0.get(),
            upper.fee_growth_outside_1.get()
        ));
        (
            fee_growth_global_0
                .checked_sub(upper.fee_growth_outside_0.get())
                .ok_or(Error::FeeGrowthSubTick)?,
            fee_growth_global_1
                .checked_sub(upper.fee_growth_outside_1.get())
                .ok_or(Error::FeeGrowthSubTick)?,
        )
    };
    #[cfg(feature = "testing-dbg")] // REMOVEME
    {
        if *fee_growth_global_0 < fee_growth_below_0 {
            dbg!((
                "fee_growth_global_0 < fee_growth_below_0",
                current_test!(),
                fee_growth_global_0.to_string(),
                fee_growth_below_0.to_string()
            ));
        }
        let fee_growth_global_0 = fee_growth_global_0.checked_sub(fee_growth_below_0).unwrap();
        if fee_growth_global_0 < fee_growth_above_0 {
            dbg!((
                "fee_growth_global_0 < fee_growth_above_0",
                current_test!(),
                fee_growth_global_0.to_string(),
                fee_growth_above_0.to_string()
            ));
        }
    }
    #[cfg(feature = "testing-dbg")]
    dbg!((
        "final stage checked sub below",
        current_test!(),
        fee_growth_global_0
            .checked_sub(fee_growth_below_0)
            .and_then(|x| x.checked_sub(fee_growth_above_0))
    ));
    Ok((
        fee_growth_global_0
            .checked_sub(fee_growth_below_0)
            .and_then(|x| x.checked_sub(fee_growth_above_0))
            .ok_or(Error::FeeGrowthSubTick)?,
        fee_growth_global_1
            .checked_sub(fee_growth_below_1)
            .and_then(|x| x.checked_sub(fee_growth_above_1))
            .ok_or(Error::FeeGrowthSubTick)?,
    ))
}
```

Especially the last lines where we have:

`fee_growth_global_0 - fee_growth_below_0 - fee_growth_above_0`

and

`fee_growth_global_1 - fee_growth_below_1 - fee_growth_above_1`

In case either one of the calculation underflows we will receive `Error::FeeGrowthSubTick` instead of being wrapped and continue working.

Uniswap allows such underflows to happen because [`PositionLibrary`](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/PositionValue.sol#L145-L166) works with Solidity version < 0.8. Otherwise, this will impact crucial operations that rely on this call, such as liquidation, and will revert unexpectedly. This behavior is quite often, especially for pools that use lower fees.

> **NOTE:** _Furthermore, one of the Main Invariants in the **README** mentioned that - Superposition should follow the UniswapV3 math `faithfully`, which is not the case here, violating the invariant._  
> [https://github.com/code-423n4/2024-08-superposition?tab=readme-ov-file#main-invariants](https://github.com/code-423n4/2024-08-superposition?tab=readme-ov-file#main-invariants)
### Recommended Mitigation

Use `unsafe`/`unchecked` math when calculating the fee growths.
### Discussion

### Notes

#### Notes 
like [[2024-08-superposition#[H-04] Position's owed fees should allow underflow but it reverts instead, resulting in locked funds|[H-04] Position's owed fees should allow underflow but it reverts instead, resulting in locked funds]]
### Tools
### Refine

{{ Refine to typical issues}}

---
## [H-07]  `swapOut` functions have invalid slippage check, causing user loss of funds
----
- **Tags**: #error #programming_error #uniswap #semantic_consistency_principle
- Number of finders: 2
- Difficulty: Hard
---
### Impact

`swapOut` functions have an invalid slippage check, causing user loss of funds.
### Proof of Concept

In `SeawaterAMM.sol`, `swapOut5E08A399` and `swapOutPermit23273373B` are intended to allow `usdc(token1) -> pool(token0)` swap with slippage check.

However, both functions have incorrect slippage checks.  
(1) We see in `swapOut5E08A399` `swapAmountOut` is used to check with `minOut`. But `swapAmountOut` is actually `usdc`(token1), Not the output token(token0). This uses an incorrect variable to check slippage.

```rust
    function swapOut5E08A399(
        address token,
        uint256 amountIn, //@audit-info note: this is usdc(token1)
        uint256 minOut
    ) external returns (int256, int256) {
            (bool success, bytes memory data) = _getExecutorSwap().delegatecall(
            abi.encodeCall(
                ISeawaterExecutorSwap.swap904369BE,
                (token, false, int256(amountIn), type(uint256).max)
            )
        );
        require(success, string(data));
        //@audit-info note: swapAmountIn <=> token0 , swapAmountOut <=> token1
|>      (int256 swapAmountIn, int256 swapAmountOut) = abi.decode(
            data,
            (int256, int256)
        );
       //@audit This should use token0 value, not token1
|>     require(swapAmountOut >= int256(minOut), "min out not reached!");
       return (swapAmountIn, swapAmountOut);
    }
```

For reference, in the swap facet, `swap_internal` called in the flow returns `Ok((amount_0, amount_1))`. This means `swapAmountOut` refers to token1, the input token amount.

```rust
//pkg/seawater/src/lib.rs
    pub fn swap_internal(
        pools: &mut Pools,
        pool: Address,
        zero_for_one: bool,
        amount: I256,
        price_limit_x96: U256,
        permit2: Option<Permit2Args>,
    ) -> Result<(I256, I256), Revert> {
        let (amount_0, amount_1, _ending_tick) =
            pools
                .pools
                .setter(pool)
                .swap(zero_for_one, amount, price_limit_x96)?;
...
        Ok((amount_0, amount_1))
```

(2) `swapOutPermit23273373B` has the same erroneous slippage check.

```rust
//pkg/sol/SeawaterAMM.sol
    function swapOutPermit23273373B(address token, uint256 amountIn, uint256 minOut, uint256 nonce, uint256 deadline, uint256 maxAmount, bytes memory sig) external returns (int256, int256) {
...
|>     (int256 swapAmountIn, int256 swapAmountOut) = abi.decode(data, (int256, int256));
|>      require(swapAmountOut >= int256(minOut), "min out not reached!");
        return (swapAmountIn, swapAmountOut);
    }
```

Invalid slippage checks will cause users to lose funds during swaps.
### Recommended Mitigation

Consider changing into:

```rust
...
        (int256 swapAmountOut, int256 swapAmountIn) = abi.decode(
            data,
            (int256, int256)
        );
                require(uint256(-swapAmountOut) >= minOut, "min out not reached!");
                return (swapAmountOut, swapAmountIn);
```

### Discussion

### Notes

#### Notes 
The code is checking if `swapAmountOut` is greater than or equal to `minOut`. However, `swapAmountOut` actually represents the amount of token1 (USDC) that was input to the swap, not the amount of token0 (pool tokens) that was received.

The function returns a tuple of `(amount_0, amount_1)`, where:

- `amount_0` is the change in token0 (the pool token)
- `amount_1` is the change in token1 (USDC)

When these values are decoded in the Solidity contract, they become:

- `swapAmountIn` corresponds to `amount_0` (change in pool tokens)
- `swapAmountOut` corresponds to `amount_1` (change in USDC)

**Semantic Consistency Principle**: When handling data across different components or languages of a system, ensure that the semantic meaning of variables and their relationships are preserved throughout the entire execution flow.
### Tools
- [[Semantic_Consistency_Principle]]
### Refine

{{ Refine to typical issues}}

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