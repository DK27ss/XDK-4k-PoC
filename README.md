# XDK-4k-PoC

| Field           | Value |
|-----------------|-------|
| **Chain**       | BSC |
| **Profit**      | ~6.84 BNB (6.840316534082275362 WBNB) |
| **Rootcause** | Fee-on-transfer recycle logic — balance/reserve desync via `skim()` |

## Contracts

| Label | Address |
|-------|---------|
| XDK Token | `0x02739BE625f7A1Cb196F42dceEe630C394DD9FAA` |
| XDK_BASE (GPC) | `0xD3c304697f63B279cd314F92c19cDBE5E5b1631A` |
| TARGET_PAIR (XDK/XDK_BASE) | `0xe3cBa5C0A8efAeDce84751aF2EFDdCf071D311a9` |
| FLASH_PAIR (WBNB/XDK_BASE) | `0x12dAbFCe08eF59c24cdee6c488E05179Fb8D64D9` |
| PancakeSwap Router V2 | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| Attacker EOA | `0xb94F61855f616057a6Dc790C2269A33D1B13A0Ed` |
| Attack Contract | `0x1e7e4e41DefdE022E78AdD6f6e406a7520B63c70` |

---

## Summary

The XDK token on BSC contained a `_recycleFromBlackHoleOnSell` function that, upon detecting a sell (transfer to the LP pair), used `super._transfer` to move tokens from the burn address directly into the pair — bypassing the pair's internal accounting, this created a persistent gap between the pair's actual token balance and its recorded reserves, which the attacker drained repeatedly via `skim()` and direct swaps, netting ~6.84 BNB in a single transaction.

---

## Root Cause

The XDK token's `_transfer` function detects sells (transfers where `to == pair`) and calls an internal recycle routine.

```
function _transfer(from, to, amount):
    // ... fee deductions (1% burn, 1% to contract, 1% to contract) ...

    if to == pair AND recycleConditionsMet():
        _recycleFromBlackHoleOnSell(to, ...)

function _recycleFromBlackHoleOnSell(pair, ...):
    recycleAmount = min(thisRecycleBalance, maxCap)
    super._transfer(DEAD_ADDRESS, pair, recycleAmount)   // <-- THE BUG
    //  ↑ This increases pair's token balance
    //    but does NOT call pair.sync()
    //    so pair.reserve0 stays stale
```

`super._transfer(DEAD, pair, amount)` directly increases the pair's XDK balance without calling `sync()`, PancakeSwap's AMM pair tracks reserves internally via `reserve0`/`reserve1`, which only update on `swap()`, `mint()`, `burn()`, or `sync()`.

```
pair.balanceOf(XDK) > pair.reserve0()
```

This excess can be extracted by anyone via `pair.skim(recipient)`, which sends `balanceOf(token) - reserve` to the recipient.

<img width="1301" height="164" alt="image" src="https://github.com/user-attachments/assets/bbd353bc-45eb-4812-8364-8f56be1b0f9c" />

---

## XDK Token Mechanics

The XDK token implements several custom mechanics relevant to the exploit:

- **Fee-on-transfer (3% total):** Every transfer deducts 3 fees — ~1% burned to `0x...dEaD`, ~1% sent to the XDK contract, ~1% sent to the XDK contract (two separate fee buckets), the contract accumulates these for later auto-swap and reward distribution.
- **Auto-swap & addLiquidity:** When accumulated fees on the contract exceed a threshold, the token auto-swaps a portion to XDK_BASE and adds liquidity to the pair.
- **Reward distribution:** The contract distributes XDK_BASE rewards to LP holders proportionally.
- **Recycle from black hole:** When a sell occurs AND `block.timestamp >= lastRecycleTime + recycleColdTime`, the contract "recycles" tokens from the dead address back into the pair, this is intended to counteract deflation.
- **`recycleColdTime`:** 86400 seconds (24 hours) — minimum time between recycles.
- **`thisRecycleBalance`:** Running counter of how much has been recycled in the current cycle.
- **`thisRecycleMaxBalance`:** Maximum allowed recycle amount per cycle.
- **Max cap per recycle:** Capped at `reserve0 / 10` per individual recycle operation.

---

## Execution Flow

```
                FLASH_PAIR
                (WBNB/XDK_BASE)
                    |
                1. Flash borrow
                ~99.78M XDK_BASE
                    |
                    v
        ┌────────────────────────┐
        │   ATTACK CONTRACT      │
        │                        │
        │  2. Init (trigger      │
        │     recycle cooldown)  │
        │                        │
        │  3. Buy loop           │──── swap XDK_BASE → XDK
        │     (~10% reserve      │     via TARGET_PAIR
        │      per iteration)    │     (each buy triggers fees
        │                        │      + recycle on sell-side)
        │  4. Skim drain loop    │
        │     (up to 55 iters)   │──── transfer XDK → pair
        │     check threshold:   │     (triggers recycle →
        │     maxBal <= thisBal? │      balance > reserve)
        │     → skim excess      │     pair.skim(attacker)
        │                        │
        │  5. Swap drain loop    │──── swap remaining XDK → XDK_BASE
        │     (remaining XDK)    │     via TARGET_PAIR
        │                        │
        │  6. Repay flash loan   │
        │     + swap profit      │──── XDK_BASE → WBNB
        │       to WBNB          │
        └────────────────────────┘
```

### Step-by-step

**Phase 1 — Flash Borrow**
- Borrow 99% of XDK_BASE from `FLASH_PAIR` (~99.78M XDK_BASE).
- PancakeSwap calls `pancakeCall()` on the attack contract.

**Phase 2 — Init (Trigger Recycle Cooldown)**
- Check if `block.timestamp >= lastRecycleTime + recycleColdTime` (24h cooldown expired).
- Send 1 XDK_BASE to `TARGET_PAIR`, swap for a small amount of XDK.
- Transfer the received XDK back to the pair and call `skim()`.
- This triggers the first recycle — resetting `lastRecycleTime` to `block.timestamp` and starting the cycle counter.

**Phase 3 — Buy Loop (Accumulate XDK)**
- While the contract holds XDK_BASE, buy XDK from `TARGET_PAIR` in chunks of ~10% of `reserve0`.
- Limiting to 10% per swap keeps the price impact manageable and avoids the per-recycle cap (`r0/10`).
- Each buy sends XDK_BASE to the pair and swaps out XDK. The sell-side fee logic triggers recycle each time, inflating the pair's XDK balance beyond its reserve.
- ~21 iterations until all borrowed XDK_BASE is spent.

**Phase 4 — Skim Drain Loop**
- Up to 55 iterations, check if `thisRecycleMaxBalance <= thisRecycleBalance` (cycle exhausted).
- Each iteration: transfer XDK to the pair (triggers another recycle → balance inflates further), then call `skim()` to extract the excess.
- Net effect: attacker gets back more XDK than they sent in.

**Phase 5 — Swap Drain Loop**
- Swap all remaining XDK back to XDK_BASE via `TARGET_PAIR`, again in ~10%-of-reserve chunks.
- Each swap further triggers recycle, compounding the drain.

**Phase 6 — Repay & Profit**
- Repay flash loan: `borrowedAmount * 10000 / 9975 + 1` (~100.04M XDK_BASE).
- Swap remaining XDK_BASE profit → WBNB via the PancakeSwap router.
- Final profit: **6.840316534082275362 WBNB**.

| Metric | Value |
|--------|-------|
| Flash borrowed | ~99.78M XDK_BASE (`99,789,278,778,620,420,792,392,638`) |
| Flash repaid | ~100.04M XDK_BASE (`100,039,377,221,674,607,310,669,312`) |
| Pre-attack XDK reserve | ~11.31M (`11,311,910,895,281,514,098,782,000`) |
| Pre-attack XDK_BASE reserve | ~14.83M (`14,838,229,195,602,116,419,462,362`) |
| Post-attack XDK reserve | ~9.49M (`9,499,198,844,327,752,138,392,151`) |
| Post-attack XDK_BASE reserve | ~9.32M (`9,329,280,916,612,873,261,152,222`) |
| Profit | **6.840316534082275362 WBNB** (~6.84 BNB) |
| Gas used | 27,900,996 |

---

```
--- XDK/XDK_BASE pair reserves BEFORE ---
  XDK (token0) reserve:     11311910.895281514098782000
  XDK_BASE (token1) reserve: 14838229.195602116419462362

attack...
SUCCESS!

--- Exploit contract balances AFTER ---
  WBNB profit:        6.840316534082275362
  XDK_BASE remaining: 0.000000000000000000
  XDK remaining:      0.000000000000000000

--- XDK/XDK_BASE pair reserves AFTER ---
  XDK (token0) reserve:     9499198.844327752138392151
  XDK_BASE (token1) reserve: 9329280.916612873261152222

  Profit (WBNB): 6.840316534082275362
```

---

## References

- [BSC Block 81556793](https://bscscan.com/block/81556793)
- [Attacker EOA on BscScan](https://bscscan.com/address/0xb94F61855f616057a6Dc790C2269A33D1B13A0Ed)
- [Attack Contract on BscScan](https://bscscan.com/address/0x1e7e4e41DefdE022E78AdD6f6e406a7520B63c70)
- [XDK Token on BscScan](https://bscscan.com/address/0x02739BE625f7A1Cb196F42dceEe630C394DD9FAA)
- [DeFiLlama — BSC Hacks](https://defillama.com/hacks)
