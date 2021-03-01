---
title: Caching Stack and Cabal Haskell builds on GitHub Actions
date: 2021-03-01
---

[mark-karpov-github-actions]: https://markkarpov.com/post/github-actions-for-haskell-ci.html
[github-ci-cache-docs]: https://docs.github.com/en/actions/guides/caching-dependencies-to-speed-up-workflows#using-the-cache-action

If you're building Haskell applications on GitHub Actions, you will certainly
want to employ caching between builds (unless you need a regular excuse for
coffee). This post won't explore the whole story -- I suggest [Mark Karpov's
blog][mark-karpov-github-actions] for that -- but will explain some caching
pitfalls, and a workaround for Cabal's constant rebuilding issue.

[Caching in GitHub CI][github-ci-cache-docs] works by attempting to match a
given key to a cache. If there isn't a match, then at the end of the workflow,
the paths mentioned are cached and stored under the given key. If there is one,
the cache is used, and **any changes are not re-cached**. In many situations,
it's useful to use an existing cache, but update and re-label it. The
`restore-keys` list is your friend here.

## Caching in Stack
Stack stores global stuff in `~/.stack`, and project stuff in `.stack-work`
(relative to the project root). We want to cache everything, and invalidate the
cache when one of our dependencies changes.

```yaml
- uses: actions/cache@v2
  with:
    path: |
      ~/.stack
      .stack-work
    # best effort for cache: tie it to Stack resolver and package config
    key: ${{ runner.os }}-stack-${{ hashFiles('stack.yaml.lock', 'package.yaml') }}
    restore-keys: |
      ${{ runner.os }}-stack
```

The primary key is derived from the Stack config (where dependencies go). If
it's not found, the latest cache with the prefix `linux-stack` (on Linux) will
be used, and re-cached as the derived primary key.

## Caching in Cabal
Attempting similar caching to Stack with Cabal will have it appear to only use
the cached dependencies. The project itself will still rebuilt, even if no
changes were made. In contrast, Stack just does what you expect, essentially
like you were building locally and made a change irrelevant to the code.

[Thanks to @tfausak](https://github.com/haskell/actions/issues/41), I learned
that it's due to GHC's rebuilding behaviour being like Make:

  > The way that GHC determines if it needs to rebuild a file is to compare the
  > source file's timestamp to the compiled file's timestamp. If the source file
  > is newer, it rebuilds.

When GitHub Actions checks out code, all file timestamps are reset. So
regardless of the cache, the project *will* be rebuilt. The reason this doesn't
happen with Stack is due in part to @tfausak's work last year, making Stack
decide whether to decide via file hashes instead. But this isn't the case with
GHC (nor Cabal). The relevant GHC issue is
[here](https://gitlab.haskell.org/ghc/ghc/-/issues/16495).

While that remains unresolved, you can work around it by setting all file
modification times to their last commit time. At the start of your workflow (or
whenever you run `actions/checkout`):

```yaml
# deep fetch, or we don't get the commit history needed to rewrite mod times
- uses: actions/checkout@v2
  with:
    fetch-depth: 0
- name: Set all tracked file modification times to the time of their last commit
  run: |
    rev=HEAD
    for f in $(git ls-tree -r -t --full-name --name-only "$rev") ; do
        touch -d $(git log --pretty=format:%cI -1 "$rev" -- "$f") "$f";
    done
```

With this, both Stack and Cabal have similar, expected caching behaviour (for
me). I hope this might save the reader a headache.
