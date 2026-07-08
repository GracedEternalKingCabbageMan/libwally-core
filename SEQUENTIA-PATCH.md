# The Sequentia patch: issuance denomination byte

This document describes the single Sequentia-specific change in this fork of
[ElementsProject/libwally-core](https://github.com/ElementsProject/libwally-core).
The patch lives on branch `sequentia-issuance-denomination` as one commit
(`5bc915e3`, "tx: parse Sequentia asset-issuance denomination byte") on top of
the upstream tag `release_1.4.0`. All other branches are unpatched upstream.

## Background

Sequentia is a Bitcoin sidechain built as a fork of Blockstream Elements
(node repo: https://github.com/GracedEternalKingCabbageMan/Sequentia). The
Sequentia node extends the Elements asset-issuance structure `CAssetIssuance`
with a per-asset decimal-precision field:

```cpp
// Sequentia node, src/primitives/confidential.h
uint8_t nDenomination = 8;

SERIALIZE_METHODS(CAssetIssuance, obj) {
    READWRITE(obj.assetBlindingNonce, obj.assetEntropy,
              obj.nAmount, obj.nInflationKeys, obj.nDenomination);
}
```

Every serialized issuance therefore carries one extra byte relative to stock
Elements.

## Wire-format difference

An Elements transaction input signals an issuance by setting the flag bit
`WALLY_TX_ISSUANCE_FLAG` (`0x80000000`) in the outpoint index. For such an
input (excluding coinbase), the issuance payload following the sequence field
is:

| Field | Size | Stock Elements | Sequentia |
|---|---|---|---|
| `assetBlindingNonce` | 32 bytes | yes | yes |
| `assetEntropy` | 32 bytes | yes | yes |
| `nAmount` | confidential value (1, 9, or 33 bytes) | yes | yes |
| `nInflationKeys` | confidential value (1, 9, or 33 bytes) | yes | yes |
| `nDenomination` | 1 byte (`uint8_t`, decimal precision) | no | **yes** |

Stock libwally stops reading after `nInflationKeys`, so it is misaligned by one
byte for every issuance input in a Sequentia transaction. The practical
symptom: `wally_tx_from_bytes` returns `WALLY_EINVAL` on any Sequentia
issuance or reissuance transaction, and consumers that must parse every block
(such as a Lightning node) crash when they reach an issuance block.

## What the patch changes

Two files, four code paths, all in the Elements-enabled build
(`BUILD_ELEMENTS` / not `WALLY_ABI_NO_ELEMENTS`):

1. `include/wally_transaction.h`: adds a public field at the end of the
   Elements section of `struct wally_tx_input`:

   ```c
   uint8_t issuance_denomination;
   ```

   It holds the parsed denomination for issuance inputs and is 0 for
   non-issuance inputs.

2. `src/transaction.c`, `analyze_tx`: when an input has the issuance flag set,
   requires and skips one extra byte after the two confidential values, so
   length validation matches the Sequentia layout.

3. `src/transaction.c`, `tx_from_bytes`: reads the byte after
   `nInflationKeys` and stores it into
   `wally_tx_input.issuance_denomination` after the input is initialized.

4. `src/transaction.c`, `get_txin_issuance_size`: adds
   `sizeof(input->issuance_denomination)` (1) to the computed issuance size,
   so length and weight calculations stay correct.

5. `src/transaction.c`, `tx_to_bytes`: re-emits the byte after the inflation
   keys when serializing an input with `WALLY_TX_IS_ISSUANCE` set.

Together, parse and serialize round-trip Sequentia issuance transactions
byte-exact. Inputs without the issuance flag are handled identically to
upstream.

## Compatibility notes

- **Sequentia-only.** The patched branch assumes the Sequentia issuance
  layout. It will misparse stock Elements/Liquid issuance transactions (it
  would consume a byte that is not there). Do not use this branch against
  Liquid or vanilla Elements.
- **ABI.** The new struct field changes the size of `struct wally_tx_input`
  in Elements-enabled builds relative to stock libwally 1.4.0. Build
  consumers from this source tree (as SeqLN does via a git submodule); do not
  mix this header with a stock libwally binary.
- **Base version.** The branch is based on upstream `release_1.4.0`. To move
  to a newer upstream release, cherry-pick commit `5bc915e3` onto the new
  base; the touched code paths are stable, but re-verify the four locations
  listed above.

## Consumers

[SeqLN](https://github.com/GracedEternalKingCabbageMan/seqln) (branch
`sequentia-stable`) pins this fork at commit `5bc915e3` through its
`external/libwally-core` submodule (`.gitmodules` sets
`url = https://github.com/GracedEternalKingCabbageMan/libwally-core.git`,
`branch = sequentia-issuance-denomination`).

## Tests

The patch commit adds no unit tests. Upstream's test suite still applies to
the unmodified paths; the issuance change is exercised in practice by SeqLN
parsing Sequentia testnet blocks that contain asset issuances. Sequentia is
testnet software; no mainnet exists.
