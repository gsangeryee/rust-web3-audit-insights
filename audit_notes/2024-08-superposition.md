# TODO 2024-08-superposition
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
### Refine

{{ Refine to typical issues}}

---
## [H-02] Unrevoked approvals allow NFT recovery by previous owner
----
- **Tags**:  #token_transfer #approval_reset_or_revoke
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
- **Tags**: #invalid_validation #lower_upper #edge_case
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
#### Impressions
{{Your feelings about uncovering this finding.}}

### Tools
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