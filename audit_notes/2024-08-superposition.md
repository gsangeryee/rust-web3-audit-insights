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

# Medium Risk Findings (xx)

---

{{Copy from Medium Risk Finding Template.md}}

---

## Audit Summary Notes
- {{summary_notes}}

## Tags
- Category: {{tags_category}}
- Priority:{{tags_priority}}