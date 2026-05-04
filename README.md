# opencodehub-testbed

Heavy evaluation, gym, corpus, and bench material for
[**OpenCodeHub**](https://github.com/theagenticguy/opencodehub). Split out
of the core repo so that the shipped CLI stays lean (target: under 600
tracked files). This repo consumes `@opencodehub/cli@latest` via `pnpm`
and exercises it nightly against known corpora.

## Why a separate repo?

The core OpenCodeHub monorepo ships 11 TypeScript packages plus a
Starlight docs site. Heavy regression fixtures — the 49-case parametrized
parity harness, multi-language gym corpora (TypeScript, Python, Go,
Rust, monorepo), baseline manifests — bloated the core repo to ~970
tracked files with multi-GB submodules. None of that content is needed
by consumers of the published CLI. Extracting it into this repo:

1. Keeps the core repo fast to clone, audit, and review.
2. Lets the testbed iterate on its own cadence (nightly re-runs against
   every new CLI release, not pinned to every core commit).
3. Models the real consumer contract — the testbed installs
   `@opencodehub/cli@latest` from npm and calls the public CLI exactly
   the way a user would.

## Layout

```
opencodehub-testbed/
  eval/                      # Python parity/regression harness (pytest,
                             # 49 parametrized cases across 14 fixture
                             # languages). Consumes `codehub` via MCP.
  gym/                       # TypeScript SCIP-indexer gym:
                             #   src/         — freeze/replay CLI
                             #   corpus/      — per-language fixture YAMLs
                             #   baselines/   — pinned manifest + thresholds
                             #   scripts/     — bench helpers
  bench/                     # Ad-hoc benchmark writeups (rust spike, etc.)
  fixtures/                  # Archival copies of in-core test fixtures
                             # (sarif, scip-ingest) for cross-ref.
  .github/workflows/
    nightly.yml              # Runs codehub analyze + gym + eval nightly.
```

## Consumption contract

Everything in here targets the **published** CLI, not a workspace link:

```bash
# Install the latest published CLI
pnpm add -g @opencodehub/cli@latest   # installs `codehub`
codehub --version

# Run eval harness
cd eval && uv sync && uv run pytest

# Run the gym against its baseline
cd gym && pnpm install && pnpm run build
node dist/cli.js run --baseline baselines/manifest.jsonl

# Smoke: analyze a single corpus repo
codehub analyze gym/corpus/repos/typescript/ts-pattern
```

The nightly workflow opens a GitHub issue if any corpus's `graphHash`
drifts from the baseline. Intentional baseline refreshes happen via
`node gym/dist/cli.js baseline` followed by a PR to this repo.

## Relationship to core

| Concern | Lives in | Updated by |
|---|---|---|
| CLI source, 11 packages | core repo | developers merging to `main` |
| Published CLI | npm (`@opencodehub/cli`) | core release-please workflow |
| Heavy evals & corpora | **this repo** | testbed maintainers |
| Minimal per-package fixtures | core repo | co-located with tests |

Changes to core that break corpus analysis surface here as a nightly
regression issue, not as a blocked PR — the core repo's own gates stay
fast.

## License

Apache-2.0. See [LICENSE](./LICENSE). The extracted content was copied
from the core repo at commit `6d5bc2cc18a7439962c76d34f8aab8c000ab3608`
(branch `feat/v1-m1-m2`).
