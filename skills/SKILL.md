---
name: ccc-integration
description: "Integrates with the Corporate Cloud Consortium (CCC), a sovereign NFT platform and creator economy on the Internet Computer. Covers verifying CCC NFT tier holdings (ICRC-7) for token-gating features, reading public creator content and teaser previews, checking governance proposals and treasury state. Use when an app needs to gate features by CCC membership tier, display CCC content, or read CCC governance. Do NOT use for general ICRC-7 collections, general ICRC-1/ICRC-2 ledger operations, or building unrelated NFT platforms."
license: Apache-2.0
compatibility: "Read access to ICP mainnet. Query calls work anonymously; update calls require an authenticated (non-anonymous) principal."
metadata:
  title: "CCC Platform Integration"
  category: Integration
---

# CCC Platform Integration

## What This Is

The Corporate Cloud Consortium (CCC) is a sovereign creator platform on the
Internet Computer: a 50-NFT ICRC-7 collection across 5 membership tiers,
holder/subscriber-gated creator content, and on-chain governance that controls
the treasury (withdrawals execute only via passed proposals — there is no admin
bypass method). This skill documents the public surface a third-party app needs
to verify CCC tier holdings, read public content, and read governance state.

## Canister IDs

| Environment | Canister | ID |
|-------------|----------|-----|
| Mainnet | CCC platform (app + ICRC-7 collection + governance) | `ipchn-lqaaa-aaaam-qizkq-cai` |
| Mainnet | ICP ledger (payments) | `ryjl3-tyaaa-aaaaa-aaaba-cai` |

Web app: https://corporatecloudconsortium.cloud

Share links use the canonical base URL above — never canister raw URLs.

## Tier Model

50 NFTs, token IDs map to tiers by fixed ranges:

| Tier | Name | Token IDs |
|------|------|-----------|
| 1 | EXEC | #001–#010 |
| 2 | SYS | #011–#020 |
| 3 | OPS | #021–#030 |
| 4 | FLD | #031–#040 |
| 5 | USR | #041–#050 |

To compute a principal's tier: fetch the token IDs they hold (ICRC-7), take the
lowest token ID, map by range. Lower tier number = higher clearance. A
principal holding no CCC NFT has no tier.

## Common Pitfalls

1. **Update methods reject anonymous callers.** Every state-changing method on
   the platform canister returns `#err("Anonymous caller rejected")` for the
   anonymous principal. Queries (`getAllPosts`, `getProposals`,
   `getTreasuryBalance`, ICRC-7 reads) work anonymously.
2. **Gated content is redacted server-side, not hidden client-side.** List
   queries (`getAllPosts`, `getPostsByCreator`) and unentitled `getPostFull`
   calls return posts with `content = variant { TextOnly = "" }`. Detect a
   locked post by that empty `TextOnly`, not by a flag. The full body is never
   present in an unentitled response — do not attempt to recover it client-side.
3. **Use the `preview` field for teasers.** Every post record includes
   `preview : text` — the first ≤200 Unicode characters of the body, computed
   in the canister. It is safe to display to anyone. It is `""` for image-only
   posts.
4. **`getPostFull` is an update call, not a query.** Entitlement
   (NFT holder OR active subscriber) is checked against the caller, so it must
   be a signed call. Calling it anonymously returns the redacted post.
5. **Treasury withdrawals cannot be triggered by any direct method.** The
   former `withdrawTreasury(amount, recipient)` admin method was removed from
   the canister interface entirely. The only path is governance:
   `submitProposal` → votes → `finalizeProposal(id)` after expiry →
   `executeProposal(id)` (admin-signed). Do not look for or attempt a direct
   withdrawal method; it does not exist.
6. **Proposals do not auto-finalize.** An expired proposal stays
   `variant { Active }` until any authenticated principal calls
   `finalizeProposal(id)`. If you read proposal state, treat
   `Active` + `expiresAt < now` as "awaiting finalization", not as failed.
7. **Result types are `variant { ok : ...; err : text }`.** Match both arms.
   The `err` text is human-readable and stable enough to surface directly.
8. **Amounts are e8s.** Treasury balance and proposal amounts are `nat` e8s
   (1 ICP = 100_000_000 e8s). The ICP ledger fee is 10_000 e8s; transfer
   recipients receive `amount - 10_000`.

## Reading Governance State (verified live)

`getProposals` query signature, as served by the live canister:

```candid
getProposals : (opt variant { Failed; Passed; Active; Executed; Cancelled })
  -> (vec record {
        id : nat;
        status : variant { Failed; Passed; Active; Executed; Cancelled };
        title : text;
        description : text;
        proposer : principal;
        yesVotes : float64;
        noVotes : float64;
        totalVotingPower : float64;
        createdAt : int;
        expiresAt : int;
        executedAt : opt int;
        proposalType : variant {
          TreasuryWithdrawal : record { recipient : principal; amount : nat };
          SponsoredPost : record { budget : nat; postId : nat };
          SubscriptionFee : record { newFee : nat };
          RoyaltyAdjustment : record { newConfig : record {
            creatorShare : nat; treasuryShare : nat; holderPoolShare : nat } };
          SuccessionReversal : record { originalFounder : principal };
          ParameterChange : record { key : text; value : text };
        };
      }) query
```

Pass no filter (`null`) to get all proposals. `getProposal : (nat) -> (opt Proposal) query`
returns a single one. `getTreasuryBalance : () -> (nat) query` returns the
treasury accumulator in e8s.

Example — anonymous read with dfx:

```bash
dfx canister call ipchn-lqaaa-aaaam-qizkq-cai getProposals '(null)' --network ic
dfx canister call ipchn-lqaaa-aaaam-qizkq-cai getTreasuryBalance --network ic
```

## Reading Public Content

```bash
# All posts (gated bodies redacted; preview field always present)
dfx canister call ipchn-lqaaa-aaaam-qizkq-cai getAllPosts --network ic
```

Post content variants:

```candid
type PostContent = variant {
  TextOnly : text;
  WithImage : record { text : text; imageData : text };  // imageData = base64
  ExternalLink : record { text : text; url : text };
};
```

A locked post arrives as `content = variant { TextOnly = "" }` with `preview`
carrying the public teaser. Link to the post at
`https://corporatecloudconsortium.cloud/feed/post/{id}` for the user to
authenticate and subscribe.

## Verifying Tier Holdings (ICRC-7)

The collection lives on the platform canister and follows the ICRC-7 standard
(verified rendering natively in Oisy wallet 2.1.0). Query the holder's tokens
and map IDs to tiers with the table above.

> VERIFY BEFORE SHIPPING: confirm the exact ICRC-7 method set exposed by the
> canister (e.g. `icrc7_tokens_of`, `icrc7_owner_of`, `icrc7_balance_of`) by
> loading the live Candid interface:
> https://a4gq6-oaaaa-aaaab-qaa4q-cai.raw.icp0.io/?id=ipchn-lqaaa-aaaam-qizkq-cai

```motoko
// Sketch: tier from held token IDs (Motoko, mo:core)
func tierOf(tokenIds : [Nat]) : ?Nat {
  var best : ?Nat = null;
  for (id in tokenIds.vals()) {
    let tier =
      if (id <= 10) 1 else if (id <= 20) 2 else if (id <= 30) 3
      else if (id <= 40) 4 else 5;
    best := switch best { case null ?tier; case (?b) ?Nat.min(b, tier) };
  };
  best;
};
```

## Verify It Works

```bash
# 1. Governance state readable anonymously:
dfx canister call ipchn-lqaaa-aaaam-qizkq-cai getTreasuryBalance --network ic
# -> (nat) e8s value

# 2. Public feed readable, gated bodies redacted:
dfx canister call ipchn-lqaaa-aaaam-qizkq-cai getAllPosts --network ic
# -> posts with preview text; locked content = TextOnly ""

# 3. Live Candid interface (browser):
# https://a4gq6-oaaaa-aaaab-qaa4q-cai.raw.icp0.io/?id=ipchn-lqaaa-aaaam-qizkq-cai
```

## Related Skills

For ICRC-1/ICRC-2 payment flows (tips, subscriptions) see the official
icrc-ledger skill. For Internet Identity login see internet-identity.
For canister access-control patterns see canister-security.
