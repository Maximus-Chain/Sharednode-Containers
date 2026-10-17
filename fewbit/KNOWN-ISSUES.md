# FewBit Core: known upstream issues

Found while testing FewBit Core 6.6.6.6 (`GENESIS` tag, commit `8947acb`) on
2026-10-17. None of them are patched in our image; they are listed here so they
can be reported to the FewBit maintainers.

## 1. `protx list` / `info` / `diff` crash the daemon on `--disable-wallet` builds

**Affects our image** (built with `--disable-wallet`). Present since
`9b72fcb` (2024-05-15).

In `src/rpc/rpcevo.cpp`, the `protx` dispatcher reads:

```cpp
    std::string command;
    if (!request.params[0].isNull()) {
        command = request.params[0].get_str();
    }

#ifdef ENABLE_WALLET
    if (command == "register" ...) {
        ...
    }
#endif
    else if (command == "list") {
```

Without the wallet, the `#ifdef` block disappears and `else if (command ==
"list")` binds to the `if (!request.params[0].isNull())` above. Any `protx`
call with a subcommand then skips every branch and falls off the end of a
non-void function (undefined behaviour): `fewbit-cli protx list` segfaults the
node (exit 139).

Today nothing calls `protx` inside our containers (Pantheon runs only
`smartnode status` / `getnetworkinfo` there), so it only triggers on manual
use.

**Fix:** turn `else if (command == "list")` into `if (command == "list")`.
Every earlier branch returns, so this is correct with and without the wallet.
Maximus, Osmium and Filopow already use `} else` before `#endif`; Kerrigan has
no dispatcher.

## 2. Regtest genesis does not verify

In `CRegTestParams` (`src/chainparams.cpp`),
`CreateGenesisBlock(1614369600, 2, 0x207fffff, 4, 50 * COIN)` uses nonce `2`,
but the first valid PoW nonce is `5` (block hash
`0xd0b516480cd55f3fee66686a381c0a82102972d3044084aae02e3946086ddd89`).
`VerifyGenesisPOW` aborts on startup, and the `hashGenesisBlock` /
`hashMerkleRoot` asserts below it are also wrong. `-regtest` cannot start.

## 3. Founder payment ignores the per-network address

`GetActiveFounderAddress()` (`src/founder_payment.cpp`) always returns the
mainnet addresses (`FJ4QNkE7…` / `FBA69LP6…`) and ignores the address passed
to `FounderPayment` in each network's chainparams. On regtest (and likely
testnet) the address does not decode, so `generatetoaddress` fails with
`Invalid FewBit Founder Address`.

## Testing on regtest

Issues 1–3 must be patched in a test-only image to run the regtest e2e
(multisig send + ProRegTx with an IPv6 service): build `--target builder`,
`sed` the three spots, then run `make` incrementally on top.
