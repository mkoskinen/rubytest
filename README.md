# avrea-ci-demo (Ruby)

A reproducible head-to-head between **GitHub Actions** and **Avrea** running
the same Ruby CI job. The Gemfile is shaped like a real Rails 8 app — 133
gems, mixing pure-Ruby downloads (`aws-sdk-s3`, `stripe`, `sentry-rails`)
with native extensions (`nokogiri`, `pg`, `bcrypt`, `image_processing`,
`ffi`, `grpc`). The CI job does the minimum useful work — `bundle install`
plus `bundle exec rails --version` — so install time dominates.

## What this measures

Three workflows, one job each, in [`.github/workflows/`](.github/workflows/):

| Workflow | Runner | Cache strategy |
|---|---|---|
| `ci-gha-nocache.yml` | `ubuntu-latest` | none (upper bound) |
| `ci-gha-cached.yml` | `ubuntu-latest` | `ruby/setup-ruby@v1` with `bundler-cache: true` — the "good citizen" GHA setup |
| `ci-avrea.yml` | `avrea-ubuntu-latest` | transparent edge package cache (no workflow config) |

Each workflow runs on `push` and `workflow_dispatch`, so you can replay any
scenario from the Actions tab.

## Results

Median of 5 runs per cell. Times are `bundle install` only (read from the
`::notice::` line in the run log; for the bundler-cache row, read from the
`setup-ruby` step's duration in the run UI — its bundle install is internal
to the action).

| Scenario | GHA (no cache) | GHA (bundler-cache) | Avrea |
|---|---|---|---|
| Cold first build           | … | … | … |
| Warm repeat build          | … | … | … |
| After lockfile change (Dependabot) | … | … | … |

Run logs:

- GHA (no cache): _link_
- GHA (bundler-cache): _link_
- Avrea: _link_

## How it works

`ruby/setup-ruby@v1`'s `bundler-cache: true` caches the installed
`vendor/bundle` directory keyed on `Gemfile.lock`'s hash. It's fast when
the lockfile is unchanged. It misses entirely the moment Dependabot bumps
any gem, because the cache key flips. The runner then redownloads every
gem from rubygems.org over the public internet and recompiles every native
extension.

Avrea's runners hit rubygems.org through a co-located mirror. The cache
is shared across all your repos, all your branches, and all your Ruby
versions — keyed on `<gem-name>-<version>` rather than `Gemfile.lock`'s
hash. So:

- **Cold first build:** Avrea is faster because the mirror is closer to
  the runner than rubygems.org's CDN, and most gems are already warm in
  the shared cache.
- **Warm repeat build:** GHA's per-repo `bundler-cache` wins by a little
  — it skips the resolver and the install step entirely. Avrea still
  downloads, just from a faster source. **Honest:** this is the row
  where GHA looks good.
- **Dependabot row:** GHA's cache misses → re-fetches everything from
  rubygems.org. Avrea's mirror still serves every gem the bumped one
  doesn't conflict with. This is where the architectural difference
  shows up.

The same pattern shows up in the [Ghostty/Nix](https://avrea.dev/ghostty)
(27×) and [Focal/Turborepo](https://avrea.dev/focal) (2×) writeups. Ruby
sits in the middle.

## Reproduce

1. Fork this repo.
2. Push any commit, or hit "Run workflow" on each of the three workflows
   in the Actions tab.
3. For the "cold" row, clear caches first:
   - GHA: Settings → Actions → Caches → delete.
   - Avrea: not needed — the edge cache is shared and read-only from
     your perspective; "cold" means a gem the mirror hasn't seen, which
     in practice means a brand-new release.
4. For the "Dependabot" row, run `bundle update nokogiri` locally, commit
   the `Gemfile.lock` change, push, and re-run all three workflows.
5. Repeat 3–5× per scenario and take the median. CI is noisy; one run
   isn't data.

## Stack details

- Ruby: see [`.ruby-version`](.ruby-version) (currently 3.4.8)
- Bundler: 2.6+
- Gem count: 133 (see [`Gemfile.lock`](Gemfile.lock))
- Platforms in lockfile: `x86_64-linux-gnu`, `x86_64-linux-musl`,
  `aarch64-linux-*`, `arm64-darwin`, `x86_64-darwin` — so the same
  lockfile works on every common runner without re-resolving.
