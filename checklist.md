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
