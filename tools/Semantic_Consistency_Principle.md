# The Semantic Consistency Principle Between Rust and Solidity in Smart Contract Audits

The Semantic Consistency Principle is particularly critical when auditing smart contracts that involve multiple programming languages like Rust and Solidity. Let me elaborate on this principle and its implications for cross-language smart contract systems.

## Core Concept

At its heart, the Semantic Consistency Principle requires that the meaning of data remains consistent as it flows through different parts of a system, especially across language boundaries. This goes beyond syntax and focuses on semantics—what the data actually represents conceptually.

## Common Error
- When handling data across different components or languages of a system, ensure that the semantic meaning of variables and their relationships are preserved throughout the entire execution flow.
	- [[2024-08-superposition#[H-02] Unrevoked approvals allow NFT recovery by previous owner]]
	- [[2024-08-superposition#[H-06] `get_fee_growth_inside` in `tick.rs` should allow for `underflow`/`overflow` but doesn't]]
- Function signatures must match exactly between Solidity and Rust.
	- Same parameter count
	- Same parameter types
	- Same parameter order
	- [[2024-10-superposition#[H-01] `createPoolD650E2D0` will not work due to mismatch in solidity and stylus function definitions|[H-01] `createPoolD650E2D0` will not work due to mismatch in solidity and stylus function definitions]]]
