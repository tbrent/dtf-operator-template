# dtf-operator-template

Forkable deployment template for running Reserve DTF proposer and defender automation from private GitHub Actions.

Fork this repository into a private repository. The private fork owns the signer config, GitHub Secrets, GitHub Variables, schedules, artifacts, and defender state. Runtime code is not built in this repository; workflows consume the published image from `ghcr.io/reserve-protocol/dtf-operator` unless `DTF_OPERATOR_IMAGE` is overridden.

Do not put operator secrets in public repositories. Do not open deployment-config PRs against `reserve-protocol/dtf-operator` or `reserve-protocol/dtf-operator-template`.

## Runtime Model

- `DTF Proposer` wakes on a schedule, checks onchain lifecycle state, and exits unless a rebalance lifecycle action is due.
- `DTF Defender` wakes on a schedule, checks for new optimistic proposals, and exits unless a new `startRebalance()` proposal needs review.
- Public operator configuration lives in `.github/dtf-operator.yml`.
- Secret material lives in GitHub Secrets or local Foundry keystores, never in the repository.
- Both workflows run the same Docker runtime image by default.

The mainline setup uses one operator signer for proposing, launching auctions, and Defender vetoes. That signer needs optimistic proposer authority, `AUCTION_LAUNCHER_ROLE`, chain gas, and enough optimistic voting weight to veto malicious optimistic proposals. Proposer, auction launcher, and Defender signers can diverge when an operator has a specific reason to split duties; set the override secrets documented below for those cases.

## Quick Start

1. Fork this template into a private repository and enable GitHub Actions in the fork.
2. Leave `DTF_ACTIONS_ENABLED` unset until config and secrets are ready.
3. Choose the DTF/Folio address and chain this fork will operate.
4. Edit `.github/dtf-operator.yml` for that DTF/Folio.
5. Run `codex login` locally.
6. Install and authenticate `gh`, and install Foundry so `cast` is available.
7. Upload required GitHub Secrets with `scripts/setup-github-proposer` and/or `scripts/setup-github-defender`.
8. Set repository variable `DTF_ACTIONS_ENABLED=true` in the fork.
9. Run manual `dry-run` or alert-only tests before relying on scheduled live runs.

The scheduled jobs are guarded by `DTF_ACTIONS_ENABLED=true` and skip in `reserve-protocol/dtf-operator-template` itself.

## Image Selection

Default runtime image:

```text
ghcr.io/reserve-protocol/dtf-operator:main
```

Set repository variable `DTF_OPERATOR_IMAGE` to pin a specific tag, preferably a `sha-*` or `v*` tag. If the GHCR image is private, set `GHCR_READ_TOKEN` and optionally `GHCR_USERNAME` so the workflows can pull it. Deployment workflows do not build from source and do not use `SDK_READ_TOKEN`.

## Supported Chains

Set `folioAddress` and `chainId` in `.github/dtf-operator.yml` for the DTF you operate.

| Chain | `chainId` | Chain-specific RPC variable |
| --- | ---: | --- |
| Ethereum Mainnet | `1` | `MAINNET_RPC_URL` |
| BNB Smart Chain | `56` | `BSC_RPC_URL` |
| Base | `8453` | `BASE_RPC_URL` |

`RPC_URL` remains a generic fallback. Prefer chain-specific variables when running multiple deployments from the same fork or host.

## GitHub Secrets And Variables

Common runtime inputs:

```text
SCOUT_ETL_API_KEY
RPC_URL or BASE_RPC_URL / MAINNET_RPC_URL / BSC_RPC_URL
CODEX_AUTH_JSON_B64
```

Required repository variable to enable workflows:

```text
DTF_ACTIONS_ENABLED=true
```

Optional repository variables:

```text
DTF_OPERATOR_IMAGE
SCOUT_ETL_BASE_URL
SCOUT_ETL_PROVIDER
GHCR_USERNAME
MIN_SIGNER_BALANCE_WEI
```

Optional image access secret:

```text
GHCR_READ_TOKEN
```

Required for live proposer broadcasts:

```text
PROPOSER_KEYSTORE_JSON_B64
PROPOSER_KEYSTORE_PASSWORD
FOUNDRY_ACCOUNT
```

Optional auction-launcher signer overrides:

```text
AUCTION_LAUNCHER_KEYSTORE_JSON_B64
AUCTION_LAUNCHER_KEYSTORE_PASSWORD
AUCTION_LAUNCHER_FOUNDRY_ACCOUNT
```

When auction-launcher-specific secrets are unset, the runtime can use the proposer signer. `proposer.auctionLauncherAddress` is still explicit in `.github/dtf-operator.yml`; set it only after granting that signer `AUCTION_LAUNCHER_ROLE`.

Required for defender alert-only mode:

```text
RESEND_API_KEY
DEFENDER_EMAIL_TO
```

Optional separate defender veto signer:

```text
DEFENDER_KEYSTORE_JSON_B64
DEFENDER_KEYSTORE_PASSWORD
DEFENDER_FOUNDRY_ACCOUNT
```

When Defender-specific signer secrets are unset, the unchanged workflows pass proposer signer secrets so Defender vetoes can use the main operator signer.

## Operator Config

Edit `.github/dtf-operator.yml` in your private fork:

```yaml
version: 1
folioAddress: "0xYourDtfOrFolioAddress"
chainId: 8453
proposer:
  rebalanceCadence: 30d
  optimisticProposerAddress: "0xGeneratedProposerSignerAddress"
  auctionLauncherAddress:
defender:
  frequencyMinutes: 30
  requireAiApproval: true
  alertOnlyWithoutSigner: true
```

`folioAddress` and `chainId` must identify the DTF/Folio this fork operates. `proposer.rebalanceCadence` is the deterministic auction cadence. For example, `30d` means the next rebalance auction should start at least 30 days after the first auction from the latest successful rebalance.

`proposer.optimisticProposerAddress` is the generated proposer signer address used for dry-run optimistic proposal simulation. In the mainline setup, set `proposer.auctionLauncherAddress` to the same address after granting that signer `AUCTION_LAUNCHER_ROLE`.

## Proposer Setup

Generate a new proposer keystore and upload required Secrets after running `codex login` locally:

```bash
scripts/setup-github-proposer \
  --generate-keystore-dir ~/.dtf-operator/example/proposer-keystores \
  --keystore-password-file ~/.dtf-operator/example/proposer-password \
  --init-keystore-password \
  --foundry-account example-proposer \
  --repo YOUR_GITHUB_OWNER/YOUR_PRIVATE_FORK
```

The helper prints `signer_address`. Fund it with chain gas, grant optimistic proposer authority, grant `AUCTION_LAUNCHER_ROLE` if this signer should open auctions, then update `.github/dtf-operator.yml`.

Manual proposer tests:

- Run `DTF Proposer` with `mode=dry-run` to produce proposal artifacts without live signer material.
- Run `mode=observe-only` to test live RPC and lifecycle scanning without broadcasting.
- Run `mode=live-one-cycle` with `confirm_live_broadcast=true` only after reviewing artifacts and signer permissions.

Scheduled proposer runs use `live-one-cycle` behavior automatically.

## Defender Setup

Upload alert-only secrets:

```bash
RESEND_API_KEY=<key> scripts/setup-github-defender \
  --email alerts@example.com \
  --repo YOUR_GITHUB_OWNER/YOUR_PRIVATE_FORK
```

Mainline onchain veto mode reuses the proposer signer. Delegate enough optimistic voting weight to the proposer signer address printed by `scripts/setup-github-proposer`.

Generate a separate defender keystore and upload veto secrets only when the veto signer should diverge from the proposer signer:

```bash
RESEND_API_KEY=<key> scripts/setup-github-defender \
  --email alerts@example.com \
  --generate-keystore-dir ~/.dtf-operator/example/defender-keystores \
  --keystore-password-file ~/.dtf-operator/example/defender-password \
  --init-keystore-password \
  --foundry-account example-defender \
  --repo YOUR_GITHUB_OWNER/YOUR_PRIVATE_FORK
```

For a separate Defender signer, delegate enough optimistic voting weight to the printed defender signer address. This does not grant proposer authority.

Only proposals containing the target Folio `startRebalance()` call are vetted. Non-rebalance proposals are archived and skipped. The cached/signed review ledger prevents invoking Codex more than once for the same optimistic proposal during normal scheduled operation.

## Signer Overrides

Use the same account by default. Override only the signers that must diverge:

- Proposer account: `FOUNDRY_ACCOUNT`, `PROPOSER_KEYSTORE_JSON_B64`, `PROPOSER_KEYSTORE_PASSWORD`.
- Auction-launcher account: `AUCTION_LAUNCHER_FOUNDRY_ACCOUNT`, `AUCTION_LAUNCHER_KEYSTORE_JSON_B64`, `AUCTION_LAUNCHER_KEYSTORE_PASSWORD`.
- Defender account: `DEFENDER_FOUNDRY_ACCOUNT`, `DEFENDER_KEYSTORE_JSON_B64`, `DEFENDER_KEYSTORE_PASSWORD`.

The setup helpers support existing keystores with `--keystore` and `--keystore-password-file`, or new keystores with `--generate-keystore-dir`. They can import a private key from an environment variable with `--import-private-key-env <ENV_VAR>`.

## Local Compose

Local Compose is optional. It is useful for long-running self-hosted operator deployments or smoke tests against the published image.

1. Copy `config/compose.env.example` to a private path such as `/srv/YOUR_DTF_SLUG/compose.env`.
2. Copy `config/project.env.example` to `/srv/YOUR_DTF_SLUG/config/project.env` and fill DTF, chain, RPC, Scout, and notification values.
3. Put Codex OAuth `auth.json` at `/srv/YOUR_DTF_SLUG/codex/auth.json` when AI review/self-check will run.
4. Put Foundry keystores and password files under `/srv/YOUR_DTF_SLUG/foundry-keystores`.
5. Run Compose with `--env-file /srv/YOUR_DTF_SLUG/compose.env`.

Example smoke command:

```bash
docker compose --env-file /srv/YOUR_DTF_SLUG/compose.env --profile defender up --abort-on-container-exit --exit-code-from defender defender
```

Keep local `LIVE`, `ALLOW_BROADCAST`, and `VETO_BROADCAST` at `0` until dry-run and observe-only behavior is understood.

## Operational Safety

- Keep workflows off `pull_request` and never use `pull_request_target` for signer or Codex automation.
- Run scheduled and manual workflows only from the fork default branch after GitHub Actions is enabled.
- Use `DTF_ACTIONS_ENABLED=true` as the final arming switch after config and Secrets are ready.
- Pin `DTF_OPERATOR_IMAGE` for production deployments instead of floating on `main`.
- Keep this repository private because workflow artifacts and config can reveal operational details even when Secrets are not committed.

## Source Repository

Runtime source, image builds, and tests live in `reserve-protocol/dtf-operator`. Open PRs there only for shared source, image, workflow, docs, or setup-template changes. Do not open per-operator config PRs upstream.
