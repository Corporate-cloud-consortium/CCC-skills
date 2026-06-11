
name: ccc-integration
description: “Integrates with the Corporate Cloud Consortium (CCC), a sovereign NFT platform and creator economy on the Internet Computer. Covers verifying CCC NFT tier holdings (ICRC-7) for token-gating features, reading public creator content and teaser previews, checking governance proposals and treasury state. Use when an app needs to gate features by CCC membership tier, display CCC content, or read CCC governance. Do NOT use for general ICRC-7 collections, general ICRC-1/ICRC-2 ledger operations, or building unrelated NFT platforms.”
license: Apache-2.0
compatibility: “Read access to ICP mainnet. Query calls work anonymously; update calls require an authenticated (non-anonymous) principal.”
metadata:
title: “CCC Platform Integration”
category: Integration

CCC Platform Integration

What This Is

The Corporate Cloud Consortium (CCC) is a sovereign creator platform on the Internet Computer: a 50-NFT ICRC-7 collection across 5 membership tiers, holder/subscriber-gated creator content, and on-chain governance that controls the treasury (withdrawals execute only via passed proposals — there is no admin bypass method). This skill documents the public surface a third-party app needs to verify CCC tier holdings, read public content, and read governance state.

Canister IDs

|Environment|Canister                                           |ID                           |
|-----------|---------------------------------------------------|-----------------------------|
|Mainnet    |CCC platform (app + ICRC-7 collection + governance)|`ipchn-lqaaa-aaaam-qizkq-cai`|
|Mainnet    |ICP ledger (payments)                              |`ryjl3-tyaaa-aaaaa-aaaba-cai`|

Web app: https://corporatecloudconsortium.cloud
Share links use the canonical base URL above — never canister raw URLs.

Tier Model

50 NFTs, token IDs map to tiers by fixed ranges:

|Tier|Name|Token IDs|
|----|----|---------|
|1   |EXEC|#001–#010|
|2   |SYS |#011–#020|
|3   |OPS |#021–#030|
|4   |FLD |#031–#040|
|5   |USR |#031–#050|

To compute a principal’s tier: fetch the token IDs they hold (ICRC-7), take the lowest token ID, map by range. Lower tier number = higher clearance. A principal holding no CCC NFT has no tier.
