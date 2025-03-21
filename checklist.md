# Checklist
## Common list
1. Function signatures must match exactly between Solidity and Rust.
	- Same parameter count
	- Same parameter types
	- Same parameter order
	- [[2024-10-superposition#[H-01] `createPoolD650E2D0` will not work due to mismatch in solidity and stylus function definitions|[H-01] `createPoolD650E2D0` will not work due to mismatch in solidity and stylus function definitions]]]
2. Withdrawals Particularly Need Slippage Protection
	- [[2024-10-superposition#[H-01] `createPoolD650E2D0` will not work due to mismatch in solidity and stylus function definitions]]
3. Permission or state checks should be consistently applied across ***all_code_paths*** that perform the same or similar sensitive operations.
	- Simply compare similar functions have the similar permission or state checks
	- [[2024-10-superposition#[M-02] Tokens are pulled from users without verifying pool status contrary to requirement|[M-02] Tokens are pulled from users without verifying pool status contrary to requirement]]
4. When checking multiple conditions that all need to be true for an operation to be safe or correct (especially in financial transactions like swaps), use && (AND) instead of || (OR).
	1. [[2024-10-superposition#[M-03] Incorrect slippage handing in `swap_internal()`|[M-03] Incorrect slippage handing in `swap_internal()`]]
5. Approval Reset or revoke
	1. [[2024-08-superposition#[H-02] Unrevoked approvals allow NFT recovery by previous owner|[H-02] Unrevoked approvals allow NFT recovery by previous owner]]
6. `lower tick < upper tick` in Uniswap-style liquidity pools
	1. [[2024-08-superposition#[H-03] Missing `lower<upper` check in `mint_position`|[H-03] Missing `lower<upper` check in `mint_position`]]
7. Programming Error
	1. incorrect order of arguments
		1. [[2024-08-superposition#[H-05] Parameter Misordering in Fee Collection Function Causes Denial of Service and Fee Loss]]
	2. copy & paste
		1. [[2024-08-superposition#[H-01] `update_emergency_council_7_D_0_C_1_C_58()` updates nft manager instead of emergency council]]

## Uniswap
1. `fee_growth_inside` can be negative.
	1. [[2024-08-superposition#[H-04] Position's owed fees should allow underflow but it reverts instead, resulting in locked funds|[H-04] Position's owed fees should allow underflow but it reverts instead, resulting in locked funds]]
2. `lower tick < upper tick`
	1. [[2024-08-superposition#[H-03] Missing `lower<upper` check in `mint_position`|[H-03] Missing `lower<upper` check in `mint_position`]]
