# TAP Protocol

TAP is an Ordinals metaprotocol for fungible tokens and token backed state on Bitcoin L1.

The base token model uses inscription based deploy, mint, and transfer operations. TAP extends that model with account level actions, signed authorities, internal transfers, locks, delegated execution, staking pools, sale authorities, and AMM authorities.

The protocol has two layers:

| Layer | Purpose |
| --- | --- |
| External token layer | Deploy, mint, and inscribe transferable token balances. |
| Internal action layer | Tap confirmed inscriptions that update account state without moving a transferable inscription. This layer powers mass sends, trades, authorities, locks, staking, sales, AMM pools, and privileged mints. |

An inscription is considered tapped when it is sent back to the account that owns or controls the action. Some signed authority redeems do not require tapping by the submitter because the authority signature is the authorization.

All ticker comparisons are case insensitive. Indexers store and compare TAP tickers in lowercase. Tickers are Unicode visible length strings. Before block `861576`, valid token tickers are 3 visible characters or 5 to 32 visible characters. From block `861576`, valid token tickers are 1 to 32 visible characters.

## Activation Heights

Mainnet activation heights for this specification are:

| Feature | Height |
| --- | ---: |
| TAP index start | `779832` |
| TAP protocol start | `801993` |
| Jubilee handling | `824544` |
| DMT support | `817705` |
| Privilege authorities | `841682` |
| Full ticker length range | `861576` |
| Value stringify gate | `885588` |
| DMT NAT miner rewards | `885588` |
| Token authority whitelist fix | `916233` |
| Token locks, delegated locks, certified control, staking, sales, AMM authorities, and conditional obligations | `999999999` |
| Miner reward shield | `941848` |
| Miner reward transfer execution shield | `942002` |

Non-mainnet indexers may activate feature gates at height `0` by local configuration. Mainnet activation heights are part of consensus for production indexers.

Height `999999999` is the mainnet activation gate for the action authority feature set in this specification. A production release can only change that height by changing the consensus configuration used by indexers.

From the value stringify gate, `max`, `lim`, and `amt` values must not be JSON numbers. They must be encoded as strings. The same rule is rechecked for action amounts that are produced from delegated templates.

## Amounts

Amounts are parsed against the token deployment decimals and stored in atomic units.

```json
{
  "p": "tap",
  "op": "token-deploy",
  "tick": "tap",
  "max": "21000000",
  "lim": "1000",
  "dec": "18"
}
```

`dec` is optional for `token-deploy`. If omitted, it defaults to `18`. An indexed decimal override must parse to an integer from `0` to `17`. Values outside the override range are not used and the default `18` remains. DMT deployments use `0` decimals.

`lim` is optional. If omitted, minting has no per mint limit beyond the remaining supply.

Protocol amount strings use unsigned decimal notation without commas, signs, or exponent notation. Fractional digits beyond the deployed decimal precision are truncated toward zero before the value is converted to atomic units. A spend, mint, lock, contribution, stake, or pool action is invalid when the resulting atomic amount is zero or exceeds the protocol maximum.

## External Token Operations

### token-deploy

Deploys a TAP token.

```json
{
  "p": "tap",
  "op": "token-deploy",
  "tick": "tap",
  "max": "21000000",
  "lim": "1000",
  "dec": "18"
}
```

Rules:

- `tick`, `max`, and `op` are required.
- `max` must resolve to a positive atomic amount not larger than `18446744073709551615` whole tokens at the token decimals.
- `lim`, if present, must resolve to a positive atomic amount within the same maximum.
- A ticker can only be deployed once in its normalized form.
- A deployment can carry optional `dta` metadata with a string value up to 512 bytes.
- A deployment can carry `prv` to bind minting to a privilege authority.

### token-mint

Mints an already deployed token.

```json
{
  "p": "tap",
  "op": "token-mint",
  "tick": "tap",
  "amt": "1000"
}
```

Rules:

- `tick` and `amt` are required.
- `amt` must resolve to a positive atomic amount.
- The mint must not exceed the deployment supply or the mint limit.
- If the deployment has a privilege authority, the mint must include a valid `prv` object signed by that authority.
- A mint can carry optional `dta` metadata with a string value up to 512 bytes.

### token-transfer

Creates a transferable inscription.

```json
{
  "p": "tap",
  "op": "token-transfer",
  "tick": "tap",
  "amt": "100"
}
```

Rules:

- `tick` and `amt` are required.
- `amt` must resolve to a positive atomic amount.
- The owner must have enough available balance when the transfer inscription is created.
- Available balance is `balance - transferable - locked`. At and after the action authority activation, open account obligations are also reserved from the same balance.
- A transfer can carry optional `dta` metadata with a string value up to 512 bytes.

## Jubilee Rules

From block `824544`, new cursed inscriptions are no longer accepted.

Before block `824544`, cursed `token-deploy`, `token-mint`, and `token-transfer` inscriptions are addressed without a dash in the inscription. Indexers internally prefix those tickers with `-` to separate them from their non-cursed counterpart.

From block `824544`, transfer inscriptions for cursed tokens must use the dashed ticker. Deployments with a leading dash are invalid.

## Internal Account Operations

Internal operations update account state after tapping. They do not require a separate transferable inscription per recipient.

### block-transferables

Blocks future incoming transferable inscriptions for an account.

```json
{
  "p": "tap",
  "op": "block-transferables"
}
```

This protects accounts that rely on authority balances or signed actions. Existing transferables remain valid.

### unblock-transferables

Re-enables incoming transferable inscriptions for an account.

```json
{
  "p": "tap",
  "op": "unblock-transferables"
}
```

### token-send

Sends one or more token amounts from the tapped account.

```json
{
  "p": "tap",
  "op": "token-send",
  "items": [
    {
      "tick": "tap",
      "amt": "10000",
      "address": "bc1p9lpne8pnzq87dpygtqdd9vd3w28fknwwgv362xff9zv4ewxg6was504w20"
    },
    {
      "tick": "edg",
      "amt": "50",
      "address": "bc1p063utyzvjuhkn0g06l5xq6e9nv6p4kjh5yxrwsr94de5zfhrj7csns0aj4"
    }
  ]
}
```

Rules:

- `items` must be a non-empty array.
- Every item needs `tick`, `amt`, and `address`.
- Addresses are trimmed. Valid `bc1` addresses are lowercased.
- Each amount spends available balance only.
- Each item can carry optional `dta` metadata with a string value up to 512 bytes.
- Valid items credit receivers and debit the sender. Invalid semantic items are skipped.

### token-trade

`token-trade` creates direct TAP token to token trades.

Create a trade:

```json
{
  "p": "tap",
  "op": "token-trade",
  "side": "0",
  "tick": "tap",
  "amt": "1",
  "accept": [
    {
      "tick": "edg",
      "amt": "0.1"
    }
  ],
  "valid": "900000"
}
```

Fill a trade:

```json
{
  "p": "tap",
  "op": "token-trade",
  "side": "1",
  "trade": "<inscription id>",
  "tick": "edg",
  "amt": "0.1",
  "fee_rcv": "<fee receiver address>"
}
```

Cancel a trade:

```json
{
  "p": "tap",
  "op": "token-trade",
  "side": "0",
  "trade": "<inscription id>"
}
```

Rules:

- `side: "0"` creates or cancels from the seller side.
- `side: "1"` fills from the buyer side.
- A fill must match one accepted ticker and amount exactly.
- The seller and buyer must both have sufficient available balances at fill time.
- `valid` is a block height. A trade cannot be filled after that height.
- `fee_rcv`, if provided, receives the fixed trade fee according to the indexed rule set.
- Trades require tapping by the account that creates, fills, or cancels them.

## token-auth

`token-auth` lets an account create an authority and later execute signed redeems. It is the base primitive for signed policy decisions with onchain settlement.

An authority is identified by the inscription id of the tapped `token-auth` creation inscription.

### Create an Authority

```json
{
  "p": "tap",
  "op": "token-auth",
  "sig": {
    "v": "0",
    "r": "51143521410606380758535576062355234772504706283689533465002520447203156100709",
    "s": "23524754809729078525228087002160468580495709275023865450917881139756565577560"
  },
  "hash": "0f30c22be2f46e819538ca1281aadb82d3928cae5a699cade80013c5b14871e4",
  "salt": "0.25991027744263695",
  "auth": [
    "tap"
  ]
}
```

Rules:

- `auth` must be an array.
- If `auth` is empty, the authority can operate on any deployed token held by the authority account.
- If `auth` is not empty, every listed ticker must be deployed.
- The inscription must be tapped by the authority account.
- The signature is secp256k1.
- The signed message is `sha256(JSON.stringify(auth) + salt)`.
- `hash` is used for public key recovery.
- A compact signature can only be used once.

### Cancel an Authority

```json
{
  "p": "tap",
  "op": "token-auth",
  "cancel": "<authority inscription id>"
}
```

Rules:

- The cancel inscription must be tapped by the authority account.
- The referenced authority must exist and must not already be cancelled.
- Cancellation retires the authority for new exposure-creating actions. It must not revoke settlement paths for obligations and positions that already exist.
- After cancellation, new direct redeem items are invalid.
- After cancellation, new exposure-creating actions are invalid. This includes `lock`, `execute`, `auth-cfg`, `stake`, `fund-sale`, `contribute`, `sync-ext`, `add-liq`, `swap`, `ob-open`, `perp-policy`, `perp-open-group`, and `perp-join`.
- After cancellation, settlement and exit actions remain valid when their own conditions are met. This includes `claim`, `refund`, `claim-rwd`, `unstake`, `finalize-sale`, `claim-sale`, `refund-sale`, `cancel-sale`, eligible `withdraw-sale`, `rm-liq`, `ob-claim`, `ob-refund`, `ob-final`, `cancel-delegation`, `perp-cancel`, `perp-refund`, `perp-activate`, `perp-close`, `perp-liquidate`, `perp-settle`, and `perp-claim`.
- Existing locks, stake positions, sale contributions, sale inventories, AMM LP positions, obligations, perp groups, perp positions, and authority balances are not removed by cancellation.
- Existing consumed locks and obligations are not changed by cancellation.

This distinction is required for user safety. Once an authority has created a lock, accepted a position, joined a perp group, or recorded an LP position, cancellation must not block claims, refunds, unstaking, reward claims, sale claims, sale refunds, obligation settlement, perp terminal/exit actions, or LP removal.

### Redeem Items

Redeem items are signed by the authority and can be inscribed by anyone.

```json
{
  "p": "tap",
  "op": "token-auth",
  "sig": {
    "v": "1",
    "r": "113472523327934685528808901641630457916054343054143422440331961430719594721038",
    "s": "21393407019197854961723689634443789868582208930187383447036700552814535514199"
  },
  "hash": "82e2b0d098dcdab820ff866b011250af8841a6b59cedd7164bb94b63d2598de9",
  "salt": "0.46074583388095514",
  "redeem": {
    "auth": "<authority inscription id>",
    "items": [
      {
        "tick": "tap",
        "amt": "546",
        "address": "bc1p9lpne8pnzq87dpygtqdd9vd3w28fknwwgv362xff9zv4ewxg6was504w20"
      }
    ],
    "data": ""
  }
}
```

Rules:

- `redeem.data` must be present.
- The signed message is `sha256(JSON.stringify(redeem) + salt)`.
- The redeem public key must match the authority creation public key.
- The compact redeem signature can only be used once.
- If the authority `auth` list is not empty, every item ticker must be in that list.
- Redeem items are processed like `token-send` from the authority account.
- Redeem items do not require tapping by the submitter.

## Redeem Actions

Redeems can carry an `actions` array instead of, or alongside, `items`. Actions are authority signed operations that can lock tokens, release locks, configure authority pools, manage staking positions, run sale flows, and update AMM pool state.

```json
{
  "p": "tap",
  "op": "token-auth",
  "sig": {
    "v": "1",
    "r": "...",
    "s": "..."
  },
  "hash": "...",
  "salt": "...",
  "redeem": {
    "auth": "<authority inscription id>",
    "actions": [
      {
        "op": "lock",
        "kind": "htlc",
        "tick": "tap",
        "amt": "10",
        "claim": "bc1p...",
        "refund": "bc1p...",
        "condition": {
          "type": "hashlock",
          "hash": "<sha256 hex>"
        },
        "refund_after": "950000"
      }
    ],
    "data": {}
  }
}
```

General action rules:

- `actions` must be a non-empty array.
- Each action has an `op` string.
- Each token amount spends available balance only.
- Account available balance is `balance - transferable - locked - open account obligations`.
- Lock creation records the full committed amount.
- Claiming a lock sends the main amount to `claim` and sends allocations to their targets.
- Refunding a lock sends the full committed amount, including allocations, to `refund`.
- An action redeem that spends a token must satisfy the authority ticker whitelist if the authority has one.
- Authority cancellation blocks new exposure-creating actions and new obligations. It must not block settlement or exits for obligations and positions that already exist.

Canonical input rules:

- Redeem action fields are validated by their action shape. They do not change the global stringify handling for `max`, `lim`, and `amt`.
- `max`, `lim`, and `amt` keep the value stringify rule described above. Numeric fields outside those names define their own parsing rule inside their action shape.
- TAP token amounts are parsed with the deployed token decimals. Extra fractional digits are truncated toward zero. The protocol does not assume 18 decimals.
- External asset reserve values are atomic unsigned integer strings. The declared external decimal count identifies the unit but does not convert human decimals inside TAP.
- Key affecting identifiers must be non-empty ASCII strings up to 128 bytes using only letters, numbers, `.`, `_`, `:`, and `-`.
- Metadata keys must be non-empty ASCII strings up to 64 bytes using only letters, numbers, `.`, `_`, `:`, and `-`.
- Metadata string values are limited to 512 bytes. Metadata values can also be null, booleans, safe non-negative integers, or nested metadata objects when the action shape accepts metadata.
- Metadata objects are canonicalized with sorted keys, can nest up to four levels, cannot use arrays, and must serialize to at most 1024 bytes.
- Raw user objects that affect records are canonicalized or rejected before they are stored.
- Scalar fields reject booleans, nulls, arrays, objects, raw JSON numbers, or whitespace-only values unless the action shape explicitly allows that type.
- Unknown fields are rejected or ignored according to the action shape. They must not affect balance changes, keys, record identity, or state transitions. Signed objects are hashed exactly as specified by their signature domain.
- Same-redeem actions are evaluated in order. If any action fails, the entire redeem fails atomically.
- Faulty inputs must fail atomically. A rejected redeem must not write partial balance, lock, authority, obligation, sale, staking, AMM, perp, nonce, delegation, event, transfer, or index state.
- Accepted redeem rows are written only after every action side effect is valid and committed.

Common field classes:

| Class | Rule |
| --- | --- |
| TAP amount | Human decimal string parsed with the deployed TAP token decimals and stored in atomic units. Extra fractional digits are truncated toward zero. Zero spend values, negative values, exponent notation, comma notation, malformed strings, and raw JSON numbers are invalid after the value stringify gate. |
| Atomic integer | Unsigned integer string already expressed in atomic units. Used for AMM shares, external reserves, external heights, timestamps, basis points, and other counters that are not TAP token amounts. Leading zeros, decimals, exponent notation, negatives, commas, malformed strings, and raw JSON numbers are invalid. |
| Height | Non-negative block height in the form accepted by the action shape. Lock, sale, and obligation storage heights normalize to a 32-bit block height. AMM heights and delegation block bounds use canonical unsigned integer strings where their shapes require strings. |
| Safe id | Non-empty ASCII identifier up to the field limit, with no slash, no control character, and no whitespace-only value. |
| Ticker | Protocol ticker normalized to lowercase for keys. User display casing is outside consensus. |
| Target | Canonical compact object whose `tt` selects the target adapter and whose other keys are validated by that adapter. |
| Asset | Canonical TAP or external asset object. TAP assets reference deployed tickers. External assets use atomic reserve accounting and declare their decimal precision as metadata. |
| Metadata | Bounded object data that can be committed to records but cannot grant permission or bypass action invariants. |

### Action Matrix

| `op` | Shape | Main invariant | What it enables |
| --- | --- | --- | --- |
| `lock` | `{ op, kind, tick, amt, claim, condition, refund?, refund_after?, data?, al?, fee?, control? }` | Owner must have available balance for `amt + allocations`. Kind specific fields must match the invariant table below. | HTLC swaps, timed releases, escrow, OTC settlement, fee and reward routing. |
| `execute` | `{ op, delegation, fill, final? }` | Delegation must be signed by the authority threshold, unused, unexpired, and must materialize exactly one valid `lock`. | Pre-signed lock templates, partial authorization, final filled execution. |
| `execute-action` | `{ op, delegation, fill, final? }` | Delegation must be signed by the authority threshold, unused, unexpired, domain separated from `execute`, and must materialize exactly one supported action family. | Pre-signed action templates that are not lock templates. |
| `cancel-delegation` | `{ op, auth?, nonce }` | Redeem authority must match the cancelled authority, and the nonce must be unused and uncancelled. | Cancels unused delegated executions. |
| `claim` | `{ op, lock, preimage?, cert? }` | Lock must exist, must not be consumed, the condition must be satisfied before refund is available for hashlocks, and any scoped control must pass. | Releases locked tokens to the claim target. |
| `refund` | `{ op, lock, cert? }` | Lock must exist, must not be consumed, current block must be at least `refund_after`, and any scoped control must pass unless terminal refund is available. | Returns locked funds to the refund address. |
| `auth-cfg` with `k: "stk"` | `{ op, k, stk, rt, ctl, r, n? }` | Controller must be the current authority, stake token must exist, reward tokens must exist unless `rt` is empty, tiers must be unique and weighted. | Staking pools with weighted tiers and reward accounting. |
| `stake` | `{ op, auth, tick, amt, tier, claim }` | Staking authority must exist, `tick` must equal its stake token, tier must exist, staker must have available balance. | Opens an independent stake position. |
| `claim-rwd` | `{ op, auth, pos, rt }` | Position must be open, belong to `auth`, and have positive pending reward for `rt`. The payout target is the stored claim target and cannot be overridden by the executor. | Claims accrued staking rewards without unstaking. |
| `unstake` | `{ op, auth, pos, rt? }` | Position must be open, belong to `auth`, and current block must be at least its unlock height. | Returns staked principal after the lock duration. |
| `auth-cfg` with `k: "sale"` | `{ op, k, st, pt, ctl, tre, s, n? }` | Sale and payment tokens must exist, treasury target must be valid, height and cap rules must be coherent. | Launchpads, capped sales, allowlisted sales, refund based funding. |
| `fund-sale` | `{ op, auth, tick, amt }` | Sale authority must exist, `tick` must equal the sale token, controller must have available balance. | Deposits sale inventory. |
| `contribute` | `{ op, auth, tick, amt, claim, alw? }` | Sale must be open, `tick` must equal the payment token, caps and allowlist must pass, contributor must have available balance. | Records a buyer contribution and sale token allocation. |
| `finalize-sale` | `{ op, auth }` | Sale must not be cancelled or finalized. It can finalize at or before end only at hard cap, or after end when soft cap is met. | Sends payment pool to treasury and unlocks sale claims. |
| `resolve-sale` | `{ op, auth }` | Sale must not be cancelled or finalized. It can finalize if finalization invariants hold, or mark failed/refundable after deterministic failure conditions. | Public terminal sale resolution without controller liveness. |
| `claim-sale` | `{ op, auth, cid }` | Contribution must be open, sale must be finalized, and payout must go to the contribution claim address. | Contributor claims allocated sale token. |
| `refund-sale` | `{ op, auth, cid }` | Contribution must be open. Sale must be cancelled, failed/refundable, or ended without meeting soft cap. Payout must go to the contribution claim address. | Contributor gets payment token back. |
| `cancel-sale` | `{ op, auth }` | Controller must match, sale must allow cancellation, and sale must not be finalized or already cancelled. | Cancels an open sale. |
| `withdraw-sale` | `{ op, auth, tick, amt, tt, to }` | Controller must match. Withdrawal is allowed after finalization, cancellation, or failed end. Target must be valid. | Returns unsold inventory or moves remaining sale token balance. |
| `auth-cfg` with `k: "amm"` | `{ op, k, a, c, ctl, n?, att? }` | Controller must be the current authority, two distinct assets must be valid, fee rules must be coherent, and external pools must configure attestors. | Creates a constant product AMM authority. |
| `add-liq` | `{ op, auth, amts, min, to, exp, ref? }` | Pool must be active, both input amounts must be available, minted LP shares must be at least `min`. | Adds TAP/TAP liquidity and mints internal LP shares. |
| `rm-liq` | `{ op, auth, sh, min, to, own?, exp, ref? }` | LP owner must have enough shares, output amounts must satisfy `min`. | Burns LP shares and withdraws both pool assets. |
| `swap` | `{ op, auth, m, i, amt?, out?, min?, max?, to, exp, ref? }` | Pool must be active, slippage bound must pass, and reserve math must remain valid. | Executes exact-in or exact-out TAP/TAP swaps. |
| `sync-ext` | `{ op, auth, sid, ext, exp, sigs, salt }` | The controller must authorize the redeem, and the external snapshot must match the pool, threshold signatures, age, expiry, and replay rules. | Records attested external reserve data as policy input. |
| `ob-open` | `{ op, src, amt, cl, rf, cond, ra, exp, ctx? }` | Source adapter must be explicit and authorized. Amount, targets, condition, refund height, expiry, and context are committed. | Conditional obligations for account escrow, authority backed settlement, and AMM reserve settlement. |
| `ob-claim` | `{ op, ob, preimage }` | Obligation must be open and unconsumed. The preimage must satisfy the saved hash before refund is available. | Releases an obligation to its claim destination. |
| `ob-refund` | `{ op, ob }` | Obligation must be open and unconsumed. Current block must be at least the saved refund height. | Returns an obligation to its refund destination. |
| `ob-final` | `{ op, ob, preimage }` | Same condition as `ob-claim`, with destination adapter finalization. | Finalizes adapter settlement, for example crediting an AMM reserve. |
| `perp-policy` | `{ op, id, v, dom, net, seq, thr, signers, assets, limits, oracle, liq, def, fee, bounty, entry, exp, sigs, hash? }` | Policy id, domain, network, constraints, entry-bound policy, signers, threshold, signatures, and expiry must validate. Updates must increase `seq` and satisfy previous signer threshold. | Registers an operator policy for isolated perp groups. |
| `perp-open-group` | `{ op, pid, ph, pair, coll, form, ready, lev, close, liq, settle, def, fee, bounty, oracle, entry, ctx?, hash? }` | Referenced policy must exist and match `ph`. Pair, collateral, formation, expiry, readiness, leverage, liquidation, settlement, default, fee, bounty, oracle, and entry-bound terms must fit the policy. | Creates a non-active group in formation state. |
| `perp-join` | `{ op, gid, src, side, coll, lev, entry, claim, refund, ctx? }` | Group must be in formation, side and leverage must be allowed, entry bound must match group policy, caller must match `src`, collateral must be available, and pending same-redeem debits must not overcommit balance. Only valid for TAP-account collateral groups. | Funds a long or short TAP-collateral position in a group. |
| `perp-cancel` | `{ op, gid }` | Group must be in formation and past deadline. | Moves an unactivated group to cancelled state. |
| `perp-refund` | `{ op, gid, pos, to? }` | Group must be cancelled, position must be unrefunded, and optional `to` must equal the stored refund target. Payout always goes to the stored refund target. | Returns original collateral from a cancelled group. |
| `perp-activate` | `{ op, gid, bto?, cert }` | Group must be in formation and ready. Certificate purpose must be `entry` and match policy, group, group hash, aggregate entry bounds, state hash, sequence, signer threshold, and block validity. | Freezes entry price and moves the group to active state. |
| `perp-close` | `{ op, gid, pos, qty, cert }` | Position owner must submit, group and position must be active, quantity must be valid open collateral, and close certificate must validate. | Closes all or part of a position and reserves realized equity until settlement. |
| `perp-liquidate` | `{ op, gid, pos, cert }` | Position must be active and below maintenance at the certified price. | Closes unsafe open collateral and records liquidation accounting. |
| `perp-settle` | `{ op, gid, bto?, cert }` or `{ op, gid, bto?, fallback: "last-valid-at-expiry-v1" }` | Group must be active, expiry reached, signed settlement certificate or committed fallback must validate, and aggregate payout math must conserve locked collateral. | Records terminal settlement, fees, bounties, default state, and claim formula. |
| `perp-claim` | `{ op, gid, pos, to? }` | Group must be settled or defaulted, position payout must be unclaimed, and optional `to` must equal the stored claim target. Payout always goes to the stored claim target. | Claims a settled payout. |

### Lock Action

```json
{
  "op": "lock",
  "kind": "htlc",
  "tick": "tap",
  "amt": "10",
  "claim": "bc1p...",
  "refund": "bc1p...",
  "condition": {
    "type": "hashlock",
    "hash": "<sha256 hex>"
  },
  "refund_after": "950000",
  "data": {
    "dom": "settlement",
    "ref": "lock-1"
  },
  "al": [
    {
      "tt": "a",
      "to": "bc1p...",
      "amt": "0.2",
      "rl": "of"
    }
  ]
}
```

Fields:

| Field | Meaning |
| --- | --- |
| `kind` | Lock kind. Supported values are `htlc`, `vesting`, `cooldown`, `escrow`, and `otc`. |
| `tick` | Token ticker. Stored lowercase. |
| `amt` | Main amount locked for the claimant. |
| `claim` | Bitcoin address that receives the main amount on claim. |
| `refund` | Bitcoin address that receives the full committed amount on refund, when the kind uses refund. |
| `condition` | Claim condition. Shape depends on `kind`. |
| `refund_after` | Bitcoin block height from which refund is allowed, when the kind uses refund. |
| `data` | Optional metadata. Some kinds require `data.dom` and `data.ref`. |
| `al` | Optional allocation list for fees, rewards, authority credits, or burn. |
| `fee` | Optional shorthand for an account allocation with role `of`. |
| `control` | Optional certified control policy for claim or refund. |

The `fee` object is accepted as a shorthand for an account allocation with role `of`. `al` is the canonical allocation field.

Lock records store canonicalized conditions. Hashlock conditions store `{ "type": "hashlock", "hash": "<lowercase sha256>" }`. Height conditions store `{ "type": "height", "min": <height> }`. Authority conditions store `{ "type": "authority", "auth": "<authority inscription id>" }`. Extra condition keys are not committed to the lock record.

When `data` is present, it follows the canonical metadata rules. `data.dom` and `data.ref`, when present, must be safe identifiers. `escrow` normalizes `data.payer` and `data.payee` to validated Bitcoin addresses before the lock record is stored.

### Lock Kind Invariants

| Kind | Condition invariant | Claim invariant | Refund invariant | Data invariant | Use |
| --- | --- | --- | --- | --- | --- |
| `htlc` | `condition.type` must be `hashlock`, and `condition.hash` must be SHA-256 hex. | `claim` receives `amt` only when the preimage hashes to `condition.hash` and claim happens before `refund_after`. | `refund` receives `amt + allocations` from `refund_after` onward. | No required `data` fields. | Cross chain atomic swaps, preimage settlement, escrow with preimage release, pay for secret flows. |
| `vesting` | `condition.type` must be `height`, and `condition.min` must be a valid block height. | `claim` receives `amt` once current block is at least `condition.min`. | No refund fields are allowed. | `data.dom` and `data.ref` are required strings. | Team vesting, grant unlocks, protocol emissions, scheduled payouts. |
| `cooldown` | `condition.type` must be `height`, and `condition.min` must be a valid block height. | `claim` must equal the authority owner and can claim once current block is at least `condition.min`. | No refund fields are allowed. | `data.dom` and `data.ref` are required strings. | Withdrawal delays, unstaking cooldowns, anti-compromise waiting periods. |
| `escrow` | `condition.type` must be `authority`, and `condition.auth` must be an active token authority. | `claim` receives `amt` only when the authority named in `condition.auth` executes the claim. | `refund` receives `amt + allocations` from `refund_after` onward. | `data.dom`, `data.ref`, `data.payer`, and `data.payee` are required. `payer` must equal the lock owner. `payee` must equal `claim`. | Service escrow, dispute mediated sales, authority approved settlement. |
| `otc` | `condition.type` must be `hashlock` with a SHA-256 hash or `authority` with an active authority. | Claim follows either the hashlock rule or authority rule. | `refund` receives `amt + allocations` from `refund_after` onward. | `data.dom`, `data.ref`, and `data.cp` are required strings. | Bilateral OTC deals, private settlement tickets, negotiated fills with an explicit counterparty reference. |

### Claim Action

Claims consume a lock and credit its target.

```json
{
  "op": "claim",
  "lock": "<lock id>",
  "preimage": "<preimage>"
}
```

Rules:

- A hashlock claim must provide a preimage whose SHA-256 hash matches the lock condition.
- A hashlock claim must happen before `refund_after`.
- A height claim can execute once the chain height is at least `condition.min`.
- An authority claim can execute when the redeem authority equals `condition.auth`.
- A lock can only be consumed once.

### Refund Action

Refunds consume a lock after its refund height.

```json
{
  "op": "refund",
  "lock": "<lock id>"
}
```

Rules:

- The lock must have a `refund` address.
- The current block must be at least `refund_after`.
- The full committed amount is returned to `refund`.
- Refund availability does not expire.

### Certified Control

Certified control is optional on `lock`. It can require a threshold certificate before `claim`, `refund`, or both can consume the lock.

Lock policy:

```json
{
  "op": "lock",
  "kind": "htlc",
  "tick": "tap",
  "amt": "10",
  "claim": "bc1p...",
  "refund": "bc1p...",
  "condition": {
    "type": "hashlock",
    "hash": "<sha256 hex>"
  },
  "refund_after": "950000",
  "control": {
    "type": "cert",
    "id": "policy-1",
    "threshold": 2,
    "signers": [
      "02...",
      "03..."
    ],
    "scope": [
      "claim",
      "refund"
    ],
    "expires": "960000",
    "rules": {
      "terminal_refund_after": "970000"
    },
    "hash": "<policy hash>"
  }
}
```

Certificate:

```json
{
  "op": "claim",
  "lock": "<lock id>",
  "preimage": "<preimage>",
  "cert": {
    "v": 1,
    "policy": "policy-1",
    "action": "claim",
    "target": "<lock id>",
    "payload_hash": "<payload hash>",
    "nonce": "claim-1",
    "valid_until": "950100",
    "sigs": [
      {
        "signer": "02...",
        "hash": "<message hash>",
        "sig": {
          "v": "0",
          "r": "...",
          "s": "..."
        }
      }
    ]
  }
}
```

Policy fields:

| Field | Meaning |
| --- | --- |
| `type` | Must be `cert`. |
| `id` | Safe id for the policy. |
| `threshold` | Required signature count. Parsed as an unsigned safe integer. |
| `signers` | One to eight secp256k1 public keys. Keys are normalized to compressed form and sorted. Duplicates are invalid. |
| `scope` | Non-empty list containing `claim`, `refund`, or both. Stored in canonical order. |
| `expires` | Optional last Bitcoin block at which a scoped certificate can be used. |
| `rules.terminal_refund_after` | Required when `scope` includes `refund`. It must be greater than the lock `refund_after`. |
| `hash` | Optional policy hash supplied by the creator. If present, it must match the computed lowercase SHA-256 policy hash. |

Certificate fields:

| Field | Meaning |
| --- | --- |
| `v` | Certificate version. Missing means `1`; any other value is invalid. |
| `policy` | Must equal the lock policy id. |
| `action` | Must equal the consuming action, `claim` or `refund`. |
| `target` | Must equal the consumed lock id. |
| `payload_hash` | Lowercase SHA-256 of the canonical consuming action with `cert` removed. |
| `nonce` | Safe id. A nonce is one time for `policy + target + action`. |
| `valid_until` | Last Bitcoin block at which the certificate can be used. |
| `sigs` | Signature entries from configured signers. Each entry carries `signer`, `hash`, and `sig`. Only unique valid signers count. |

Canonical rules:

- `control` is only valid on `lock`.
- `cert` is only valid on `claim` and `refund`.
- Unknown top-level `control` or `cert` fields are invalid.
- A `cert` is invalid when the consumed lock has no matching scoped `control`.
- A scoped `claim` requires a valid `cert`.
- A scoped `refund` requires a valid `cert` before `terminal_refund_after`.
- A scoped `refund` can execute without `cert` at or after `terminal_refund_after`.
- `terminal_refund_after` never changes the normal `refund_after` rule. Refund is still invalid before `refund_after`.
- Certified-control canonical JSON sorts object keys. It accepts nulls, booleans, strings, arrays, objects, and safe integers. Unsafe numbers and undefined values are invalid.
- The policy hash is lowercase SHA-256 of the canonical policy without `hash`.
- The certificate message is:

```text
sha256(canonical_json([
  "tap-certified-control-v1",
  "tap",
  policy.id,
  policy.hash,
  action,
  target,
  payload_hash,
  nonce,
  valid_until
]))
```

- Each signature entry must declare its signer, use the certificate message hash, and recover to the declared signer.
- Threshold validation counts unique valid configured signers.
- A consumed certified action stores the canonical certificate signer set.
- A failed certificate must not consume the lock or the certificate nonce.

### Conditional Obligations

Conditional obligations are a generic settlement primitive for cases where a normal account owned lock is not the right owner model. They are separate from lock records. `lock`, `claim`, `refund`, delegated execution, staking, and sale flows keep their defined shapes.

The core lifecycle is:

```text
authorized source
        |
        | ob-open
        v
open obligation
        |
        +-- ob-claim or ob-final before refund height
        |       condition must pass
        |       destination adapter receives value
        |
        +-- ob-refund at or after refund height
                refund destination receives value
```

Open an obligation:

```json
{
  "op": "ob-open",
  "src": {
    "tt": "a",
    "to": "bc1p...",
    "tick": "tap"
  },
  "amt": "100",
  "cl": {
    "tt": "a",
    "to": "bc1p..."
  },
  "rf": {
    "tt": "a",
    "to": "bc1p..."
  },
  "cond": {
    "ty": "hash",
    "h": "<sha256 hex>"
  },
  "ra": "960000",
  "exp": "959000",
  "ctx": {
    "dom": "example",
    "ref": "operation-1"
  }
}
```

Claim, finalize, or refund:

```json
{ "op": "ob-claim", "ob": "<obligation id>", "preimage": "<preimage>" }
```

```json
{ "op": "ob-final", "ob": "<obligation id>", "preimage": "<preimage>" }
```

```json
{ "op": "ob-refund", "ob": "<obligation id>" }
```

Obligation fields:

| Field | Meaning |
| --- | --- |
| `src` | Source adapter. It decides whether value can be reserved or debited. |
| `amt` | Amount in the source asset, parsed with the source token decimals and stored in atomic units. |
| `cl` | Claim destination adapter. |
| `rf` | Refund destination adapter. |
| `cond` | Settlement condition. Supported condition type is hashlock. |
| `ra` | Refund height. Refund is valid from this Bitcoin block onward. |
| `exp` | Last block at which `ob-open` may create the obligation. |
| `ctx` | Optional context. It is committed to the obligation but does not grant permission. |

`ctx`, when present, follows the canonical metadata rules. If `ctx.ref` is present, it must be a safe identifier because it can be indexed by reference. `ctx.amm` is a structured metadata object used by AMM obligations and is still validated by the AMM context rules below.

Source adapters:

| `src.tt` | Required fields | Rule |
| --- | --- | --- |
| `a` | `to`, `tick` | Account source. The tapped account must equal `to` and have available balance after transferables, locks, and open obligations. |
| `h` | `to`, `tick` | Authority source. `to` is the source authority id, and the authority must be active when the obligation is opened. |
| `amm` | `pid`, `i` | AMM source. `pid` is the AMM authority and `i` is the reserve side. The pool controller must authorize the open and the pool must allow the action. |

Destination adapters:

| `tt` | Required fields | Rule |
| --- | --- | --- |
| `a` | `to` | Credits a Bitcoin account address. |
| `h` | `to` | Credits an authority balance if the target authority is valid for the token. |
| `amm` | `pid`, `i` | Credits an AMM reserve through AMM finalization rules. |
| `b` | none or canonical burn address | Burns the amount when burn is allowed for that destination. |

AMM obligation context:

When an obligation uses an AMM source or destination and the pool has an external leg, `ctx.amm` is required. Compact keys are used because these records can become large at scale.

| Field | Meaning |
| --- | --- |
| `pid` | AMM authority id. Must match the AMM source or destination. |
| `i` | TAP side index, `0` or `1`. Must be the TAP asset side of the pool. |
| `sid` | External reserve snapshot id. The snapshot must exist, match the external leg, and not be expired. |
| `set` | External settlement reference. The protocol commits it but does not verify external state. |
| `h` | Hashlock hash. Must equal `cond.h`. |
| `ns`, `cid`, `pool`, `aid` | Optional external identifiers. If present, each must match the snapshot. |

Obligation invariants:

- `amt` must be a protocol amount string. Negative values, exponent notation, comma notation, malformed strings, raw JSON numbers after the value stringify gate, and values that resolve to zero are invalid. Extra fractional digits are truncated toward zero.
- `ob-open` must reject unknown source and destination adapters.
- `ob-open` must reject after `exp`.
- Source availability is checked before the obligation is opened. The same redeem cannot overcommit the same balance through multiple obligation opens.
- The refund destination must restore the source adapter: account sources refund to the same account, authority sources refund to the same authority, and AMM sources refund to the same AMM side.
- AMM sources and destinations only move TAP-side reserves. TAP/TAP pools can use AMM adapters directly. TAP/external pools require `ctx.amm` plus an accepted external snapshot for the other side.
- A claim or refund can consume the obligation only once.
- Hashlock claim and finalization require a preimage whose SHA-256 hash matches `cond.h`.
- Hashlock claim and finalization must happen before `ra`.
- Refund is valid at `ra` and remains valid after it. It does not expire.
- Failed settlement must not write partial balance, status, AMM reserve, or authority updates.
- Authority cancellation blocks new obligations from that authority. It does not block claim, finalization, or refund for obligations that were already opened.

Obligations are useful when value belongs to a protocol object instead of a simple user account. Account owned locks remain the direct path for account HTLCs and delegated lock flows.

## Allocations

Allocations split claim proceeds into additional targets.

```json
{
  "al": [
    {
      "tt": "a",
      "to": "bc1p...",
      "amt": "0.2",
      "rl": "of"
    },
    {
      "tt": "h",
      "to": "<authority id>",
      "amt": "0.2",
      "rl": "sr"
    },
    {
      "tt": "b",
      "to": "1BitcoinEaterAddressDontSendf59kuE",
      "amt": "0.1",
      "rl": "burn"
    }
  ]
}
```

Rules:

- `al` can contain up to 16 entries.
- Every entry needs `tt`, `to`, `amt`, and `rl`.
- `tt: "a"` sends to a Bitcoin account address.
- `tt: "h"` sends to an authority balance.
- `tt: "b"` burns to the canonical burn address `1BitcoinEaterAddressDontSendf59kuE`.
- `rl` is a short role string matching `^[a-z0-9_-]{1,16}$`.
- Roles must be unique inside one allocation list.
- Amounts are parsed with the locked token decimals.
- Allocation amounts are added on top of the main lock amount.
- On refund, allocation amounts return to the refund address together with the main amount.
- `rl: "sr"` on a target authority credits a staking reward accumulator if that authority is a staking authority.

## Delegated Lock Execution

Delegated execution lets an authority pre-sign a lock template. A later filler provides the final values, and anyone can inscribe the final action if the signatures and constraints pass.

Authority setup:

```text
authority account
  |
  | inscribe token-auth with auth array
  | tap to own address
  v
active token authority
  |
  | owns balances and signs later policy
  v
redeem items and redeem actions
```

Delegated execution lifecycle:

```text
authority
  |
  | signs delegation:
  | auth, nonce, expiry, threshold,
  | signers, template, constraints
  v
signed delegation
  |
  | offchain discovery
  v
filler
  |
  | chooses fill values:
  | claim, refund, hash, refund_after
  | optionally gets finalizer signatures
  v
token-auth redeem with execute action
  |
  | inscribed by the filler or any broadcaster
  v
ordered inscription stream
  |
  | verifies authority, signatures, constraints,
  | nonce, expiry, whitelist, and available balance
  v
lock record
```

Delegation cancellation:

```text
authority account
  |
  | signs and taps normal redeem action
  | { op: "cancel-delegation", nonce }
  v
delegation cancel record
  |
  | same auth + nonce can no longer execute
  v
stale delegated execution is unusable
```

Delegated-only redeem:

```json
{
  "p": "tap",
  "op": "token-auth",
  "sig": {
    "v": "1",
    "r": "...",
    "s": "..."
  },
  "hash": "...",
  "salt": "...",
  "redeem": {
    "actions": [
      {
        "op": "execute",
        "delegation": {
          "auth": "<authority inscription id>",
          "nonce": "delegation-1",
          "expiry": "960000",
          "threshold": 1,
          "signers": [
            "<compressed secp256k1 pubkey>"
          ],
          "template": {
            "op": "lock",
            "kind": "htlc",
            "tick": "tap",
            "amt": "10",
            "claim": "$claim",
            "refund": "$refund",
            "condition": {
              "type": "hashlock",
              "hash": "$hash"
            },
            "refund_after": "$refund_after"
          },
          "constraints": {
            "claim": {
              "type": "btc-address"
            },
            "refund": {
              "type": "btc-address"
            },
            "hash": {
              "type": "sha256"
            },
            "refund_after": {
              "type": "block-offset",
              "base": "current",
              "min": "24",
              "max": "24"
            }
          },
          "sigs": [
            {
              "hash": "...",
              "sig": {
                "v": "0",
                "r": "...",
                "s": "..."
              }
            }
          ],
          "salt": "..."
        },
        "fill": {
          "claim": "bc1p...",
          "refund": "bc1p...",
          "hash": "<sha256 hex>",
          "refund_after": "950000"
        }
      }
    ],
    "data": {}
  }
}
```

Rules:

- Delegated-only redeems omit `redeem.auth`.
- Every action in a delegated-only redeem must be `execute` or `execute-action`.
- The delegated authority is verified from `delegation.auth`.
- `nonce` must match `^[A-Za-z0-9._:-]{1,128}$`.
- A nonce can be executed once or cancelled once.
- `expiry` is a block height. Execution fails after that height.
- `signers` are compressed or uncompressed secp256k1 public keys. They are normalized to compressed keys.
- At least one signer must be the authority public key.
- `threshold` must be positive and cannot exceed 8 or the signer count.
- The template must produce a `lock` action after placeholders are filled.
- Constraints are checked against placeholders and direct paths.
- A placeholder that is not constrained to an exact value requires finalizer signatures.

### Generic Delegated Actions

`execute-action` is separate from `execute`. It is not a lock delegation and cannot be validated with the `tap-delegated-lock-v1` or `tap-delegated-lock-v2` domains.

```json
{
  "op": "execute-action",
  "delegation": {
    "kind": "action",
    "v": "1",
    "auth": "<authority inscription id>",
    "nonce": "action-delegation-1",
    "expiry": "960000",
    "family": "perp-join",
    "threshold": 1,
    "signers": ["<compressed secp256k1 pubkey>"],
    "template": {
      "op": "perp-join",
      "gid": "<group id>",
      "src": { "tt": "a", "to": "$owner" },
      "side": "$side",
      "coll": "$collateral",
      "lev": { "n": "$leverage", "d": "1" },
      "entry": "$entry",
      "claim": { "tt": "a", "to": "$owner" },
      "refund": { "tt": "a", "to": "$owner" }
    },
    "constraints": {
      "owner": { "type": "btc-address" },
      "side": { "allowed": ["long", "short"] },
      "collateral": { "type": "number-string" },
      "leverage": { "type": "number-string" },
      "entry": { "allowed": [{ "max": { "p": "109", "q": "10000" } }] }
    },
    "sigs": [
      { "hash": "...", "sig": { "v": "0", "r": "...", "s": "..." } }
    ],
    "salt": "..."
  },
  "fill": {
    "owner": "bc1p...",
    "side": "long",
    "collateral": "100",
    "leverage": "10",
    "entry": { "max": { "p": "109", "q": "10000" } }
  }
}
```

Rules:

- `delegation.kind` must be `action`.
- `delegation.v` must be `1`.
- `delegation.family` must name the action family produced by the filled template.
- The filled template action `op` must equal `delegation.family`.
- Supported families are `perp-join` and `perp-close`.
- The action delegation message is:

```text
sha256(JSON.stringify([
  "tap-delegated-action-v1",
  "tap",
  kind,
  v,
  auth,
  nonce,
  expiry,
  family,
  threshold,
  signers,
  template,
  constraints,
  finalizers
]) + salt)
```

- Signature, signer, threshold, authority participation, nonce, cancellation, expiry, template, constraint, and finalizer rules are the same as delegated locks unless this section defines a stricter rule.
- Finalizer signatures for `execute-action` use the `tap-delegated-final-action-v1` domain and sign the action delegation message plus the filled final action.
- An old lock delegation signature cannot execute an `execute-action`.
- A generic action delegation cannot execute a `lock`.
- A nonce consumed by `execute-action` cannot be used again by `execute` or another `execute-action`.

### Delegation Signature Thresholds

Delegated execution uses threshold signatures over the delegation message.

`threshold` is the required signature count. `signers` is the full signer set. A delegation with `threshold: 2` and three entries in `signers` is therefore a 2-of-3 authorization.

The signed delegation message is:

```text
sha256(JSON.stringify([
  "tap-delegated-lock-v2",
  auth,
  nonce,
  expiry,
  threshold,
  signers,
  template,
  constraints,
  finalizers
]) + salt)
```

Delegations with `finalizers` use the `tap-delegated-lock-v2` domain and include the `finalizers` element. Delegations without `finalizers` use the `tap-delegated-lock-v1` domain and omit that element.

Validation rules:

- `signers` must contain unique secp256k1 public keys.
- Signer keys may be compressed or uncompressed. They are normalized to compressed keys before comparison.
- The signer set cannot be empty and cannot contain more than 8 keys.
- `threshold` is parsed as an integer and must be at least 1.
- `threshold` cannot exceed the signer count and cannot exceed 8.
- `sigs` may contain more entries than required, but only valid signatures from unique configured signers count.
- At least one valid counted signature must be from the authority public key. This prevents a non-authority signer group from spending an authority delegation without the authority participating.
- The delegation is still subject to nonce, expiry, ticker whitelist, template, constraint, finalizer, and balance checks. Signature threshold alone never makes an invalid action valid.

Common patterns:

| Pattern | Shape | Use |
| --- | --- | --- |
| 1-of-1 | `threshold: 1`, one signer, authority key included | Single authority signs an executable delegation. |
| 2-of-2 | `threshold: 2`, authority key and policy key | Authority and policy signer must both approve. Useful when a secondary signer enforces fee or domain policy. |
| 2-of-3 or higher | Multiple signers, threshold below signer count | Redundant policy signer or committee approval without requiring every signer to be online. |

Supported constraint types:

| Type | Rule |
| --- | --- |
| `btc-address` | Value must be a valid Bitcoin address. |
| `sha256` | Value must be a valid SHA-256 hex string. |
| `string` | Value must be a string. Optional `min` and `max` check string length. |
| `number-string` | Value must be a decimal number string. |
| `block-offset` | Value must be a block height. `base` must be `current`. At least one of `min` or `max` is required. Bounds are measured against the block that indexes the execution. |

Constraints can also use exact matching:

```json
{
  "claim": {
    "equals": "bc1p..."
  },
  "tick": {
    "allowed": [
      "tap",
      "edg"
    ]
  }
}
```

### Final Action Signatures

If `delegation.finalizers` is present, the final action must be signed by the configured finalizers. Finalizers use the same n-of-m threshold model as delegations, but they sign the filled action, not the delegation template.

```json
{
  "finalizers": {
    "threshold": 1,
    "signers": [
      "<compressed secp256k1 pubkey>"
    ]
  }
}
```

The execute action then includes:

```json
{
  "final": {
    "sigs": [
      {
        "hash": "...",
        "sig": {
          "v": "0",
          "r": "...",
          "s": "..."
        }
      }
    ],
    "salt": "..."
  }
}
```

Final signatures protect filled actions when the delegation template allows values that are not exact at delegation signing time.

The finalizer message is:

```text
sha256(JSON.stringify([
  "tap-delegated-final-action-v1",
  delegation_message,
  finalizers.threshold,
  finalizers.signers,
  final_action
]) + final.salt)
```

Finalizer validation counts unique valid signatures from `finalizers.signers`. The threshold must be at least 1, cannot exceed the finalizer signer count, and cannot exceed 8.

### Cancel a Delegation

```json
{
  "op": "cancel-delegation",
  "auth": "<authority inscription id>",
  "nonce": "delegation-1"
}
```

Rules:

- The action must be in a normal authority redeem.
- The redeem authority must match `auth`.
- The nonce must not already be used or cancelled.
- Cancellation only affects unused delegated offers. It does not unlock funds because no lock exists before execution.

## Perp Groups

Perp groups are isolated fixed-lifetime margin groups. A group has formation, active, terminal, and exit states. TAP collateral is held by the group authority id. Perp collateral inside this protocol is TAP collateral only. External quote or oracle metadata is price metadata only and does not create external custody or settlement state. No group can debit balances outside its committed collateral pool.

### Perp Policy

```json
{
  "op": "perp-policy",
  "id": "perp-main",
  "v": "1",
  "dom": "tap-perp-policy-v1",
  "net": "bitcoin:mainnet",
  "seq": "1",
  "thr": "1",
  "signers": ["02..."],
  "assets": {
    "tap": { "mode": "wildcard-or-list", "ticks": [] },
    "external": { "mode": "wildcard-or-list", "refs": [] },
    "pairs": { "mode": "wildcard-or-list", "items": [] }
  },
  "limits": {
    "max_lev": { "n": "200", "d": "1" },
    "min_coll": "1",
    "max_not": "1000000",
    "min_dur": "1",
    "max_dur": "52560",
    "min_form": "1",
    "max_form": "2016",
    "min_ratio": { "n": "0", "d": "1" },
    "max_ratio": { "n": "999999999", "d": "1" }
  },
  "oracle": {
    "rules": ["spot-vwap-v1"],
    "max_age": "144",
    "min_trades": "1",
    "min_volume": "1",
    "stale": "fallback-or-reject",
    "fallbacks": ["last-valid-at-expiry-v1"]
  },
  "liq": {
    "rules": ["isolated-maintenance-margin-v1"],
    "min_mmr": { "n": "5", "d": "1000" }
  },
  "def": {
    "rules": ["pro-rata-positive-equity-v1"],
    "dust": "largest-remainder-v1"
  },
  "fee": {
    "rules": ["position-reserve-bps-v1"],
    "max_bps": "250",
    "receivers": [{ "tt": "a", "to": "bc1p...", "share": "10000", "rl": "pf" }]
  },
  "bounty": {
    "rules": {
      "liquidate": { "mode": "position-collateral-bps", "bps": "50", "public": true }
    }
  },
  "entry": {
    "mode": "one-sided-v1",
    "required": true,
    "allow_unbounded": false,
    "max_slippage_bps": "500"
  },
  "exp": "999999999",
  "sigs": [
    { "signer": "02...", "hash": "<message hash>", "sig": { "v": "0", "r": "...", "s": "..." } }
  ]
}
```

Policy fields:

| Field | Meaning |
| --- | --- |
| `id` | Policy id. |
| `dom` | Must be `tap-perp-policy-v1`. |
| `net` | Bitcoin network identifier used by the policy. |
| `seq` | Monotonic policy sequence. |
| `thr` | Required signature count. |
| `signers` | Policy signer set. Duplicate signers reject. |
| `assets` | Allowed TAP and external assets. Empty `wildcard-or-list` lists allow the namespace. Lists must be unique after canonicalization. |
| `limits` | Bounds for leverage, collateral, notional, formation, duration, and maintenance. |
| `oracle` | Price certificate rules, staleness bounds, and fallback names. Signers and threshold are the policy signer set and `thr`. |
| `liq` | Liquidation rule set and minimum maintenance margin. |
| `def` | Group-local default and dust assignment rule. |
| `fee` | Position reserve fee rules, maximum bps, and canonical receiver list. Receiver shares must sum to `10000`. |
| `bounty` | Maximum liquidation bounty rule. |
| `entry` | Entry-bound policy for positions formed before activation. |
| `exp` | Last block where the policy action can be accepted. |

The policy hash is the canonical hash of the action without `hash` and `sigs`. The policy signature message is:

```text
sha256(canonical_json(["tap-perp-policy-v1", "tap", id, seq, policy_hash]))
```

Perp fee receivers are canonical targets:

- Account receiver: `{ "tt": "a", "to": "<address>", "share": "<bps-share>", "rl": "pf" }`.
- Operator account receiver may also use role `of`.
- Staking reward receiver: `{ "tt": "h", "to": "<staking authority id>", "share": "<bps-share>", "rl": "sr" }`.
- `share` is an integer string from `1` to `10000`.
- The receiver list must be non-empty and must not exceed `16` entries.
- Receiver shares must sum exactly to `10000`.
- Receiver order is committed and is used for deterministic dust assignment.
- Duplicate receiver triples reject.
- Duplicate receiver roles reject.
- `tt: "a"` receivers must target valid addresses and may only use `pf` or `of`.
- `tt: "h"` receivers must target an existing staking authority and may only use `sr`.
- `sr` is only valid for TAP-collateral groups. It credits the staking authority reward accumulator for the group collateral ticker.
- If the staking authority restricts reward tickers, the group collateral ticker must be allowed.
- If the staking authority has no shares, `sr` allocation is valid only when the authority empty-pool policy preserves rewards for later accounting.
- Malformed targets, unsafe authority ids, zero shares, negative values, decimal values, exponent notation, non-string amount fields, unknown roles, and non-canonical receiver shapes reject.

### Group Formation

```json
{
  "op": "perp-open-group",
  "pid": "perp-main",
  "ph": "<policy hash>",
  "pair": {
    "base": { "ns": "tap", "tick": "megazero", "dec": "0" },
    "quote": { "ns": "eip155", "cid": "eip155:1", "ak": "native", "aid": "native", "dec": "18" },
    "price_dir": "quote-per-base"
  },
  "coll": {
    "asset": { "ns": "tap", "tick": "megazero", "dec": "0" },
    "mode": "tap-account",
    "min": "1",
    "max": "1000000"
  },
  "form": {
    "start": "900000",
    "deadline": "900144",
    "early": true
  },
  "ready": {
    "min_long_coll": "1",
    "min_short_coll": "1",
    "min_total_coll": "2",
    "min_long_not": "1",
    "min_short_not": "1",
    "ratio_min": { "n": "0", "d": "1" },
    "ratio_max": { "n": "999999999", "d": "1" },
    "max_imbalance_not": "1000000"
  },
  "lev": {
    "min": { "n": "1", "d": "1" },
    "max": { "n": "200", "d": "1" },
    "step": { "n": "1", "d": "1" }
  },
  "close": { "full": true, "partial": true, "payout": "reserved-until-settlement", "min_remaining_not": "0" },
  "liq": { "rule": "isolated-maintenance-margin-v1", "mmr": { "n": "5", "d": "1000" }, "fee_bps": "0" },
  "settle": { "expiry": "901000", "rule": "expiry-price-v1", "fallback": "last-valid-at-expiry-v1" },
  "def": { "rule": "pro-rata-positive-equity-v1", "dust": "largest-remainder-v1" },
  "fee": {
    "rule": "position-reserve-bps-v1",
    "bps": "250",
    "receivers": [{ "tt": "a", "to": "bc1p...", "share": "10000", "rl": "pf" }]
  },
  "bounty": { "rule": "operator-policy-bounty-v1", "liquidate": "policy-default" },
  "oracle": { "rule": "spot-vwap-v1", "source": "spot", "max_age": "144" },
  "entry": { "mode": "one-sided-v1", "required": true, "allow_unbounded": false, "max_slippage_bps": "500" }
}
```

The group id is derived from inscription id plus action index. It is not read from the payload.

Group creation records policy id, policy hash, group terms hash, pair assets, collateral asset, formation bounds, expiry, readiness thresholds, leverage bounds, maintenance margin, fee rule, fee receivers, liquidation bounty rule, oracle rule, entry-bound policy, empty aggregate entry bounds, and state `formation`.

The group fee rule must be an exact copy of the referenced policy fee rule. It must not reduce fees, increase fees, reorder receivers, drop receivers, add receivers, change receiver targets, change receiver roles, change shares, or replace the rule name. A group whose fee rule differs from the policy fee rule rejects even if the difference would reduce the fee.

Collateral mode is explicit:

- `tap-account` collateral debits and credits TAP balances inside the protocol.
- External collateral modes are not valid TAP perp collateral modes.
- `coll.surface` is invalid for TAP perp groups.
- Direct `perp-join` is valid only for TAP collateral groups.
- External quote metadata can appear in the pair or oracle terms, but it does not imply external custody, external settlement, or external claim/refund paths.

Reader-visible pair indexes use encoded asset keys, not raw asset labels.

- TAP asset key: `tap:` plus lowercase UTF-8 ticker bytes encoded as lowercase hex.
- External asset key: `ext:` plus lowercase hex components for `ns`, `cid`, `ak`, and `aid`, joined by `:`.
- Pair key: `<base asset key>|<quote asset key>`.
- Example: TAP/TAP uses `tap:746170|tap:746170`.

Raw tickers, symbols, chain ids, and external asset ids are not written directly into slash-delimited pair index keys.

```json
{
  "op": "perp-join",
  "gid": "<group id>",
  "src": { "tt": "a", "to": "bc1p..." },
  "side": "long",
  "coll": "100",
  "lev": { "n": "10", "d": "1" },
  "entry": { "max": { "p": "109", "q": "10000" } },
  "claim": { "tt": "a", "to": "bc1p..." },
  "refund": { "tt": "a", "to": "bc1p..." }
}
```

For `tap-account` collateral, join debits the caller's available collateral and credits the group authority. Same-redeem pending debits are included before acceptance. `claim` and `refund` are immutable payout targets for that position.

Entry-bound rules:

- `entry.mode` is stored on the group from policy-controlled terms.
- `one-sided-v1` accepts `entry.max` for long positions and `entry.min` for short positions.
- Ratio fields are `{ "p": "<unsigned integer string>", "q": "<positive unsigned integer string>" }`.
- Long activation price must be less than or equal to the strictest stored long maximum.
- Short activation price must be greater than or equal to the strictest stored short minimum.
- Comparisons use exact cross multiplication. Implementations must not divide, round, or use floating point.
- A required entry policy rejects missing bounds.
- Unsupported side/bound combinations reject.
- A valid join stores the bound on the position and updates group aggregate bounds in O(1).
- Aggregate bounds are reader-visible group state.
- Activation checks aggregate bounds in O(1). It must not scan positions.
- If activation price violates aggregate bounds, activation rejects without consuming certificate replay state and without mutating group state.
- A group that cannot activate because of entry bounds remains cancellable after formation deadline and positions remain refundable through `perp-refund`.

If a group is past the deadline and has not activated, `perp-cancel` moves it to `cancelled`. This remains valid even if the group is ready, because no participant should be stranded when activation was not submitted. Only then can a participant use `perp-refund`. Refund pays the immutable refund target. If `to` is present, it must equal the stored target.

### Price Certificates

Price-bearing actions carry `cert`. The certified price is inside `cert.price`.

```json
{
  "op": "perp-settle",
  "gid": "<group id>",
  "cert": {
    "v": "1",
    "dom": "tap-perp-price-v1",
    "net": "bitcoin:mainnet",
    "pid": "perp-main",
    "ph": "<policy hash>",
    "gid": "<group id>",
    "gh": "<group terms hash>",
    "purpose": "settlement",
    "seq": "1",
    "valid_from": "900900",
    "valid_until": "901144",
    "source": { "rule": "spot-vwap-v1", "from": "900800", "to": "900900", "trades": "1", "volume": "1" },
    "pair": {
      "base": { "ns": "tap", "tick": "megazero", "dec": "0" },
      "quote": { "ns": "eip155", "cid": "eip155:1", "ak": "native", "aid": "native", "dec": "18" },
      "price_dir": "quote-per-base"
    },
    "price": { "p": "109", "q": "10000", "seq": "1" },
    "state_hash": "<action hash without cert>",
    "salt": "settlement-1",
    "sigs": [
      { "signer": "02...", "hash": "<message hash>", "sig": { "v": "0", "r": "...", "s": "..." } }
    ]
  }
}
```

`perp-settle` may use the committed no-new-signature fallback after `expiry + oracle.max_age`:

```json
{
  "op": "perp-settle",
  "gid": "<group id>",
  "fallback": "last-valid-at-expiry-v1"
}
```

The fallback uses the last price already accepted into the group state. Accepted group prices are entry, close, liquidation, and signed settlement prices. Display prices, API-only marks, pending certificates, and backend rows are not group state.

Certificate rules:

- `purpose` must match the action family.
- `entry` is used by `perp-activate`.
- `close` is used by `perp-close`.
- `liquidation` is used by `perp-liquidate`.
- `settlement` is used by `perp-settle`.
- `state_hash` is the canonical hash of the action without `cert`.
- The certificate payload hash is the canonical hash of `cert` without `sigs`.
- If `pair` is present in the certificate, it must normalize to the stored group base and quote assets and `price_dir` must be `quote-per-base`.
- Certificate sequence is monotonic per policy, group, and purpose. A certificate must use a sequence greater than the latest accepted sequence for that purpose.
- The certificate signer threshold is read from the stored policy signer set and `thr`.
- `valid_from` and `valid_until` are inclusive block bounds.
- Failed certificates do not consume the sequence.
- Fallback settlement consumes a deterministic fallback marker and does not create a price-certificate history entry.

For non-TAP collateral enforcement, a settlement certificate must also bind the exact chain-local settlement payload. At minimum the bound payload must include group id, operator fee amount, pool fee amount, claim pool, total equity, aggregate settlement terms, and the chain-local claim formula accepted by that chain. A valid certificate for one conserved settlement set must not be reusable with another conserved settlement set.

The certificate signature message is:

```text
sha256(canonical_json(["tap-perp-price-v1", "tap", policy_id, policy_hash, purpose, group_id, cert_payload_hash, seq, valid_until]))
```

### Active And Terminal State

`perp-activate` requires a ready formation group and a valid certificate. The certificate price must satisfy the stored aggregate entry bounds before any mutation. It stores the entry price and moves the group to active state. Position activity is derived from the position and group state. Original-collateral refund is no longer available after activation.

`perp-close` requires the position owner. It computes realized equity for the closed collateral at the certified price and reserves that equity until terminal settlement. It does not create an immediate claimable payout.

`perp-liquidate` closes a position when equity falls below the maintenance threshold. Liquidation records remaining equity and pays exactly the position's open reserved liquidation bounty when the state transition succeeds. The bounty is reserved at position open, is not capped by current equity, and cannot be consumed by PnL or bad debt.

`perp-settle` is valid at or after expiry. It uses either a valid settlement certificate or, after `expiry + oracle.max_age`, the committed `last-valid-at-expiry-v1` fallback. It computes terminal group totals from stored aggregates, moves open unused liquidation bounty reserve into fee accounting, applies reserve-based fee distribution, and records an immutable claim formula. It does not scan positions, does not write per-position payout rows, and does not pay a settlement bounty.

For external collateral groups, settlement records the same terminal accounting in protocol state but does not mint or release TAP balances. Chain-local settlement, fee payment, claim, and refund are enforced by the committed settlement surface. The protocol record must still conserve the external collateral amount recorded by accepted evidence.

Per-position reserve rule:

```text
total_fee_reserve = floor(gross_collateral * fee_bps / 10000)
liquidation_bounty_reserve = floor(gross_collateral * liquidation_bounty_bps / 10000)
marketplace_fee_reserve = total_fee_reserve - liquidation_bounty_reserve
margin_collateral = gross_collateral - total_fee_reserve
```

`gross_collateral`, `total_fee_reserve`, `liquidation_bounty_reserve`, `marketplace_fee_reserve`, and `margin_collateral` are fixed for the position when it opens. `margin_collateral` is the collateral used for notional, equity, maintenance, close, liquidation, and settlement PnL. Formation cancellation refunds `gross_collateral`.

Settlement payout rule:

```text
settlement_balance = group authority balance
unused_bounty_fee = open_liquidation_bounty_reserve
fee = marketplace_fee_reserve_total + unused_bounty_fee
claim_pool = settlement_balance - fee
claim_basis_total = total_equity if total_equity > 0 else total_margin_collateral
claim_basis_remaining = claim_basis_total
claim_pool_remaining = claim_pool
```

Liquidation bounty availability is not a terminal settlement validity condition.

Claim payout rule:

```text
position_basis = position_equity if total_equity > 0 else position_original_collateral

if position_basis == claim_basis_remaining:
  payout = claim_pool_remaining
else:
  payout = floor(position_basis * claim_pool_remaining / claim_basis_remaining)

claim_basis_remaining -= position_basis
claim_pool_remaining -= payout
```

The last claimant for the remaining basis receives the remaining claim pool. `claimed_total + fee + bounty` never exceeds the group authority balance, and no residual collateral remains assigned to nobody.

Settlement fee split rule:

```text
receiver_fee_i = floor(fee * receiver_share_i / 10000)
fee_dust = fee - sum(receiver_fee_i)
```

`fee_dust` is assigned one atomic unit at a time to receivers sorted by largest remainder, with receiver list order as the deterministic tie breaker. Receiver-level fee amounts sum exactly to `fee`.

Account fee receivers credit the receiver account. Staking reward receivers credit the target staking authority reward accumulator for the group collateral ticker using the same reward accounting as ordinary staking reward allocations. A settlement that cannot credit an `sr` receiver rejects. Refund, cancel, and unactivated exit paths do not allocate settlement fees.

Terminal settlement state records total equity, claim pool, claimed total, total settlement fee, receiver-level fee amounts, residual, default flag, open-side aggregate equity, closed equity, and deterministic dust handling. Duplicate settlement rejects.

If a block containing a settlement is removed by chain reorganization, every balance debit, balance credit, receiver fee credit, staking reward allocation, settlement aggregate, claimed-total update, position status update, and terminal group status created by that settlement is removed with it. Re-applying the same surviving inscription after the reorg must produce the same state as first application.

`perp-claim` pays the immutable claim target after settlement or default. The claim amount is computed from the position record and terminal settlement formula, then added to group `claimed_total`. The executor does not choose the payout target. If `to` is present, it must equal the stored claim target. Duplicate claims reject.

## Staking Authority

Staking is implemented as authority configuration plus stake, reward claim, and unstake actions. It is not a lock kind.

Create a staking authority:

```json
{
  "op": "auth-cfg",
  "k": "stk",
  "n": "Example staking",
  "stk": "tap",
  "rt": [],
  "ctl": {
    "ty": "ta",
    "auth": "<controller authority id>"
  },
  "r": {
    "cm": "arps",
    "rnd": "flr",
    "aw": false,
    "ep": "carry",
    "ud": 0,
    "tr": [
      {
        "id": "3m",
        "dur": "12960",
        "w": "1"
      },
      {
        "id": "6m",
        "dur": "25920",
        "w": "2"
      },
      {
        "id": "12m",
        "dur": "52560",
        "w": "4"
      }
    ]
  }
}
```

Fields:

| Field | Meaning |
| --- | --- |
| `k` | Authority kind. `stk` means staking. |
| `stk` | Token being staked. |
| `rt` | Reward tickers. Empty array means rewards can be any token held by the authority. |
| `ctl` | Controller. Staking authorities use `{ "ty": "ta", "auth": "<authority id>" }`. |
| `r.cm` | Reward accounting mode. Allowed value is `arps`, accumulated reward per share. |
| `r.rnd` | Rounding mode. Allowed value is `flr`, floor. |
| `r.aw` | Auto withdraw. Allowed value is `false`. |
| `r.ep` | Empty pool policy. Values are `reject`, `hold`, or `carry`. |
| `r.tr` | Tiers. Each tier has `id`, block duration `dur`, and weight `w`. |
| `r.ud` | Optional update delay. The default is `0`. |

Tier ids are labels. `dur` is the exact number of TAP blocks added to the staking block to derive the unlock height.

Empty pool policies:

| Policy | Behavior |
| --- | --- |
| `reject` | Reward allocation to a staking authority with no shares is invalid. |
| `hold` | Reward allocation with no shares is accepted and kept in the authority balance without distribution. |
| `carry` | Reward allocation with no shares is carried and distributed when shares exist. Dust that cannot yet be distributed is also carried. |

Stake:

```json
{
  "op": "stake",
  "auth": "<staking authority id>",
  "tick": "tap",
  "amt": "100",
  "tier": "12m",
  "claim": "bc1p..."
}
```

Claim rewards:

```json
{
  "op": "claim-rwd",
  "auth": "<staking authority id>",
  "pos": "<stake position id>",
  "rt": "tap"
}
```

Reward claims pay the stake position's stored claim target. The executor cannot redirect the payout, cannot claim future accrual, and duplicate claims for the same accrued amount reject.

Unstake:

```json
{
  "op": "unstake",
  "auth": "<staking authority id>",
  "pos": "<stake position id>",
  "rt": "tap"
}
```

Rules:

- A stake position is created from the staking inscription id and action index.
- The staker spends available balance.
- Shares are `staked atomic amount * tier weight`.
- Each position has its own reward debt and unlock height.
- Rewards can be claimed while the position is open.
- Unstake is only valid at or after the position unlock height.
- If `unstake.rt` is present, the protocol claims that reward ticker before returning the stake when a claimable amount exists.

Staking can support fee sharing, reward programs, vote escrow style weights, and loyalty systems. It records deposits and reward accounting without custody by an external process.

## Sale Authority

Sale authorities support token launch, capped sale, refund, and treasury flows.

Create a sale authority:

```json
{
  "op": "auth-cfg",
  "k": "sale",
  "n": "Example sale",
  "st": "newtoken",
  "pt": "tap",
  "ctl": {
    "ty": "ta",
    "auth": "<controller authority id>"
  },
  "tre": {
    "tt": "a",
    "to": "bc1p..."
  },
  "s": {
    "sh": "950000",
    "eh": "960000",
    "hc": "100000",
    "sc": "1000",
    "mn": "10",
    "mx": "1000",
    "r": {
      "cm": "fix",
      "pa": "1",
      "sa": "100",
      "rnd": "flr"
    },
    "ov": "reject",
    "cx": true,
    "alw": {
      "ty": "sha256-merkle",
      "lf": "addr",
      "root": "<sha256 merkle root>"
    }
  }
}
```

Fields:

| Field | Meaning |
| --- | --- |
| `k` | Authority kind. `sale` means sale. |
| `st` | Sale token distributed to contributors. |
| `pt` | Payment token paid by contributors. |
| `ctl` | Controller authority. |
| `tre` | Treasury target for payment tokens. Target uses `tt` and `to`. |
| `s.sh` | Start block height. |
| `s.eh` | End block height. |
| `s.hc` | Hard cap in payment token units. |
| `s.sc` | Optional soft cap in payment token units. |
| `s.mn` | Optional minimum contribution. |
| `s.mx` | Optional maximum contribution per claim address. |
| `s.r` | Fixed exchange rate. `cm` must be `fix`; `rnd` must be `flr`; `pa` is the payment amount unit; `sa` is the sale token amount unit. |
| `s.ov` | Overflow policy. Allowed value is `reject`. |
| `s.cx` | If true, controller can cancel the sale. |
| `s.alw` | Optional SHA-256 Merkle allowlist. `lf` can be `addr` or `addr-cap`. |

Sale actions:

```json
{ "op": "fund-sale", "auth": "<sale authority id>", "tick": "newtoken", "amt": "100000" }
```

```json
{ "op": "contribute", "auth": "<sale authority id>", "tick": "tap", "amt": "50", "claim": "bc1p..." }
```

```json
{ "op": "finalize-sale", "auth": "<sale authority id>" }
```

```json
{ "op": "resolve-sale", "auth": "<sale authority id>" }
```

```json
{ "op": "claim-sale", "auth": "<sale authority id>", "cid": "<contribution id>" }
```

```json
{ "op": "refund-sale", "auth": "<sale authority id>", "cid": "<contribution id>" }
```

```json
{ "op": "cancel-sale", "auth": "<sale authority id>" }
```

```json
{
  "op": "withdraw-sale",
  "auth": "<sale authority id>",
  "tick": "newtoken",
  "amt": "1000",
  "tt": "a",
  "to": "bc1p..."
}
```

Rules:

- `fund-sale` deposits sale inventory into the sale authority.
- `contribute` deposits payment tokens and records the contributor claim address.
- Contributions are accepted only during the configured block window.
- Contributions must satisfy caps, min and max values, and allowlist rules.
- A contribution allocates `floor(payment_amount * s.r.sa / s.r.pa)` sale token units.
- If the sale has an allowlist, each contribution must include a valid Merkle proof. `addr-cap` allowlists bind the claim address and maximum payment amount.
- `finalize-sale` is valid at or before the end height only when the hard cap is reached. After the end height, it is valid when the soft cap is reached. It requires the controller authority.
- Finalization sends payment tokens to the treasury target when payment balance, inventory balance, allocation totals, and target credit are all valid.
- `resolve-sale` is public. It finalizes under the same finalization invariants, or marks the sale failed/refundable when deterministic failure conditions are met.
- Deterministic failure reasons include missed soft cap, inventory underfunding, payment underfunding, and unavailable treasury target.
- `resolve-sale` writes one terminal resolution record. Duplicate or conflicting sale terminal actions reject atomically.
- `claim-sale` lets a contributor claim allocated sale tokens after finalization.
- `refund-sale` lets a contributor recover payment tokens if the sale is cancelled, failed/refundable, or fails its soft cap after the end height.
- `claim-sale` and `refund-sale` pay the contribution's stored claim address. The executor cannot redirect the payout.
- `cancel-sale` is only valid when cancellation is enabled by `s.cx`.
- `withdraw-sale` lets the controller withdraw remaining sale tokens after finalization, cancellation, or failed end.

Sale authorities can support launchpads, token presales, fixed price allocations, allowlisted mints, and refund based funding rounds.

## AMM Authority

An AMM authority holds pool reserves and records LP shares. AMM pools use constant product accounting for TAP to TAP add liquidity, remove liquidity, and swap actions. TAP/external pools can use external reserve snapshots plus conditional obligations for the TAP-side settlement leg. Snapshots are policy records only. They do not move external assets and they do not prove external consensus.

Create a TAP/TAP AMM authority:

```json
{
  "op": "auth-cfg",
  "k": "amm",
  "n": "TAP DMT pool",
  "a": [
    { "ty": "tap", "tick": "tap" },
    { "ty": "tap", "tick": "dmt" }
  ],
  "c": {
    "ty": "cpmm",
    "fee": "30",
    "pf": "0",
    "min": "1000",
    "pause": false
  },
  "ctl": {
    "ty": "ta",
    "auth": "<controller authority id>"
  }
}
```

AMM config fields:

| Field | Meaning |
| --- | --- |
| `k` | Authority kind. AMM uses `amm`. |
| `a` | Exactly two assets. Add liquidity, remove liquidity, and swap require both assets to be TAP assets. Conditional AMM obligations can use the TAP side of a TAP/external pool. |
| `a[].ty` | `tap` for protocol enforced TAP reserves, `ext` for attested external reserve metadata. |
| `a[].tick` | TAP ticker when `ty` is `tap`. Tickers are normalized with the same lowercase rules as the rest of TAP. |
| `a[].ns` | External namespace when `ty` is `ext`, for example `eip155`, `solana`, or `bitcoin`. |
| `a[].cid` | External chain id when `ty` is `ext`. |
| `a[].aid` | External asset id when `ty` is `ext`, for example `native`, a contract address, or a mint address. |
| `a[].dec` | External asset decimals as an integer string. |
| `a[].pool` | Optional external pool or contract id for external snapshots. |
| `att` | Required attestor policy when either asset has `ty: "ext"`. Invalid when both assets are TAP assets. |
| `att.thr` | Required attestor threshold. It must be at least `2` and cannot exceed signer count or `8`. |
| `att.signers` | Unique secp256k1 attestor public keys, normalized to compressed keys. |
| `att.max_age` | Maximum accepted external snapshot age in Bitcoin block intervals. One interval is treated as 600 seconds. |
| `att.reorg` | Declared external-chain reorg tolerance. It is an unsigned height-sized integer string. |
| `c.ty` | Curve type. Supported value is `cpmm`. |
| `c.fee` | Swap fee in basis points. Maximum is `1000`, meaning 10 percent. |
| `c.pf` | Protocol fee share in basis points of the swap fee. `0` disables protocol fee routing. |
| `c.pp` | Protocol fee target. Required only when `c.pf` is greater than `0`. |
| `c.min` | Minimum initial LP shares withheld from the first liquidity provider. Withheld shares remain unowned and unremovable. |
| `c.pause` | Blocks new adds and swaps when true. Removes remain possible. |
| `ctl` | Controller target. AMM authorities use a token authority controller. |

Validation rules:

- `a` must contain exactly two distinct normalized assets.
- TAP assets must already be deployed.
- External assets must have non-empty namespace, chain id, asset id, and decimals not greater than `38`.
- External pools must include `att`; TAP/TAP pools must not include `att`.
- `c.fee`, `c.pf`, and `c.min` must be integer strings. JSON numbers, decimals, exponent notation, negative values, comma strings, and malformed values are invalid.
- `c.fee` must not exceed `1000`.
- `c.pf` must not exceed `10000`.
- `c.pp` is required when `c.pf` is greater than `0` and invalid when `c.pf` is `0`.
- `c.pause` must be a boolean.
- A cancelled controller cannot create new AMM pools, external snapshots, liquidity additions, swaps, or AMM-sourced obligations. LP removal remains available for existing LP positions.

### Add Liquidity

```json
{
  "op": "add-liq",
  "auth": "<amm authority id>",
  "amts": ["100000000", "50000000"],
  "min": "70000000",
  "to": { "tt": "a", "to": "bc1p..." },
  "exp": "960000",
  "ref": "client-order-id"
}
```

Rules:

- Both pool assets must be TAP assets.
- The signer spends both amounts from available account balance.
- The AMM authority receives both reserve amounts.
- LP shares are internal records. They are not TAP transferable inscriptions.
- First liquidity mints `integer_sqrt(amount0 * amount1) - c.min` shares to `to`.
- `c.min` shares are added to total shares without an owner and cannot be removed.
- Later liquidity mints `min(floor(amount0 * total_shares / reserve0), floor(amount1 * total_shares / reserve1))`.
- Imbalanced deposits are allowed only if minted shares are at least `min`. Any imbalance benefits existing LPs.
- `exp` is required. The action is invalid after that block height.
- `ref` is optional and prevents duplicate execution for the same authorizer and pool.

### Remove Liquidity

```json
{
  "op": "rm-liq",
  "auth": "<amm authority id>",
  "sh": "10000000",
  "min": ["9900000", "4900000"],
  "to": { "tt": "a", "to": "bc1p..." },
  "exp": "960000",
  "ref": "client-order-id"
}
```

Rules:

- The signer burns account owned LP shares unless `own` points to an authority owned position controlled by the signer authority.
- Output amounts are `floor(shares * reserve / total_shares)` for each side.
- Each output must be at least the matching `min` value.
- Remove is allowed while the pool is paused.
- Remove is allowed after controller cancellation because it exits an existing LP position.
- Pool close is not an action. Remaining dust stays in the pool.

### Swap

Exact-in swap:

```json
{
  "op": "swap",
  "auth": "<amm authority id>",
  "m": "xin",
  "i": 0,
  "amt": "10000000",
  "min": "9800000",
  "to": { "tt": "a", "to": "bc1p..." },
  "exp": "960000",
  "ref": "client-order-id"
}
```

Exact-out swap:

```json
{
  "op": "swap",
  "auth": "<amm authority id>",
  "m": "xout",
  "i": 0,
  "out": "10000000",
  "max": "10200000",
  "to": { "tt": "a", "to": "bc1p..." },
  "exp": "960000",
  "ref": "client-order-id"
}
```

Swap fields:

| Field | Meaning |
| --- | --- |
| `m` | `xin` for exact input, `xout` for exact output. |
| `i` | Input side, `0` or `1`. |
| `amt` | Input amount for `xin`. |
| `min` | Minimum output for `xin`. |
| `out` | Output amount for `xout`. |
| `max` | Maximum input for `xout`. |
| `to` | Output target. |
| `exp` | Last valid block height. |
| `ref` | Optional idempotency reference. |

Exact-in formula:

```text
gross_fee = floor(amount_in * fee_bps / 10000)
protocol_fee = floor(gross_fee * protocol_fee_share_bps / 10000)
amount_in_after_fee = amount_in - gross_fee
amount_out = floor(amount_in_after_fee * reserve_out / (reserve_in + amount_in_after_fee))
```

Exact-out formula:

```text
amount_in_after_fee = ceil(reserve_in * amount_out / (reserve_out - amount_out))
amount_in = ceil(amount_in_after_fee * 10000 / (10000 - fee_bps))
gross_fee = floor(amount_in * fee_bps / 10000)
protocol_fee = floor(gross_fee * protocol_fee_share_bps / 10000)
```

Fee rules:

- The LP fee stays in pool reserves and increases LP value.
- If `c.pf` is greater than `0`, the protocol fee is credited to `c.pp`.
- Fee routing can target an account, an authority, or the burn target.
- All math is integer math in atomic units. Floor rounding is used unless a formula states `ceil`.

Slippage and ordering rules:

- Exact-in swaps require `min`.
- Exact-out swaps require `max`.
- All swaps require `exp`.
- Same-redeem actions are evaluated in order.
- Same-block ordering follows inscription processing order.
- If any action in a redeem fails, no AMM state from that redeem is written.

### External Reserve Snapshots

External reserve snapshots publish signed liquidity data for a TAP/external pool. They are not a light client and cannot move funds.

AMM config with an external leg:

```json
{
  "op": "auth-cfg",
  "k": "amm",
  "a": [
    { "ty": "tap", "tick": "tap" },
    {
      "ty": "ext",
      "ns": "eip155",
      "cid": "1",
      "aid": "native",
      "dec": "18",
      "pool": "0x..."
    }
  ],
  "att": {
    "thr": 2,
    "signers": [
      "02...",
      "03..."
    ],
    "max_age": "24",
    "reorg": "12"
  },
  "c": {
    "ty": "cpmm",
    "fee": "30",
    "pf": "0",
    "min": "1000",
    "pause": false
  },
  "ctl": {
    "ty": "ta",
    "auth": "<controller authority id>"
  }
}
```

Snapshot action:

```json
{
  "op": "sync-ext",
  "auth": "<amm authority id>",
  "sid": "snapshot-1",
  "ext": {
    "ns": "eip155",
    "cid": "1",
    "pool": "0x...",
    "aid": "native",
    "res": "250000000000000000000",
    "h": "12345678",
    "ts": "1710000000"
  },
  "exp": "960000",
  "sigs": [
    {
      "hash": "...",
      "sig": {
        "v": "0",
        "r": "...",
        "s": "..."
      }
    }
  ],
  "salt": "..."
}
```

Signed message input:

```text
[
  "tap-amm-external-liquidity-v1",
  auth,
  sid,
  ext.ns,
  ext.cid,
  ext.pool,
  ext.aid,
  ext.res,
  ext.h,
  ext.ts,
  exp
]
```

The signature digest is SHA-256 over the JSON stringified array plus `salt`.

External snapshot rules:

- External pools require at least 2 attestor signers.
- Threshold cannot exceed signer count or `8`.
- Signers must be unique secp256k1 public keys and are normalized to compressed keys.
- Snapshot id `sid` is one time per AMM authority.
- The snapshot binds authority id, external namespace, chain id, pool id, asset id, reserve, external height or slot, timestamp, expiry, and salt.
- A snapshot can update AMM policy data, but it cannot debit or credit balances by itself.
- Cross-chain AMM settlement uses `sync-ext` as signed policy input and conditional obligations for the TAP-side settlement leg. External contracts or programs still move the external asset.
- New TAP-side obligations against a TAP/external pool must bind `ctx.amm.pid`, `ctx.amm.i`, `ctx.amm.sid`, `ctx.amm.set`, and `ctx.amm.h`.
- `ctx.amm.sid` must reference an accepted snapshot for the same AMM authority. The snapshot must match the external leg on the other side of `ctx.amm.i`.
- `ctx.amm.h` must equal the obligation hashlock. This binds the TAP settlement path to the same preimage as the external settlement path.
- `ctx.amm.set` identifies the external settlement reference. TAP commits this value but does not interpret it.

### AMM Flow

TAP/TAP pool lifecycle:

```text
controller authority
        |
        v
auth-cfg k=amm
        |
        v
AMM authority record
        |
        +-- add-liq --> authority reserve balances + LP position
        |
        +-- swap ----> debit trader, update reserves, credit output target
        |
        +-- rm-liq --> burn LP shares, debit reserves, credit output target
```

External snapshot lifecycle:

```text
external pool
        |
        v
attestors observe reserve
        |
        v
sync-ext signed snapshot
        |
        v
TAP AMM policy record
        |
        v
quoted state can read it, but TAP still does not validate external state
```

Attested cross-chain AMM with TAP-side obligation:

```text
external settlement source
        |
        v
attestors sign reserve snapshot
        |
        | sync-ext
        v
AMM policy record
        |
        v
settlement context binds snapshot, chain, asset, amount, slippage, hashlock
        |
        | ob-open
        v
TAP-side obligation
        |
        +-- ob-claim or ob-final after matching external settlement reveals the preimage
        |
        +-- ob-refund if the external settlement never completes
```

Conditional obligations are the settlement primitive for AMM source and destination adapters. Each adapter must prove that it is allowed to move its own state.

### AMM Composition Patterns

| Pattern | Required actions | Critical invariants |
| --- | --- | --- |
| TAP token swap pool | `auth-cfg k:"amm"`, `add-liq`, `swap`, `rm-liq` | Constant product math, slippage bounds, expiry, LP share accounting, no partial writes. |
| Protocol owned liquidity | LP target `tt:"h"` on `add-liq` and controlled `rm-liq` | Authority owned LP positions, no reserve withdrawal outside valid remove logic. |
| Fee sharing pool | `swap` with `c.pf` and `c.pp` | Protocol fee target is fixed in pool config and uses atomic integer rounding. |
| Router | Multiple `swap` actions in one redeem | Ordered pending reserve accounting and atomic failure if a later hop misses slippage. |
| External quote board | `sync-ext` | Snapshot replay resistance and clear attestor trust boundary. |
| Attested cross-chain pool | `sync-ext`, `ob-open`, `ob-claim`, `ob-refund`, `ob-final` | Snapshot binding, hashlock settlement, slippage context, one-time obligation consumption, and no direct external consensus assumptions. |

## Authority Targets

Several actions use compact target records:

```json
{
  "tt": "a",
  "to": "bc1p..."
}
```

Target types:

| `tt` | Target |
| --- | --- |
| `a` | Bitcoin account address. |
| `h` | Token authority id. |
| `amm` | AMM pool side when the action explicitly supports AMM adapters. |
| `b` | Burn address. |

The Bitcoin burn address is `1BitcoinEaterAddressDontSendf59kuE`.

## Protocol Composition Patterns

| Pattern | Main actions and kinds | Required invariant support | Notes |
| --- | --- | --- | --- |
| Delegated HTLC settlement | `execute`, `lock` with `kind: "htlc"`, `claim`, `refund`, `cancel-delegation`, allocations with `rl: "of"` and `rl: "sr"` | Delegation signatures, nonce uniqueness, hashlock preimage claims, refund height, fee and staking reward allocations. | One authority signs a template once; a later filler supplies constrained values. |
| Direct atomic swap | `lock` with `kind: "htlc"`, `claim`, `refund` | Matching hashlocks, different refund windows, and one-time lock consumption. | Participants exchange lock data offchain. |
| HTLC liquidity bridge | `execute`, `lock` with `kind: "htlc"`, `claim`, `refund`, optional allocations | Same hash preimage on TAP and the external leg, conservative refund window ordering, one-time lock consumption, and delegated LP expiry. | Swap bridge pattern. Not wrapped-asset minting. |
| Attested settlement bridge | `sync-ext`, `ob-open`, `ob-claim`, `ob-refund`, `ob-final`, optional `lock` with `kind: "htlc"` | Attestor threshold policy, replay-resistant event references, hashlock settlement, one-time obligation consumption, and settlement-safe exits. | Redeem actions do not mint bridge supply and do not prove external consensus by themselves. |
| OTC settlement | `lock` with `kind: "otc"`, `claim`, `refund`, optional `execute` | Explicit counterparty reference in `data.cp`, hashlock or authority release, refund fallback. | Negotiated bilateral settlement with an indexed audit trail. |
| Vesting | `lock` with `kind: "vesting"`, `claim` | Height based release and required `data.dom` plus `data.ref`. | Scheduled release. |
| Cooldown | `lock` with `kind: "cooldown"`, `claim` | Claim target must be the authority owner and no refund path is allowed. | Protocol enforced waiting period before withdrawal. |
| Escrow | `lock` with `kind: "escrow"`, `claim`, `refund` | Authority controlled claim, payer and payee binding, refund fallback. | Mediated release with refund protection. |
| Staking rewards | `auth-cfg` with `k: "stk"`, `stake`, `claim-rwd`, `unstake`, allocations with `rl: "sr"` | Weighted tiers, per-position reward debt, reward tick rules, unlock height, empty pool policy. | Fee sharing, loyalty rewards, and long term holder programs. |
| Token sale | `auth-cfg` with `k: "sale"`, `fund-sale`, `contribute`, `finalize-sale`, `resolve-sale`, `claim-sale`, `refund-sale`, `cancel-sale`, `withdraw-sale` | Hard cap, soft cap, contribution bounds, Merkle allowlist, treasury target, public resolution, sale state transitions. | Fixed rate sale with refunds if the round fails. |
| Fee splitter | Allocations with target types `a`, `h`, and `b` | Allocation role uniqueness, target validation, refund returns all allocations to refund address. | Routes proceeds to accounts, reward authorities, treasuries, or burn. |
| Signed reward claim | Redeem `items` or `lock` with authority condition | Signature uniqueness, authority ownership, optional domain reference in `data`. | Offchain reward or credit authorization. |
| Account obligation escrow | `ob-open` from `src.tt:"a"`, then `ob-claim` or `ob-refund` | Source balance reservation, hashlock release, refund height, and one-time consumption. | Account funds reserved without a normal lock record. |
| Authority backed settlement | `ob-open` from `src.tt:"h"` | Active authority at open, saved policy, atomic target validation. | Authority balance committed to a future claim or refund while preserving cancellation safety. |
| Cross-chain AMM | `sync-ext`, `ob-open` with AMM source or destination, `ob-final`, `ob-refund` | Snapshot binding, AMM reserve math, hashlock, slippage context, no partial reserve writes. | TAP-side state for an attested external pool leg. |

## privilege-auth

`privilege-auth` lets a project or authority sign mint permissions and content hash verifications.

Create a privilege authority:

```json
{
  "p": "tap",
  "op": "privilege-auth",
  "sig": {
    "v": "0",
    "r": "86508516128453602592995353796555408942165629541645840597158531627249758208446",
    "s": "34207792571992691071321015990261453361621215423082720220623653945331263914520"
  },
  "hash": "a7a8fe6a46d7dd3aac3518ecdc97577be5319eb5ae8588eee9c92ad96eed3b68",
  "salt": "0.41631368860188056",
  "auth": {
    "name": "Example authority"
  }
}
```

Use a privilege authority in a deployment:

```json
{
  "p": "tap",
  "op": "token-deploy",
  "tick": "tap",
  "max": "21000000",
  "lim": "1000",
  "prv": "<privilege authority id>"
}
```

Use a privilege authority in a mint:

```json
{
  "p": "tap",
  "op": "token-mint",
  "tick": "tap",
  "amt": "1000",
  "prv": {
    "sig": {
      "v": "0",
      "r": "...",
      "s": "..."
    },
    "hash": "...",
    "address": "bc1p...",
    "salt": "..."
  }
}
```

Verify a content hash:

```json
{
  "p": "tap",
  "op": "privilege-auth",
  "sig": {
    "v": "0",
    "r": "...",
    "s": "..."
  },
  "hash": "...",
  "address": "bc1p...",
  "salt": "...",
  "prv": "<privilege authority id>",
  "verify": "<sha256 hex>",
  "col": "collection",
  "seq": 1
}
```

Rules:

- `name` visible length must be valid for the authority creation.
- Signed mints must target the address that appears in the signed payload.
- `verify` is a SHA-256 hash.
- `col` is required and can be an empty string.
- `seq` must be an unsigned integer, not a string.
- A privilege authority can be cancelled by tapping a `privilege-auth` inscription with `cancel`.

## BRC-20 Deployments

TAP indexes BRC-20 deployments from block `779832` to block `861575` when they carry the built in privilege authority id:

```text
c14d3de97cecc573d86592240ef38bf5ba298c8c2eaf68e17b99dbbeedbab7e4i0
```

Only deployments are indexed in this bridge path. BRC-20 mints, transfers, and other functions are not indexed as TAP operations.

Once the authority permits minting, the deployed BRC-20 ticker can be used in TAP internal and external flows.

## Bitmap

TAP indexers track Bitmap inscriptions under Bitmap rules. Cursed support is not part of the TAP Bitmap path.

## DMT Tokens

TAP supports DMT elements for field `4` block height, field `10` nonce, and field `11` bits, following the DMT specification at https://digital-matter-theory.gitbook.io/digital-matter-theory/introduction/digital-matter-theory. The supported operations are `dmt-deploy` and `dmt-mint`.

From Bitcoin block `861576`, Blockdrops for DMT UNATs and Bitmaps are supported.

## DMT NAT Miner Rewards

From block `885588`, regular `dmt-nat` mints are ignored. Instead, DMT NAT rewards are credited from coinbase transaction outputs.

Rules:

- Each block distributes DMT NAT according to the bits value of that block.
- Reward shares follow the BTC value of coinbase outputs.
- The reward denominator includes every coinbase output, including non-addressable outputs and `OP_RETURN` outputs.
- Outputs that cannot be credited to a valid address are treated as burned.
- From block `941848`, miner reward addresses have `token-transfer` disabled by default.
- Miner reward addresses must use `unblock-transferables` if they want to receive future transferables.
- `token-send` and `token-trade` stay disabled for miner reward addresses after the shield.
- `token-auth` redeem remains available.

## Indexer Requirements

Indexers must produce the same accepted and rejected protocol state for the same ordered inscription stream.

Required behavior:

- Protocol changes must be activation height gated.
- Numeric protocol values must follow the amount and stringify rules in this document.
- Tickers must be normalized consistently.
- Actions must reject invalid shapes before applying state.
- Authority cancellation must be enforced as retirement for new exposure-creating actions, not revocation of existing settlement rights.
- Locks, authority balances, transfer rows, stake positions, sale records, AMM pools, LP positions, obligations, and delegation cancellations must remain internally consistent.
- Conformance coverage must include happy paths, invalid shapes, duplicate consumption, cancelled authorities, insufficient available balances, decimals, stringified numbers, replay attempts, malformed JSON values, and attempts to bypass delegation, certified control, lock, obligation, sale, staking, or AMM constraints.

The protocol is intentionally sparse in record keys because high volume indexes can hold millions of rows. Index fields remain compact and avoid duplicate data unless it is needed for efficient reads.
