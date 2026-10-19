# PageKey — Studio Dev (chain 61997) deploy record

| | |
|---|---|
| **Network** | GenLayer Studio Dev / Studio Next — chain `61997` |
| **Contract** | [`0xb5F562154eF5c8c6a22De84E2e32645E8c0aAe4F`](https://explorer-studio-dev.genlayer.com/address/0xb5F562154eF5c8c6a22De84E2e32645E8c0aAe4F) |
| **Deploy tx** | [`0x025f6dfe…abddf1`](https://explorer-studio-dev.genlayer.com/tx/0x025f6dfe223c02022ad659c5dc4e23d4a95ffb0ac80bf8475b3e75e7a0abddf1) from `0x73064D61A55Fdc895a87fA0432637CdBfc80B7a3`, which has no role in the contract |
| **Source** | [`contracts/PageKey.py`](contracts/PageKey.py) — runner `py-genlayer:5jycge4q8k23462jtb0b9fyey1s9qz928sz2nbrd9mg4sxqg2qng` |
| **Source sha256** | `857f77f54f26078e5832765e081d2fcd859cebaa6d82fba29ad050dcaeb3757b` |
| **Previous deployment** | `0x909151ae02909e88a2D03c73591e90a0DA0AF95d` — [its run](docs/STUDIO_DEV_V1_RUN.md): a page changing hands, the old binding lapsing |

## Lint

Every refusal raises `gl.vm.UserError`, and every method of the contract class takes `self`.

```bash
pip install genvm-linter
sh scripts/lint_current_sdk.sh
```

The script runs `genvm-lint lint` on the deployed file, then `genvm-lint check` and
`genvm-lint typecheck` on a copy whose header and imports are switched to the public runner
`1jb45aa8…` from GenVM v0.3.0-rc7. Studio Dev accepts only runner `5jycge4q…`, which is in no
public GenVM release, so the linter cannot download its SDK; the public runners are refused by
Studio Dev with `invalid_contract runner malformed`. Result: `Lint passed (3 checks)`,
`Validation passed`, `No type errors found`. CI runs the same script on every push.

## Verify the source yourself

```bash
curl -s -X POST https://studio-dev.genlayer.com/api -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"gen_getContractCode","params":["0xb5F562154eF5c8c6a22De84E2e32645E8c0aAe4F"]}' \
  | python3 -c "import sys,json,base64,hashlib; print(hashlib.sha256(base64.b64decode(json.load(sys.stdin)['result'])).hexdigest())"
shasum -a 256 contracts/PageKey.py
# both print 857f77f54f26078e5832765e081d2fcd859cebaa6d82fba29ad050dcaeb3757b
```

## The run

| Name | Address |
|---|---|
| Alice | `0xB6148C265E55060E81B1b4F74FD9Bf4C08079E60` |
| Bob | `0x70A54D608C4CaFd4537DB22B43ebE3709758964D` |
| Carol | `0x977cEe45900e48880e65AC26584D29419F88623F` |

Pages, fetched through `https://raw.githubusercontent.com/valentinzubok/PageKey/main/web/public/fixtures/`:
[`profile-2.html`](https://github.com/valentinzubok/PageKey/blob/a836907825ede0d84ced1baef57d9ae9bb3c8b58/web/public/fixtures/profile-2.html) states Bob's token in the owner's own
words; [`thread-2.html`](https://github.com/valentinzubok/PageKey/blob/a836907825ede0d84ced1baef57d9ae9bb3c8b58/web/public/fixtures/thread-2.html) carries Bob's token only in a
visitor's comment. Recheck interval 60 s, one miss lapses a binding.

| # | Call | Sender | Result | What happened | Tx |
|---|------|--------|--------|---------------|----|
| 1 | `claim(thread-2.html)` | **Bob** | accepted, **binds nothing** | Bob's token is on the page, but only in a visitor's comment. Validators: `not_own` | [`0x8332aade…09f166`](https://explorer-studio-dev.genlayer.com/tx/0x8332aadeae5d8d0251cfe19877498caa40bf78cf06bf9c308739e28c8e09f166) |
| 2 | `claim(profile-2.html)` | **Alice** | accepted, **binds nothing** | the page states Bob's token, not Alice's: `absent`, decided in code, no model asked | [`0x24c450e9…7ffb02`](https://explorer-studio-dev.genlayer.com/tx/0x24c450e958ffa04646ecf317b05b62d25392687c48f43c4febb5ae5d357ffb02) |
| 3 | `claim(profile-2.html)` | Bob | accepted, **bound** | the profile states his token in its own voice. Validators: `own`. Page sha-256 `957c4f5ea473…` | [`0xf10c11d4…2f0a3f`](https://explorer-studio-dev.genlayer.com/tx/0xf10c11d48b27a082567b9800e8de3f788607b7b1e136d9fa54cfdc7e8f2f0a3f) |
| 4 | `claim(profile-2.html)` | **Alice** | **ERROR** | **UserError** `this page is bound to another address` — refused before any fetch | [`0x9dd91aad…33f65e`](https://explorer-studio-dev.genlayer.com/tx/0x9dd91aadfc37c191e369350ecead11e85202f8e097801eae7e3c0ec5ff33f65e) |
| 5 | `recheck(profile-2.html)` | Carol | **ERROR** | **UserError** `too early: the next recheck opens at …` | [`0x6041973c…461834`](https://explorer-studio-dev.genlayer.com/tx/0x6041973c31c787f7e1a060d37c4aab5ebcff6c0fe987692958e7b8b227461834) |
| 6 | `release(profile-2.html)` | **Carol** | **ERROR** | **UserError** `only the holder may release` | [`0x7669df7f…00a20a`](https://explorer-studio-dev.genlayer.com/tx/0x7669df7f3042020cb0f1a859bfdd7a9597ab1865b65564861e342be59b00a20a) |
| 7 | `recheck(profile-2.html)` | Carol | accepted | same hash → `unchanged`: confirmed without a model, `last_seen_at` refreshed | [`0xa31caf76…3c9b40`](https://explorer-studio-dev.genlayer.com/tx/0xa31caf7686bd0410fff1314501a29d9e1497dc07837c4f23470e6623f83c9b40) |
| 8 | `release(profile-2.html)` | Bob | accepted | the holder gives the page up | [`0x39507dd8…d57752`](https://explorer-studio-dev.genlayer.com/tx/0x39507dd8e5e72a24a2e2a4f833227b4d3e3c2c54e6a1bf1f9ff0543052d57752) |

`is_bound(profile-2, Bob, 3600)` returned `true` after row 3 and `is_bound(profile-2, Alice, 3600)`
returned `false`. End state: `owner_of(profile-2.html)` = `{"bound":false,"holder":"","status":"released"}`,
history `claimed 957c4f5e… → unchanged 957c4f5e…`.

A page changing hands, and the old binding lapsing on a recheck, is shown in the
[first run](docs/STUDIO_DEV_V1_RUN.md); that logic is unchanged.
