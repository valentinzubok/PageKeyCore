# PageKey v1 — the first on-chain run (superseded)

This is the record of the first deployment, `0x909151ae…F95d`. Its refusals raised bare `Exception`s, which the current GenVM linter flags. The corrected contract and its run are in [`STUDIO_DEV_DEPLOY.md`](../STUDIO_DEV_DEPLOY.md); the binding logic is the same. Links below to `contracts/` point at the corrected file.

| | |
|---|---|
| **Network** | GenLayer Studio Dev / Studio Next — chain `61997` |
| **RPC** | `https://studio-dev.genlayer.com/api` |
| **Contract** | [`0x909151ae02909e88a2D03c73591e90a0DA0AF95d`](https://explorer-studio-dev.genlayer.com/address/0x909151ae02909e88a2D03c73591e90a0DA0AF95d) |
| **Deploy tx** | [`0xe8b0951f…0ffb02`](https://explorer-studio-dev.genlayer.com/tx/0xe8b0951f1580e0077a13278a2c0c2bda4af9dfd9df35389c1039d170c30ffb02) from `0x42F132a0519Ee2702A97f1f1f3Fe0E8041A2d542`, which has no role in the contract |
| **Source** | [`contracts/PageKey.py`](../contracts/PageKey.py) — runner `py-genlayer:5jycge4q8k23462jtb0b9fyey1s9qz928sz2nbrd9mg4sxqg2qng` |
| **Source sha256** | `e640f2e2247ed0cb9e8c450dc0afd866b4a16cd12d97fef49f252d82abdcaae9` |

## Verify the source yourself

```bash
curl -s -X POST https://studio-dev.genlayer.com/api -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"gen_getContractCode","params":["0x909151ae02909e88a2D03c73591e90a0DA0AF95d"]}' \
  | python3 -c "import sys,json,base64,hashlib; print(hashlib.sha256(base64.b64decode(json.load(sys.stdin)['result'])).hexdigest())"
shasum -a 256 contracts/PageKey.py
# both print e640f2e2247ed0cb9e8c450dc0afd866b4a16cd12d97fef49f252d82abdcaae9
```

`scripts/verify_deployment.py` does the same in CI on every push.

## The run

Three fresh accounts:

| Name | Address | Token for `profile.html` |
|---|---|---|
| Alice | `0xA56F1228B61AA7FBEcC1E95C8c870060fd1445b9` | `pagekey-2e5eccf721e9a2dd7167` |
| Bob | `0x5C1c6cADA907258b2061843865F6C9aA937e67B0` | `pagekey-94a7a7654ddc27f3486a` (and `pagekey-cf92ffd729ce28be37f2` for `thread.html`) |
| Carol | `0xb912dfcd8C7cfBEF9A0A7A09BC285fAFd1629154` | none: she only triggers rechecks |

Two pages, both files on the `main` branch of the console repository, fetched through
`https://raw.githubusercontent.com/valentinzubok/PageKey/main/web/public/fixtures/`:

- `profile.html` — a personal profile. First it is
  [Mara's profile](https://github.com/valentinzubok/PageKey/blob/9cdfcc2d86e516740b74d64047320c0a76460b10/web/public/fixtures/versions/profile-v1-mara.html)
  with Alice's token in the bio; a public commit then turns it into
  [a profile under new management](https://github.com/valentinzubok/PageKey/blob/9cdfcc2d86e516740b74d64047320c0a76460b10/web/public/fixtures/versions/profile-v2-new-owner.html)
  with Bob's token.
- [`thread.html`](https://github.com/valentinzubok/PageKey/blob/9cdfcc2d86e516740b74d64047320c0a76460b10/web/public/fixtures/thread.html) —
  a forum post. Bob's token for this page appears in it, in a visitor's comment.

Every transaction is listed, including the four that reverted and the three claims that were
accepted and bound nothing. Recheck interval 420 s, one miss lapses the binding.

| # | Call | Sender | Result | What happened | Tx |
|---|------|--------|--------|---------------|----|
| 1 | `claim(thread.html)` | **Bob** | accepted, **binds nothing** | Bob's token is on the page, but only inside a visitor's comment. Validators: `not_own` | [`0xb05e6691…0f8d0b`](https://explorer-studio-dev.genlayer.com/tx/0xb05e6691fea3d9aaf5541f712bdb16b9583ad34efca0e6fe7299d5e4e70f8d0b) |
| 2 | `claim(profile.html)` | **Bob** | accepted, **binds nothing** | the profile states Alice's token, not Bob's. `absent`, decided in code, no model asked | [`0x5c3fc269…594272`](https://explorer-studio-dev.genlayer.com/tx/0x5c3fc269c9348a015e0c0c665b38d30a9528d9657d466628bb891b535b594272) |
| 3 | `claim(profile.html)` | Alice | accepted, **bound** | the profile states her token in its own bio. Validators: `own`. Page sha-256 `92afb4c770b2…` | [`0x82db9f6c…6e0bbc`](https://explorer-studio-dev.genlayer.com/tx/0x82db9f6cbb9a0db02e6148a2c0f0542f2e6282fbfb6e4643b9376123a66e0bbc) |
| 4 | `claim(profile.html)` | **Bob** | **ERROR** | refused before any fetch: this page is bound to another address | [`0x1f69ab76…6c6d17`](https://explorer-studio-dev.genlayer.com/tx/0x1f69ab76690d3e0f47bb7dae87b155499e6a64ade6c34d4e315f6e15db6c6d17) |
| 5 | `recheck(profile.html)` | Carol | **ERROR** | refused: too early, the recheck interval has not elapsed | [`0xc4a6b8c9…986a9a`](https://explorer-studio-dev.genlayer.com/tx/0xc4a6b8c923cf86af3ea1e77157a21ab7446d628cfd7f0c5c0b36c18f87986a9a) |
| 6 | `release(profile.html)` | **Carol** | **ERROR** | refused: only the holder may release | [`0xbcc5f1b9…f2c2e0`](https://explorer-studio-dev.genlayer.com/tx/0xbcc5f1b90ddf742feb3b67e634c106f29d587207da7f2d21e2aa5c045cf2c2e0) |
| 7 | `recheck(profile.html)` | Carol | accepted | same hash `92afb4c770b2…` → `unchanged`: confirmed, no model asked, `last_seen_at` refreshed | [`0x5272a275…b51fb9`](https://explorer-studio-dev.genlayer.com/tx/0x5272a275bde1319f20f2df8b1b7feac9629ee2f34b2a45e87f9f7c50d6b51fb9) |
| 8 | `recheck(profile.html)` | Carol | accepted | the page was changed by commit [`9cdfcc2`](https://github.com/valentinzubok/PageKey/commit/9cdfcc2d86e516740b74d64047320c0a76460b10) ("under new management", `b41fc81aa188…`). Alice's token is gone → `absent` → the binding **lapsed** | [`0x827a7aa5…d73c5d`](https://explorer-studio-dev.genlayer.com/tx/0x827a7aa50ac1733181e749b8f1f23f50356642055adec49fd019a3917ad73c5d) |
| 9 | `claim(profile.html)` | **Alice** | accepted, **binds nothing** | the old holder tries again: her token is no longer there. `absent` | [`0xa00548d9…f71b8c`](https://explorer-studio-dev.genlayer.com/tx/0xa00548d97e4942eeada36973a86430c386cd5860df91fdfab0755f0184f71b8c) |
| 10 | `claim(profile.html)` | Bob | accepted, **bound** | the page now states Bob's token in its own voice. `own`. Holder #2 | [`0xf69a6313…10b1ef`](https://explorer-studio-dev.genlayer.com/tx/0xf69a6313f336c6ef0318e961596479d7e1cf88d261350170f8e980dacd10b1ef) |
| 11 | `release(profile.html)` | **Alice** | **ERROR** | refused: only the holder may release, and she is not the holder any more | [`0xf12ebfe6…21b9dc`](https://explorer-studio-dev.genlayer.com/tx/0xf12ebfe661f00e407e2610984d4a076fe04d36551195fa788fb076a72c21b9dc) |
| 12 | `release(profile.html)` | Bob | accepted | the holder gives the page up | [`0x3018708d…bc56f2`](https://explorer-studio-dev.genlayer.com/tx/0x3018708d3b9e8ed81e646dd7681e44cfa5451df3a32765cd18d1e3cb32bc56f2) |

### What this shows

| Attempt | Why it failed, or did not |
|---|---|
| A token in someone's comment section (row 1) | the token really is on the page, so a "contains" check would have bound it. Validators agreed it is not the page's own content |
| A token that belongs to another address (rows 2, 9) | tokens are derived from the page and the address; someone else's is simply absent |
| Taking a bound page (row 4) | reverts before the page is fetched |
| The page changes hands (rows 8, 10) | the old binding lapsed on the next recheck, and the new owner of the page could bind it |

`is_bound(profile, Alice, 3600)` returned `true` after row 3 and `is_bound(profile, Alice, 999999)`
returned `false` after row 8: a lapsed binding is not a binding at any age.

End state, read from chain:

```
owner_of(profile.html)   {"bound":false,"holder":"","status":"released"}
owner_of(thread.html)    {"bound":false,"holder":"","status":"none"}
get_binding(profile).history   claimed 92afb4c7… → unchanged 92afb4c7… → absent b41fc81a… → claimed b41fc81a…   (holder #2)
```

## Notes on Studio Dev

- The header must be `# v0.3.0` followed by the pinned `Depends` line.
- Every transaction carries a fee deposit (`genlayer-js@2.0.0-rc.1`, `studioDevnet`, `estimateTransactionFees`).
- `raw.githubusercontent.com` caches a branch file for about five minutes, so the demo used a
  420-second recheck interval and waited a full interval after the commit. The contract's own
  minimum is 60 seconds.
