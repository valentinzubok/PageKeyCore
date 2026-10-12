# PageKeyCore

<p align="center">
  <strong>A wallet is bound to a web page when validators agree the page itself says so.</strong>
</p>

<p align="center">
  <a href="https://github.com/valentinzubok/PageKeyCore/actions/workflows/ci.yml"><img src="https://github.com/valentinzubok/PageKeyCore/actions/workflows/ci.yml/badge.svg" alt="CI" /></a>
  <img src="https://img.shields.io/badge/GenLayer-Studio%20Dev%2061997-38bdf8?style=flat-square" alt="Studio Dev" />
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue?style=flat-square" alt="MIT" /></a>
</p>

This repository holds the intelligent contract, its tests and its deploy record, and nothing
else. The console that uses it is a separate project: [PageKey](https://github.com/valentinzubok/PageKey).

---

## The trust problem

"This address belongs to whoever runs that page" is everywhere: the official wallet of a
project, the payout address of a maintainer, the account behind a profile. Today it is a badge
an issuer hands out and can take away, or a signed message nobody outside the wallet world can
verify. A contract cannot read either.

The naive on-chain version is "the page contains a code". It is wrong the moment the page has
a comment box: anyone can make a forum thread, a guestbook or an issue tracker "contain" their
code.

**PageKey makes the question a consensus one: is this token published by the page, in its own
voice?** And it keeps asking. A binding exists only while the page keeps saying so.

```
challenge_for(url, address)      a view. The token that address must publish on that page.
                                 Derived from both, so a copied token binds nothing.

claim(url, recheck_interval, misses_to_lapse)
      validators fetch the page and freeze it under eq_principle.strict_eq
        token not on the page          -> absent. Counted in code over the whole page;
                                          no model is asked
        token there                    -> validators agree on one boolean: is it part of the
                                          page's own content, and not a visitor comment, a
                                          quote, a list of other people's tokens or a code
                                          sample?
      own -> the page is bound to the caller. Anything else binds nothing.
      a page bound to someone else cannot be claimed; the call reverts before any fetch

recheck(url)                     anyone may call it; it takes a URL and nothing else
      refused until a full recheck_interval has elapsed since the last accepted check
        page identical to the verified one  -> confirmed, no model asked
        page changed, token still its own   -> confirmed
        token gone, or no longer its own    -> a miss
        unreachable                         -> the first is only noted; a second in a row
                                               is a miss
      misses_to_lapse misses in a row -> the binding LAPSES and the page is free again

release(url)                     the holder only

owner_of(url)                    {bound, holder, bound_at, last_seen_at, age}
is_bound(url, address, max_age)  what another contract calls. It chooses max_age:
                                 a binding nobody has rechecked lately is not a binding
```

### Why it fails the way it does

A binding is an identity claim that other contracts will act on, so doubt resolves to **not
bound**, and a malformed model answer, a model error or a consensus failure **reverts**: nothing
is recorded in either direction.

| Risk | What stops it |
|---|---|
| An issuer granting or revoking identities | There is none: no owner, no admin, no assign, no revoke, no transfer. The only ways a live binding ends are the holder's `release` and consecutive misses agreed by validators. |
| Posting your code in someone else's comment section | Presence is not enough. Validators must agree the token is the page's own content. The prompt lists what does not count and resolves "cannot tell who published it" to false. |
| Copying a token that is already on a page | The token is `sha256(url, address)`. Someone else's token is, for you, simply absent, and the claim is refused without a model call. |
| The model deciding whether the token is there | It never does. The token is counted in code over the whole normalized page before any prompt is built; the model is asked only who published it. |
| A page changing hands and the old binding living on | `recheck` is callable by anyone. A page that drops the token collects misses and lapses. Consumers pass `max_age` to `is_bound`, so a binding that has not been rechecked recently is treated as not bound. |
| Hammering `recheck` to lapse or to refresh a binding | `next_recheck_at` is stored on chain and moves to *this transaction's time + recheck_interval* after every accepted check. Elapsed time, not a calendar window. |
| An outage ending a healthy binding, or keeping a dead one fresh | One outage is only noted and never updates `last_seen_at`. Consecutive outages count as misses. |
| Taking over a page that is bound | `claim` on an actively bound page reverts before the page is fetched. |
| Spamming claims | A refused claim starts a 60-second cooldown for that address and page, measured in elapsed time. A reverted claim records nothing and starts none. |
| `"true"`, `1` or `"yes"` counted as a verdict | `literal_bool()` accepts only JSON `true`/`false`. Anything else reverts. |
| The page steering the model | The URL, the token and the page excerpts are fenced as untrusted data, inner fences are neutralized, and instruction-like phrasing is flagged to the model and stored on the binding. |
| Consensus quietly degrading | No fallback: if `prompt_comparative` cannot run, the transaction reverts. `principle` is passed positionally (it is positional-only in GenVM v0.3). |

## Live

| | |
|---|---|
| Network | GenLayer Studio Dev / Studio Next — chain `61997` |
| Contract | [`0x909151ae02909e88a2D03c73591e90a0DA0AF95d`](https://explorer-studio-dev.genlayer.com/address/0x909151ae02909e88a2D03c73591e90a0DA0AF95d) |
| Deploy record | [`STUDIO_DEV_DEPLOY.md`](STUDIO_DEV_DEPLOY.md) — every transaction of the demo, including the refused ones |

`scripts/verify_deployment.py` runs in CI and fails the build if the deployed bytes stop matching
[`contracts/PageKey.py`](contracts/PageKey.py).

The pages the on-chain demo read are kept in [`fixtures/`](fixtures/): a profile that states one
wallet's token, the same profile under new management, and a forum thread where a token appears
only in a visitor's comment.

## Tests

```bash
pip install -r requirements-dev.txt
python3 -m pytest -q      # 58 tests
```

[`tests/test_pagekey.py`](tests/test_pagekey.py), by what it proves:

- **Only the page's own voice binds.** A stated token binds; a copied token is absent and no
  model is asked; a token in a visitor comment does not bind; presence is counted over the whole
  page, including a token 14 KB in.
- **Nobody else can touch a binding.** No owner, admin, assign, revoke or transfer in the
  contract; a bound page cannot be claimed and is not even fetched; only the holder releases.
- **It keeps being true or it stops.** No recheck before a full interval for three accounts; the
  interval is elapsed time across a clock boundary; an unchanged page is confirmed with no
  prompt; a changed page is judged again; the token moving into a comment is a miss; misses lapse
  the binding and free the page; a good check resets them.
- **An outage is not a verdict.** First unreachable only noted, second is a miss, and an outage
  never refreshes `last_seen_at`.
- **Consumers choose freshness.** `is_bound` with different `max_age` values, and a lapsed or
  released binding is not bound at any age.
- **Nothing malformed becomes a verdict.** Twelve malformed answers revert a claim, six revert a
  recheck and leave the binding byte-identical; a model error and a consensus failure revert; a
  reverted claim does not start the cooldown.
- Fencing, planted fences, flagged injection phrasing, history with page hashes.

## License

[MIT](LICENSE) © 2026 Valentyn Zubok
