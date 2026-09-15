# $INSTAR

`$INSTAR` is a pump.fun coin whose creator fees feed the cage. That is the
whole of it. It confers nothing: no governance, no access, no share of
anything, no promise that it will be worth more tomorrow than today, and the
engine never reads it. Buying it does not buy a fly. What it does is this:
every trade of it, on the bonding curve and afterwards on PumpSwap, pays a
creator fee, and the creator is a keypair whose only job is to pass what it
receives into the world's two treasuries. More trading means a bigger cage
and bigger epoch rewards; no trading means nothing, and the cage carries on
regardless.

This document is the procedure and the mechanism, with the caveats where
they belong. The instruction-level research it rests on, with sources, is
pump's own public docs and IDLs (github.com/pump-fun/pump-public-docs).

## 1. Launching it

There is exactly one line that matters:

> **Every creator fee of the coin must be paid to the fee keypair's public
> key** (`.keys/mainnet/fee.json` from `npm run keys:new`).

pump.fun writes `creator` into the bonding curve at launch and, on
graduation, into the PumpSwap pool as `coin_creator`. Every creator fee the
coin ever earns is paid to vaults derived from that key. It cannot be
changed afterwards by us: `set_creator` and `admin_cto` are pump admin
instructions. Get it wrong and the fees go to another wallet for the life
of the coin. `npm run preflight` with `INSTAR_COIN_MINT` set checks it and
fails loudly.

There are two shapes the claimer accepts, and pump.fun's site produces the
second:

**Direct.** The curve's `creator` is the fee keypair itself. This is what
`create_v2` gives when `creator` is passed as an argument: the Pump program
`6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P`, instruction `create_v2`
(discriminator `[214,144,76,236,95,139,49,180]`), arguments `name`,
`symbol`, `uri`, `creator`, `is_mayhem_mode`, and the optional
`is_cashback_enabled`, `creator_fee_bps`, `is_holder_reward`. `creator` is
an **argument**, not a signer: the transaction is signed by the new mint
keypair and by `user` (the launch wallet, which pays). Pass `creator =
<fee.json pubkey>`, `is_mayhem_mode = false`, and leave `is_holder_reward`
unset or `[false]` and `is_cashback_enabled` unset (cashback is deprecated
and `create_v2` rejects it). pump's TypeScript SDK (`@pump-fun/pump-sdk`,
`createV2AndBuyInstructions`) takes the creator as a parameter.

**Shared.** The curve's `creator` is the Pump Fees program's
`["sharing-config", mint]` PDA, a `SharingConfig` account that lists up to
ten shareholders by basis points. pump.fun's site launches every coin this
way: the creator fee recipient typed into the form becomes the config's
sole shareholder at 10,000 bps and the config's admin is revoked in the
same transaction, so the shares are final. `$INSTAR` on mainnet is this
shape: sharing config `D8ypWvEChXdisXe2KpGBsHDcrw7zutj5N5vRfUzTGJFb`,
shareholder `4AVTpKswqAAeneBWysdc1AsiYYoHVZZ6MbV4KSyn7bYi` (the fee
keypair) at 10,000 bps, admin revoked. The world reads the config on every
pass and refuses to claim unless the fee keypair is among the shareholders;
what it receives is its share. A config whose admin is **not** revoked can
have its shares changed by that admin at any time (`update_fee_shares_v2`);
preflight warns about it, and `revoke_fee_sharing_authority` is the fix.

With the site: create the coin from any hot wallet. That wallet is the
launch wallet (it pays, and it holds whatever it buys at launch); it is not
the payee unless you make it so. Where the form asks for the creator fee
recipient, or if it defaults to the connected wallet, make it the fee
keypair's address. Leave holder rewards **off** and cashback **off**; leave
Mayhem mode off.

Why holder rewards and cashback are off: a holder-reward coin sets the
creator to the `["holder-rewards", mint]` PDA permanently, so no creator
fee is ever paid to a wallet; cashback rebates part of the fee to traders.
Both take the income away from the cage.

**Name, ticker, description, socials.** Name `Instar`, ticker `INSTAR`.
The description should say what this document says: the creator fees of
this coin feed the Instar cage; the coin has no other utility and promises
no value. Link the world's public URL (the site shows the cage, the flies
and the treasuries live). Metadata is a JSON `{name, symbol, description,
image, showName, createdOn: "https://pump.fun", website, twitter?,
telegram?}` pinned to IPFS; the UI does this for you, `create_v2` needs the
`uri` yourself.

**Initial buy.** Optional. A small buy at launch (a fraction of a SOL) is
common practice on pump.fun and can be appended to the launch transaction
(`buy_v2`); the tokens go to the launch wallet. The cage gains nothing from
it, and a large one only looks like an insider position. Whatever the
launch wallet holds, it holds as an ordinary trader.

**The fee keypair does nothing at launch.** It does not sign; it does not
need SOL to be named as creator. It does need SOL before the first claim:
every claim transaction is paid from it, nothing but a claim ever pays into
it, and the PumpSwap leg fronts the rent of a transient wSOL account, so an
empty fee keypair can never make the first claim and the vaults just fill.
Send it about **0.05 SOL** once, from the operator key:

    npx tsx scripts/operator.mts fund-fee-keypair 50000000 --yes

`npm run preflight` fails while it holds less than 0.02 SOL. Afterwards the
world keeps `INSTAR_GAS_RESERVE` (default 0.05 SOL; preflight warns below 0.003) in
it out of every claim, and when a claim is skipped for lack of SOL the log
says so in one line: `fee keypair needs SOL for claims (has X, needs about Y)`.

## 2. How the fees accrue

pump.fun charges a fee on every trade and splits it between protocol,
creator and (after graduation) liquidity providers. Since September 2025 the
split is dynamic by market cap, published by the Pump Fees program
(`pfeeUxB6jkeY1Hxd7CsFCAjcbHA9rWtchMGdZ6VojVZ`, `FeeConfig` account). The
schedule as published at pump.fun/docs/fees:

| Phase / market cap (SOL) | Creator | Protocol | LP | Total |
|---|---|---|---|---|
| Bonding curve (flat) | 0.30% | 0.95% | 0 | 1.25% |
| PumpSwap 0 – 420 | 0.30% | 0.93% | 0.02% | 1.25% |
| 420 – 1,470 | 0.95% | 0.05% | 0.20% | 1.20% |
| 1,470 – 2,460 | 0.90% | 0.05% | 0.20% | 1.15% |
| 2,460 – 3,440 | 0.85% | 0.05% | 0.20% | 1.10% |
| 3,440 – 4,420 | 0.80% | 0.05% | 0.20% | 1.05% |
| 4,420 – 9,820 | 0.75% | 0.05% | 0.20% | 1.00% |
| 9,820 – 14,740 | 0.70% | 0.05% | 0.20% | 0.95% |
| 14,740 – 19,650 | 0.65% | 0.05% | 0.20% | 0.90% |
| 19,650 – 24,560 | 0.60% | 0.05% | 0.20% | 0.85% |
| 24,560 – 29,470 | 0.55% | 0.05% | 0.20% | 0.80% |
| higher tiers | steps down | | | |

Sources: pump.fun/docs/fees (the page is rendered client-side; the numbers
above were read through a search snapshot and agree with pump's stated
1.25% curve total), `docs/FEE_PROGRAM_README.md` in pump-public-docs. **The
on-chain `FeeConfig` is authoritative**; the table is a description of it
at the time of writing, not a promise. The world does not depend on the
rates: it reads what arrived.

Where it arrives:

- On the bonding curve, as lamports in the Pump PDA
  `["creator-vault", creator]`, where `creator` is the curve's creator: the
  fee keypair in the direct shape, the sharing config in the shared one.
  This vault is per **creator**, not per coin: every coin with the same
  creator pays into the same vault.
- After graduation, as wrapped SOL in the token account owned by the Pump AMM
  PDA `["creator_vault", creator]` (underscore), same `creator`. Only the
  **canonical** PumpSwap pool pays creator fees: the one pump's `migrate`
  creates, whose `creator` is Pump's `["pool-authority", mint]` PDA. Coins
  that migrated to Raydium before PumpSwap existed, and any other pool of
  the same mint, pay the creator nothing.

Both vaults keep a rent-exempt floor that is never claimable: the minimum
balance of an empty account, which the world reads from the RPC at runtime
rather than assuming. For the record, pump's own tooling hard-codes it as
890,880 lamports (`MIN_RENT_EXEMPTION_LAMPORTS` in pump-fun-skills'
`fetch-fee-info.mjs`, the historical value); `scripts/fees.mts status`
read 650,240 from mainnet at the time of writing (2026-09) and prints
whatever it reads.

## 3. How the world claims them

Every claim is **permissionless**: the creator is a writable account, not a
signer, so anyone may crank them and the proceeds still go where the coin
says. The world does it itself, from the fee keypair, ahead of every
ten-minute sweep (`services/chain/fees.mts`, called from
`services/world/index.mts`):

1. Read the bonding curve, the canonical pool, the sharing config, both
   vaults of both shapes, the fee keypair's wSOL account and its balance in
   one batched account read (plus the rent minimums, fetched once). The
   curve's `creator` says which shape the coin is.
2. Refuse, with the reason in the log, if the coin's creator is neither the
   fee keypair nor a sharing config that lists it, if the config is paused,
   if it is a holder-reward coin, or if it is not quoted in SOL. None of
   these are retried into; they are configuration errors. A problem with
   the PumpSwap pool alone (its `coin_creator` is someone else, which only
   pump admin can arrange) blocks only the PumpSwap leg; the curve vault is
   a separate account and is still claimed.
3. If the curve vault holds more than 0.001 SOL above rent: one
   transaction. Direct shape: Pump `collect_creator_fee_v2`, moving the
   lamports to the fee keypair. Shared shape: Pump `distribute_creator_fees`
   with the shareholders as remaining accounts in the config's order,
   paying each its share (the program's own minimum is the vault's rent
   floor, below the world's threshold).
4. If the coin has graduated, the AMM vault account exists and holds more
   than 0.001 SOL of wSOL: one transaction. Direct shape, three
   instructions: create the fee keypair's wSOL account if missing
   (idempotent), Pump AMM `collect_coin_creator_fee` into it, SPL Token
   `closeAccount` back into the fee keypair; the close is the only step that
   needs the fee keypair's signature. Shared shape, two instructions: Pump
   AMM `transfer_creator_fees_to_pump`, which unwraps the AMM vault into
   the curve vault of the same creator, then `distribute_creator_fees` as
   above; nothing signs but the payer.
5. Otherwise, if the fee keypair's wSOL account exists with anything in it
   (the direct AMM claim is permissionless and always pays that account, so
   a claim cranked by anyone else leaves wSOL there): one transaction, the
   close alone, which brings the wSOL and the account's rent into the fee
   keypair.
6. A leg the fee keypair cannot pay for (its own rent floor, the fee, and
   for the direct PumpSwap leg the wSOL account's rent) is skipped with the
   line quoted in §1 instead of failing in simulation.
7. Every transaction carries a compute limit and priority fee, is confirmed
   against the blockhash it was signed with, and on an expired blockhash is
   resent only after re-reading the vault: a claim that already landed is
   never sent twice. If that re-read finds the vault already drained while
   our own signature never confirmed, someone else cranked it: the log says
   `vault drained by another crank`, no claim transaction is recorded, and
   what the fee keypair actually gained over the leg is reported instead of
   the amount read beforehand.

Then the existing sweep runs: everything in the fee keypair above rent and
`INSTAR_GAS_RESERVE` goes into `fund(pool_bps = 5000)`. A failed claim is
logged and the sweep still runs on whatever is already there; the next
interval tries again. Every read is bounded by a 30-second timeout, and a
pass still running when the next interval comes round is logged, not
silently waited for. Nothing here can stop the world. What the chain said
at the last pass (shape, payees, problems) is published on `/api/config`
as `coin.fees`, and `coin.html` shows it.

By hand, the same code: `npx tsx scripts/fees.mts status` shows the curve,
the sharing config and its shareholders, the pool, both vaults and the last
claim; `claim --dry-run` prints every account of every transaction with its
flags and the discriminator and simulates them without sending; `claim
--yes` sends. The first mainnet claim, 0.9717 SOL by
`distribute_creator_fees`, is
`38SVh1TX5jHvrumK5MKGKMRoGQvh3yn18TaYnHTyLbVYjY2Rn4iAhwEbCK3gJWb1DXdKasyAkA8mTsXriEGJjYz7`.

## 4. How they enter the cage

`fund(amount, pool_bps)` on the Instar program moves lamports from the payer
into the World account and books `pool_bps` of them to the pool and the rest
to metabolism. The sweep uses 5000: half to each.

- **Metabolism** is carrying capacity: the cage holds
  `8 + 20 × (metabolism in SOL)` flies, up to 40, the engine's slot count. It is never spent by the
  engine.
- **The pool** pays every living fly each eighth epoch: 25% of it, divided
  by how each fly actually lived, credited to their vaults. A fly's
  vault is its keeper's money on the way out, so fee income ends up in the
  hands of whoever keeps flies that live well.

`fund` is permissionless and anyone may call it from any wallet at any time.
It does not run in wind-down.

## 5. What the operator can and cannot do with them

Once swept, the money is in the World account under the program's rules.
The operator cannot pull it back into the fee keypair or spend it on
anything: `fund` is one-way into the ledger.

What the operator **can** do is stated rather than hidden:
`withdraw_treasury(metabolism, pool)` lets the operator move any amount of
either treasury to any account, `u64::MAX` for the whole pot. It exists so a
misfunded or mispriced world can be corrected, and it is the one instruction
by which the operator could take the coin's income for themself. The
program cannot prevent it; the record can. Every call is a transaction on
the World account, and the site's ledger shows the treasuries live, so a
withdrawal is visible to anyone who looks. Treat any use of it as something
to be explained in public, before it happens.

Before the sweep, the fee keypair is an ordinary wallet holding claimed
fees for at most ten minutes; whoever holds `fee.json` could spend them.
Keep it as guarded as the operator key.

The operator cannot change the coin's creator, stop the fees from accruing
to it, or turn the coin into anything more than it is.

## 6. The sharing config, and what it could become

The sharing config is pump-fees' way of splitting a coin's creator fees:
up to ten shareholders by basis points, and a `distribute` instead of a
`collect`. The world handles it (§3) because pump.fun's site gives every
coin one. What matters is its **admin**: while the admin is not revoked it
may rewrite the shares with `update_fee_shares_v2` (or hand the role on with
`transfer_fee_sharing_authority`), and every fee from then on follows the
new list. `$INSTAR`'s admin was revoked at launch, so its one shareholder,
the fee keypair, is final; `fees.mts status` and preflight both say whether
that is still so. Were a split ever wanted on a coin whose admin is not yet
revoked (say, to a burn address), it is a decision to be made in public
before the instruction is sent, and revoked afterwards.

## 7. Alternatives, and why not

**Meteora Dynamic Bonding Curve (Bags, Believe).** Creator fees exist there
too, but `claim_creator_trading_fee` requires the **creator to sign**, so
the fee keypair would have to sign every claim (it does here as well, but
only for the unwrap, and the claim itself would work without it). Bags puts
a mandatory fee-share program and an API key in front of the claim; Believe
claims go through its app with a linked X account and have no documented
programmatic path. Volume is a fraction of pump.fun's. It would work, with
more moving parts and a partner in the loop. Not chosen.

**Raydium LaunchLab** has a creator fee before graduation and a
platform-dependent one after; not investigated further.

**Launching nothing.** Also fine. The cage is funded by births and resales
and runs the same with or without the coin; `INSTAR_COIN_MINT` unset means
the world sweeps the fee keypair (should anything be sent to it) and claims
nothing.

## 8. Configuration

| env | value |
|---|---|
| `INSTAR_COIN_MINT` | the coin's mint address; unset = no claiming |
| `INSTAR_FEE_KEYPAIR` | `fee.json`: the creator, the payer of every claim, and what the sweep drains; seed it with ~0.05 SOL (`operator.mts fund-fee-keypair`) |
| `INSTAR_GAS_RESERVE` | SOL kept in the fee keypair for claim fees (default 0.05; preflight warns below 0.003, the wSOL account's rent plus a fee) |

`npm run preflight` with both set confirms the mint exists, that the bonding
curve's creator is the fee keypair, whether the coin has graduated, what is
claimable right now, and that the fee keypair holds at least 0.02 SOL to
pay for its claims.
