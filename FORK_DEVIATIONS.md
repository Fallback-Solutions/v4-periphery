# Fork deviations

This fork carries deliberate divergences from `Uniswap/v4-periphery`. Each one has to be
re-applied by hand after every upstream sync, because a merge will happily restore upstream's
version without conflicting.

**Read this file when syncing.** Nothing in CI here catches the first deviation being lost.

## 1. Single-swap parameters stay at the deployed encoding

### What differs

`ExactInputSingleParams` and `ExactOutputSingleParams` in `src/interfaces/IV4Router.sol` keep
five fields and do **not** carry upstream's `uint256 minHopPriceX36`:

```solidity
struct ExactInputSingleParams {
    PoolKey poolKey;
    bool zeroForOne;
    uint128 amountIn;
    uint128 amountOutMinimum;
    bytes hookData;
}
```

Consequently:

- The `V4TooLittleReceivedPerHopSingle` and `V4TooMuchRequestedPerHopSingle` errors are removed.
- The two single-swap length guards in `src/libraries/CalldataDecoder.sol` are `0x140`, where
  upstream has `0x160`.
- `src/V4Router.sol` has no per-hop price check in the two single-swap branches.

The **multi-hop** structs are untouched: they keep upstream's `uint256[] minHopPriceX36` and the
`0xe0` guards. That array is byte-identical to the `maxHopSlippage` it replaced upstream, so it
needs no intervention.

### Why

Routers already deployed on Base and Unichain speak the five-field layout. Adopting upstream's
extra word would mean callers had to know which chain they were talking to and encode
differently per chain. Holding one encoding everywhere is worth the sync cost.

### Why dropping it loses no protection

On a single hop the per-hop price bound and the amount bound are the same constraint, differing
only by a constant factor:

- exact input: `amountIn` is fixed, so bounding `amountOut / amountIn` and bounding `amountOut`
  are equivalent.
- exact output: `amountOut` is fixed, so bounding the ratio and bounding `amountIn` are
  equivalent.

A per-hop bound only carries information once there is more than one hop, and there it is kept.

### Re-applying after a sync

1. Delete `minHopPriceX36` from both single-swap structs in `src/interfaces/IV4Router.sol`.
2. Delete the two `*PerHopSingle` errors from the same file.
3. Delete the per-hop check blocks from the two single-swap branches in `src/V4Router.sol`.
4. Set both single-swap guards in `src/libraries/CalldataDecoder.sol` back to `0x140`. Leave the
   multi-hop guards at `0xe0`.
5. Update the construction sites. As of this writing that is 86 sites across 8 test files:
   `test/router/V4Router.t.sol`, `V4Router.gas.t.sol`, `V4RouterExactOutputUnfilled.t.sol`,
   `V4RouterHookFundedExactOutput.t.sol`, `Payments.t.sol`, `Payments.gas.t.sol`,
   `test/hooks/permissionedPools/PermissionedV4Router.t.sol` and
   `shared/PermissionedDeployers.sol`.
6. Regenerate gas snapshots with `FORGE_SNAPSHOT_CHECK` unset and `--isolate`, since CI runs
   `forge test --isolate` and gas differs enough between isolated and non-isolated runs to look
   like unrelated drift.

### Failure mode if this is lost

Quiet and expensive. A sync restores the sixth field, the struct gains a word, and the routers
already deployed start rejecting the calldata our bot builds. Nothing in this repository
exercises a deployed router, so CI here stays green while the bot stops working.

### One thing the guard does not do

The guards are `if lt(params.length, ...)`, a lower bound, and they do not count the leading
struct offset word, so each is loose by one word. Upstream's `0x160` was loose in exactly the
same way, so this change preserves upstream's convention rather than introducing or fixing the
looseness.

The practical consequence: a `params` blob that is too *long* passes the guard and is then
misparsed. A layout mismatch here does not fail closed, which is the substantive reason for
holding one encoding rather than relying on the decoder to reject the wrong one.

## 2. `V4Router._handleAction` is virtual

### What differs

`src/V4Router.sol:34` is:

```solidity
function _handleAction(uint256 action, bytes calldata params) internal virtual override {
```

Upstream omits `virtual`, which seals the action set at this level.

### Why

`BaseActionsRouter` declares `_handleAction` virtual, and both `_unlockCallback` and
`_executeActionsWithoutUnlock` are non-virtual, so this override is the only seam an inheriting
router has. The universal-router fork uses it to run non-v4 swap legs from inside the v4 lock,
which is what lets a flash-borrowed amount reach an external venue and return before the lock
settles.

No override exists in this repository, so the keyword alone changes no behaviour here.

### Re-applying after a sync

Add `virtual` back to that one declaration.
