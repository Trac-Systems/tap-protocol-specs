# TAP Protocol

TAP is an Ordinals metaprotocol for fungible tokens and token backed application logic on Bitcoin L1.

The base token model follows the inscription based flow familiar from BRC-20. TAP extends that model with account level actions, signed authorities, internal transfers, locks, delegated execution, staking pools, and sale authorities.

The protocol has two layers:

| Layer | Purpose |
| --- | --- |
| External token layer | Deploy, mint, and inscribe transferable token balances. This layer is compatible with the usual BRC-20 style marketplace and wallet model. |
| Internal action layer | Tap confirmed inscriptions that update account state without moving a transferable inscription. This layer powers mass sends, trades, authorities, locks, staking, sales, and privileged mints. |

An inscription is considered tapped when it is sent back to the account that owns or controls the action. Some signed authority redeems do not require tapping by the submitter because the authority signature is the authorization.

All ticker comparisons are case insensitive. Indexers store and compare TAP tickers in lowercase. Tickers are Unicode visible length strings. Before block `861576`, valid token tickers are 3 visible characters or 5 to 32 visible characters. From block `861576`, valid token tickers are 1 to 32 visible characters.

## Activation Heights

Mainnet activation heights used by the current upgrade implementation are:

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
| Token locks, delegated locks, staking, and sales | `999999999` pending mainnet activation review |
| Miner reward shield | `941848` |
| Miner reward transfer execution shield | `942002` |

Non-mainnet indexers in the upgrade implementation activate feature gates at height `0` unless configured otherwise. Mainnet activation heights are part of consensus for production indexers.

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

`dec` is optional for `token-deploy`. If omitted, it defaults to `18`. In the current reference behavior, an indexed decimal override must parse to an integer from `0` to `17`. Values outside the override range are not used and the default `18` remains. DMT deployments use `0` decimals.

`lim` is optional. If omitted, minting has no per mint limit beyond the remaining supply.

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
- Available balance is `balance - transferable - locked`.
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

`token-trade` is the original TAP token to token trade primitive. It is retained for direct TAP token trades.

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

`token-auth` lets an account create an authority and later execute signed redeems. It is the base primitive for applications that need offchain policy decisions with onchain settlement.

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
- Cancellation retires the authority for new obligations. It must not revoke settlement paths for obligations that already exist.
- After cancellation, new direct redeem items are invalid.
- After cancellation, new obligation creating actions are invalid. This includes `lock`, `execute`, `auth-cfg`, `stake`, `fund-sale`, and `contribute`.
- After cancellation, settlement and exit actions remain valid when their own conditions are met. This includes `claim`, `refund`, `claim-rwd`, `unstake`, `finalize-sale`, `claim-sale`, `refund-sale`, eligible `withdraw-sale`, and `cancel-delegation`.
- Existing locks, stake positions, sale contributions, sale inventories, and authority balances are not removed by cancellation.
- Existing consumed locks are not changed by cancellation.

This distinction is required for user safety. Once an authority has created a lock or accepted a user position, cancellation must not let the operator block claims, refunds, unstaking, reward claims, sale claims, or sale refunds.

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

Redeems can carry an `actions` array instead of, or alongside, `items`. Actions are authority signed operations that can lock tokens, release locks, configure authority pools, manage staking positions, and run sale flows.

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
- Available balance is `balance - transferable - locked`.
- Lock creation records the full committed amount.
- Claiming a lock sends the main amount to `claim` and sends allocations to their targets.
- Refunding a lock sends the full committed amount, including allocations, to `refund`.
- An action redeem that spends a token must satisfy the authority ticker whitelist if the authority has one.
- Authority cancellation only blocks new obligations. It must not block settlement or exits for previously created obligations.

### Action Matrix

| `op` | Shape | Main invariant | What it enables |
| --- | --- | --- | --- |
| `lock` | `{ op, kind, tick, amt, claim, condition, refund?, refund_after?, data?, al? }` | Owner must have available balance for `amt + allocations`. Kind specific fields must match the invariant table below. | HTLC swaps, timed releases, escrow, OTC settlement, fee and reward routing. |
| `execute` | `{ op, delegation, fill, final? }` | Delegation must be signed by the authority threshold, unused, unexpired, and must materialize exactly one valid `lock`. | Maker signed offers, PSBT style partial authorization, taker final fill without maker returning online. |
| `cancel-delegation` | `{ op, auth?, nonce }` | Redeem authority must match the cancelled authority, and the nonce must be unused and uncancelled. | Cancels signed offers that were not executed. |
| `claim` | `{ op, lock, preimage? }` | Lock must exist, must not be consumed, and the condition must be satisfied before refund is available for hashlocks. | Releases locked tokens to the claim target. |
| `refund` | `{ op, lock }` | Lock must exist, must not be consumed, and current block must be at least `refund_after`. | Returns locked funds to the refund address. |
| `auth-cfg` with `k: "stk"` | `{ op, k, stk, rt, ctl, r, n? }` | Controller must be the current authority, stake token must exist, reward tokens must exist unless `rt` is empty, tiers must be unique and weighted. | Staking pools with weighted tiers and reward accounting. |
| `stake` | `{ op, auth, tick, amt, tier, claim }` | Staking authority must exist, `tick` must equal its stake token, tier must exist, staker must have available balance. | Opens an independent stake position. |
| `claim-rwd` | `{ op, auth, pos, rt }` | Position must be open, belong to `auth`, and have positive pending reward for `rt`. | Claims accrued staking rewards without unstaking. |
| `unstake` | `{ op, auth, pos, rt? }` | Position must be open, belong to `auth`, and current block must be at least its unlock height. | Returns staked principal after the lock duration. |
| `auth-cfg` with `k: "sale"` | `{ op, k, st, pt, ctl, tre, s, n? }` | Sale and payment tokens must exist, treasury target must be valid, height and cap rules must be coherent. | Launchpads, capped sales, allowlisted sales, refund based funding. |
| `fund-sale` | `{ op, auth, tick, amt }` | Sale authority must exist, `tick` must equal the sale token, controller must have available balance. | Deposits sale inventory. |
| `contribute` | `{ op, auth, tick, amt, claim, alw? }` | Sale must be open, `tick` must equal the payment token, caps and allowlist must pass, contributor must have available balance. | Records a buyer contribution and sale token allocation. |
| `finalize-sale` | `{ op, auth }` | Sale must not be cancelled or finalized. It can finalize early only at hard cap, or after end when soft cap is met. | Sends payment pool to treasury and unlocks sale claims. |
| `claim-sale` | `{ op, auth, cid }` | Contribution must be open, sale must be finalized, caller must be the contribution claim address. | Contributor claims allocated sale token. |
| `refund-sale` | `{ op, auth, cid }` | Contribution must be open. Sale must be cancelled, or ended without meeting soft cap. Caller must be the contribution claim address. | Contributor gets payment token back. |
| `cancel-sale` | `{ op, auth }` | Controller must match, sale must allow cancellation, and sale must not be finalized or already cancelled. | Cancels an open sale. |
| `withdraw-sale` | `{ op, auth, tick, amt, tt, to }` | Controller must match. Withdrawal is allowed after finalization, cancellation, or failed end. Target must be valid. | Returns unsold inventory or moves remaining sale token balance. |

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
    "dom": "market",
    "ref": "order-1"
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
| `data` | Optional application data. Some kinds require `data.dom` and `data.ref`. |
| `al` | Optional allocation list for fees, rewards, authority credits, or burn. |

The legacy `fee` object is accepted as a shorthand for an account allocation with role `of`. New applications should use `al`.

### Lock Kind Invariants

| Kind | Condition invariant | Claim invariant | Refund invariant | Data invariant | What apps can build |
| --- | --- | --- | --- | --- | --- |
| `htlc` | `condition.type` must be `hashlock`, and `condition.hash` must be SHA-256 hex. | `claim` receives `amt` only when the preimage hashes to `condition.hash` and claim happens before `refund_after`. | `refund` receives `amt + allocations` from `refund_after` onward. | No required `data` fields. | Cross chain atomic swaps, marketplace fills, escrow with preimage release, pay for secret flows. |
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

This is useful for marketplaces because the maker signs once, then the taker executes without requiring the maker to come back online.

Authority setup:

```text
authority wallet
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

Delegated execution with a maker and a taker:

```text
maker authority
  |
  | signs delegation:
  | auth, nonce, expiry, threshold,
  | signers, template, constraints
  v
signed delegation offer
  |
  | offchain discovery
  v
taker
  |
  | chooses fill values:
  | claim, refund, hash, refund_after
  | optionally gets finalizer signatures
  v
token-auth redeem with execute action
  |
  | inscribed by taker or any broadcaster
  v
indexer
  |
  | verifies authority, signatures, constraints,
  | nonce, expiry, whitelist, and available balance
  v
lock record
```

Delegation cancellation:

```text
authority wallet
  |
  | signs and taps normal redeem action
  | { op: "cancel-delegation", nonce }
  v
delegation cancel record
  |
  | same auth + nonce can no longer execute
  v
stale offchain offer is unusable
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
          "nonce": "order-1",
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
- Every action in a delegated-only redeem must be `execute`.
- The delegated authority is verified from `delegation.auth`.
- `nonce` must match `^[A-Za-z0-9._:-]{1,128}$`.
- A nonce can be executed once or cancelled once.
- `expiry` is a block height. Execution fails after that height.
- `signers` are compressed or uncompressed secp256k1 public keys. They are normalized to compressed keys.
- At least one signer must be the authority public key.
- `threshold` must be positive and cannot exceed 8 or the signer count.
- The template must produce a `lock` action after placeholders are filled.
- Constraints are checked against placeholders and direct paths.
- If a placeholder is not constrained to an exact value, finalizer signatures are required after final fill activation.

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

If `finalizers` is not present, the domain string is `tap-delegated-lock-v1` and the `finalizers` element is omitted.

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
| 1-of-1 | `threshold: 1`, one signer, authority key included | Single authority signs an executable offer. |
| 2-of-2 | `threshold: 2`, authority key and operator key | Authority and operator must both approve. Useful when an operator enforces fee or marketplace policy. |
| 2-of-3 or higher | Multiple signers, threshold below signer count | Redundant operator or committee approval without requiring every signer to be online. |

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

If `delegation.finalizers` is present, the final action must be signed by the configured finalizers. Finalizers use the same n-of-m threshold model as delegations, but they sign the filled action, not the original delegation template.

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

Final signatures protect takers and operators when the maker template allows values that are not exact at maker signing time.

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
  "nonce": "order-1"
}
```

Rules:

- The action must be in a normal authority redeem.
- The redeem authority must match `auth`.
- The nonce must not already be used or cancelled.
- Cancellation only affects unused delegated offers. It does not unlock funds because no lock exists before execution.

## Staking Authority

Staking is implemented as authority configuration plus stake, reward claim, and unstake actions. It is not a lock kind.

Create a staking authority:

```json
{
  "op": "auth-cfg",
  "k": "stk",
  "n": "Marketplace staking",
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
        "id": "12m",
        "dur": "51840",
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
| `ctl` | Controller. Current implementation uses `{ "ty": "ta", "auth": "<authority id>" }`. |
| `r.cm` | Reward accounting mode. Current value is `arps`, accumulated reward per share. |
| `r.rnd` | Rounding mode. Current value is `flr`, floor. |
| `r.aw` | Auto withdraw. Current value is `false`. |
| `r.ep` | Empty pool policy. Values are `reject`, `hold`, or `carry`. |
| `r.tr` | Tiers. Each tier has `id`, block duration `dur`, and weight `w`. |
| `r.ud` | Optional update delay. Current implementation stores it and defaults to `0`. |

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
- If `unstake.rt` is present, the implementation attempts to claim that reward ticker before returning the stake.

Staking can support fee sharing, reward programs, vote escrow style weights, and loyalty systems. It is suitable when an application needs protocol indexed deposits and reward accounting without custody by the application server.

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
      "sa": "100"
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
| `s.r` | Fixed exchange rate. `cm` must be `fix`, `rnd` is stored as `flr`. |
| `s.ov` | Overflow policy. Current value is `reject`. |
| `s.cx` | If true, controller can cancel the sale. |
| `s.alw` | Optional SHA-256 Merkle allowlist. |

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
- `finalize-sale` is valid before the end height only when the hard cap is reached. After the end height, it is valid when the soft cap is reached.
- Finalization sends payment tokens to the treasury target.
- `claim-sale` lets a contributor claim allocated sale tokens after finalization.
- `refund-sale` lets a contributor recover payment tokens if the sale is cancelled or fails its soft cap after the end height.
- `cancel-sale` is only valid when cancellation is enabled by `s.cx`.
- `withdraw-sale` lets the controller withdraw remaining sale tokens after finalization, cancellation, or failed end.

Sale authorities can support launchpads, token presales, fixed price allocations, allowlisted mints, and refund based funding rounds.

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
| `b` | Burn address. |

The Bitcoin burn address is `1BitcoinEaterAddressDontSendf59kuE`.

## Lock and Authority Index Records

Token locks are stored separately from transfer rows.

Lock record:

```json
{
  "id": "<lock id>",
  "owner": "bc1p...",
  "auth": "<authority id>",
  "kind": "htlc",
  "tick": "tap",
  "amt": "10",
  "remaining": "10",
  "claim": "bc1p...",
  "refund": "bc1p...",
  "condition": {
    "type": "hashlock",
    "hash": "<sha256 hex>"
  },
  "refund_after": 950000,
  "data": null,
  "blck": 940000,
  "tx": "<txid>",
  "vo": 0,
  "val": "546",
  "ins": "<inscription id>",
  "num": 123,
  "ts": 1710000000
}
```

Lock consume record:

```json
{
  "lock": "<lock id>",
  "action": "claim",
  "kind": "htlc",
  "owner": "bc1p...",
  "target": "bc1p...",
  "tick": "tap",
  "amt": "10",
  "blck": 940001,
  "tx": "<txid>",
  "vo": 0,
  "val": "546",
  "ins": "<inscription id>",
  "num": 124,
  "ts": 1710000600
}
```

If allocations exist, lock and consume records also include:

```json
{
  "al": [
    {
      "tt": "h",
      "to": "<authority id>",
      "amt": "1",
      "rl": "sr"
    }
  ],
  "total": "11"
}
```

Staking authority, stake position, reward claim, sale status, sale contribution, sale claim, sale refund, sale cancel, and sale withdrawal records are protocol records. Implementations should expose them through their own APIs without changing the record meaning.

## Example Applications

| Application | Main actions and kinds | Required invariant support | Notes |
| --- | --- | --- | --- |
| Cross chain marketplace | `execute`, `lock` with `kind: "htlc"`, `claim`, `refund`, `cancel-delegation`, allocations with `rl: "of"` and `rl: "sr"` | Delegated maker signatures, nonce uniqueness, hashlock preimage claims, refund height, fee and staking reward allocations. | Lets a maker sign once and a taker fill without the maker returning online. |
| Direct atomic swap | `lock` with `kind: "htlc"`, `claim`, `refund` | Matching hashlocks, different refund windows, and one-time lock consumption. | Can be coordinated without a marketplace if both parties exchange lock data. |
| OTC desk | `lock` with `kind: "otc"`, `claim`, `refund`, optional `execute` | Explicit counterparty reference in `data.cp`, hashlock or authority release, refund fallback. | Useful for negotiated bilateral trades that need a clear audit trail. |
| Vesting dashboard | `lock` with `kind: "vesting"`, `claim` | Height based release and required `data.dom` plus `data.ref`. | Suitable for team allocations, grants, investor unlocks, or protocol emissions. |
| Cooldown vault | `lock` with `kind: "cooldown"`, `claim` | Claim target must be the authority owner and no refund path is allowed. | Gives wallets or apps a protocol enforced waiting period before withdrawal. |
| Service escrow | `lock` with `kind: "escrow"`, `claim`, `refund` | Authority controlled claim, payer and payee binding, refund fallback. | A mediator can approve release while the payer keeps refund protection. |
| Staking rewards | `auth-cfg` with `k: "stk"`, `stake`, `claim-rwd`, `unstake`, allocations with `rl: "sr"` | Weighted tiers, per-position reward debt, reward tick rules, unlock height, empty pool policy. | Supports fee sharing, loyalty rewards, and long term holder programs. |
| Token launchpad | `auth-cfg` with `k: "sale"`, `fund-sale`, `contribute`, `finalize-sale`, `claim-sale`, `refund-sale`, `cancel-sale`, `withdraw-sale` | Hard cap, soft cap, contribution bounds, Merkle allowlist, treasury target, sale state transitions. | Supports fixed rate sales with refunds if the round fails. |
| Fee splitter | Allocations with target types `a`, `h`, and `b` | Allocation role uniqueness, target validation, refund returns all allocations to refund address. | Can route proceeds to operators, reward authorities, treasuries, or burn. |
| Signed coupon or reward claim | Redeem `items` or `lock` with authority condition | Signature uniqueness, authority ownership, optional domain reference in `data`. | Useful for offchain games, reward systems, and bridge crediting. |

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

TAP indexers track Bitmap according to the original Bitmap rules. Cursed support is not part of the TAP Bitmap path.

## DMT Tokens

TAP supports DMT elements for field `4` block height, field `10` nonce, and field `11` bits, following the DMT specification at https://digital-matter-theory.gitbook.io/digital-matter-theory/introduction/digital-matter-theory. The supported operations are `dmt-deploy` and `dmt-mint`.

From Bitcoin block `861576`, Blockdrops for DMT UNATs and Bitmaps are supported.

## DMT NAT Miner Rewards

From block `885588`, regular `dmt-nat` mints are ignored. Instead, DMT NAT rewards are credited from coinbase transaction outputs.

Rules:

- Each block distributes DMT NAT according to the bits value of that block.
- Reward shares follow the BTC value of coinbase outputs.
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
- Authority cancellation must be enforced as retirement for new obligations, not revocation of existing settlement rights.
- Locks, authority balances, transfer rows, stake positions, sale records, and delegation cancellations must remain internally consistent.
- Tests should cover happy paths, invalid shapes, duplicate consumption, cancelled authorities, insufficient available balances, decimals, stringified numbers, and attempts to bypass delegation or lock constraints.

The protocol is intentionally sparse in record keys because high volume indexes can hold millions of rows. New index fields should remain compact and should not duplicate data unless it is needed for efficient reads.
