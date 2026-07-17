# dtf-operator-template

Forkable deployment template for running Reserve DTF proposer and defender automation from private GitHub Actions.

This template is currently intended for internal operators and testing. Fork it into a private repository. The private fork owns the non-secret operator config, GitHub Secrets, GitHub Variables, schedules, artifacts, and defender state for GitHub-hosted operation. Runtime code is not built in this repository; workflows consume the published image from `ghcr.io/reserve-protocol/dtf-operator` unless `DTF_OPERATOR_IMAGE` is overridden.

Do not put operator secrets in public repositories. Do not open deployment-config PRs against `reserve-protocol/dtf-operator` or `reserve-protocol/dtf-operator-template`.

## Runtime Model

- `DTF Proposer` wakes on a schedule, checks onchain lifecycle state, and exits unless a rebalance lifecycle action is due.
- `DTF Defender` wakes on a schedule, checks the configured Folio target or targets for new optimistic proposals, and exits unless a new `startRebalance()` proposal needs review.
- Non-secret operator configuration lives in `.github/dtf-operator.yml`.
- Secret material lives in GitHub Secrets or local Foundry keystores, never in the repository.
- Both workflows run the same Docker runtime image by default.

This template intentionally does not include Docker Compose files. Local Docker and self-hosted Compose operation are documented in `reserve-protocol/dtf-operator`, which owns the image contract and local runtime helpers.

The mainline setup uses one operator signer for proposing, launching auctions, and Defender vetoes. That signer needs optimistic proposer authority, `AUCTION_LAUNCHER_ROLE`, chain gas, and enough optimistic voting weight to veto malicious optimistic proposals. Proposer, auction launcher, and Defender signers can diverge when an operator has a specific reason to split duties; set the override secrets documented below for those cases.

## Quick Start

1. Fork this template into a private repository and enable GitHub Actions in the fork.
2. Leave `DTF_ACTIONS_ENABLED` unset until config and secrets are ready.
3. Choose the top-level DTF/Folio that Proposer will operate and, if needed, the additional Folios that Defender will protect.
4. Edit `.github/dtf-operator.yml` for those targets.
5. Authenticate CLIProxyAPI locally with its `-codex-login -no-browser` flow; the resulting auth directory defaults to `~/.cli-proxy-api`.
6. Install and authenticate `gh`, and install Foundry so `cast` is available.
7. Upload required GitHub Secrets with `scripts/setup-github-proposer` and/or `scripts/setup-github-defender`.
8. Set repository variable `DTF_ACTIONS_ENABLED=true` in the fork.
9. Run manual `dry-run` or alert-only tests before relying on scheduled live runs.

The scheduled jobs are guarded by `DTF_ACTIONS_ENABLED=true` and skip in `reserve-protocol/dtf-operator-template` itself.

## Image Selection

Default runtime image:

```text
ghcr.io/reserve-protocol/dtf-operator:stable
```

`stable` is the default branch-based channel for operator forks. It can receive background bugfix rollouts without moving every fork to the source repository's `main` integration channel.

Set repository variable `DTF_OPERATOR_IMAGE` to choose a different channel or pin a specific image. Use `ghcr.io/reserve-protocol/dtf-operator:main` for integration testing, or a `sha-*` or `v*` tag when a fork needs immutable runtime selection. The default GHCR package is public, so the workflows pull it without GHCR credentials. Deployment workflows do not build from source and do not use `SDK_READ_TOKEN`.

The workflows separately pin the tested CLIProxyAPI sidecar by immutable image digest. Leave `CLI_PROXY_IMAGE` unset for that default. Set it only when deliberately testing another digest; mutable tags such as `latest` are not recommended for production operators.

## Supported Chains

Set top-level `folioAddress` and `chainId` in `.github/dtf-operator.yml` for the DTF that Proposer operates. Defender uses that same target when `defender.folios` is omitted or empty. For multi-Folio Defender operation, each `defender.folios` entry supplies its own `folioAddress`, `chainId`, `governorAddress`, and `proposalScan.fromBlock`.

| Chain | `chainId` | Chain-specific RPC variable |
| --- | ---: | --- |
| Ethereum Mainnet | `1` | `MAINNET_RPC_URL` |
| BNB Smart Chain | `56` | `BSC_RPC_URL` |
| Base | `8453` | `BASE_RPC_URL` |

`RPC_URL` remains a generic fallback. Prefer chain-specific variables when Defender targets more than one chain from the same fork.

## GitHub Secrets And Variables

Common runtime inputs:

```text
RPC_URL or BASE_RPC_URL / MAINNET_RPC_URL / BSC_RPC_URL
CLIPROXY_AUTH_TGZ_B64
INFERENCE_API_KEY
```

Scout ETL endpoint, provider, and short-term API key access are owned by the published runtime image. Forks do not configure Scout ETL.

Required repository variable to enable workflows:

```text
DTF_ACTIONS_ENABLED=true
```

Optional repository variables:

```text
DTF_OPERATOR_IMAGE
CLI_PROXY_IMAGE
MIN_SIGNER_BALANCE_WEI
INFERENCE_AUTH_ALERT_COOLDOWN_SECONDS
PROPOSER_CODEX_MODEL
PROPOSER_CODEX_REASONING_EFFORT
DEFENDER_CODEX_MODEL
DEFENDER_CODEX_REASONING_EFFORT
```

Before scanning or proposing, each workflow queries CLIProxyAPI's authenticated management endpoint to verify that an active Codex OAuth credential is loaded. This check performs no model inference and consumes no subscription tokens. A failed check stops the operator before onchain work and sends a Resend email when `RESEND_API_KEY` and `DEFENDER_EMAIL_TO` are configured. Alerts are rate-limited by persisted workflow state; `INFERENCE_AUTH_ALERT_COOLDOWN_SECONDS` defaults to 86400 seconds.

The management endpoint is reachable only inside the job's private Docker network. The sidecar port exposed on the runner remains bound to `127.0.0.1`, the control panel is disabled, and the management key is the same per-repository `INFERENCE_API_KEY` Secret used by the runtime.

`INFERENCE_API_KEY` is not an OpenAI key and does not grant access to a model account. It is a random local gateway and management password for this repository's sidecar. Both setup helpers generate and upload a fresh value when the environment variable `INFERENCE_API_KEY` is unset; set that environment variable only when you deliberately want to choose the value.

`CLIPROXY_AUTH_TGZ_B64` is a base64-encoded CLIProxy OAuth auth archive. It bootstraps each runner and encrypts the refreshed CLIProxy auth archive saved in GitHub Actions cache after each run, so refresh-token rotations survive ephemeral runners. If you replace the bootstrap archive, update `CLIPROXY_AUTH_TGZ_B64`; the old encrypted cache will fail to decrypt and the next workflow run will bootstrap from the new secret.

`proposer.inference` and `defender.inference` in `.github/dtf-operator.yml` are required and default this fork to `gpt-5.5` with `medium` reasoning effort. The optional `PROPOSER_CODEX_*` and `DEFENDER_CODEX_*` repository variables override the matching YAML values without committing config changes. Reasoning effort must be `minimal`, `low`, `medium`, `high`, or `xhigh`.

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

Optional defender email sender override:

```text
DEFENDER_EMAIL_FROM
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
  inference:
    model: gpt-5.5
    reasoningEffort: medium
  optimisticProposerAddress: "0xGeneratedProposerSignerAddress"
  auctionLauncherAddress:
  proposalScan:
    fromBlock: 12345678
defender:
  frequencyMinutes: 30
  proposalScan:
    fromBlock: 12345678
  requireAiApproval: true
  alertOnlyWithoutSigner: true
  inference:
    model: gpt-5.5
    reasoningEffort: medium
```

Top-level `folioAddress` and `chainId` always identify the Proposer target. They are also the backward-compatible single-Folio Defender target when `defender.folios` is omitted or empty. `proposer.rebalanceCadence` is the deterministic auction cadence. For example, `30d` means the next rebalance auction should start at least 30 days after the first auction from the latest successful rebalance.

`defender.frequencyMinutes` is configurable and may be raised above `30`. It gates the workflow's 15-minute wakeups; it is not a hard maximum delay. GitHub Actions schedules are best-effort, so an eligible run can start later than the configured interval during platform congestion or while the repository-wide concurrency lock is occupied.

Set `proposer.proposalScan.fromBlock` before enabling scheduled runs. In legacy single-Folio Defender mode, also set `defender.proposalScan.fromBlock`; in multi-Folio mode, set `proposalScan.fromBlock` on every `defender.folios` entry instead. Use a chain block before the relevant DTF/Folio's first governance proposal so first-run log scans are bounded and RPC providers do not need to scan from genesis.

Defender automatically shrinks `eth_getLogs` block ranges when an RPC rejects or times out on a configured range. It saves discovered proposals and advances its reorg-aware cursor after every successful chunk, so a later provider failure resumes from completed work instead of restarting the historical scan. Leave `defender.proposalScan.logChunkBlocks` and `logChunkDelayMs` blank for normal operation; use them only as explicit provider-specific overrides.

`proposer.optimisticProposerAddress` is the generated proposer signer address used for dry-run optimistic proposal simulation. In the mainline setup, set `proposer.auctionLauncherAddress` to the same address after granting that signer `AUCTION_LAUNCHER_ROLE`.

### Multi-Folio Defender

To protect multiple Folios, add a non-empty `defender.folios` list. This list replaces the top-level target for Defender only; it does not change Proposer behavior.

```yaml
version: 1
folioAddress: "<PROPOSER_FOLIO_ADDRESS>"
chainId: 8453
proposer:
  rebalanceCadence: 30d
  proposalScan:
    fromBlock: 34567890
  inference:
    model: gpt-5.5
    reasoningEffort: medium
defender:
  folios:
    - folioAddress: "<FIRST_DEFENDER_FOLIO_ADDRESS>"
      chainId: 56
      governorAddress: "<FIRST_TRADING_GOVERNOR_ADDRESS>"
      proposalScan:
        fromBlock: 12345678
    - folioAddress: "<SECOND_DEFENDER_FOLIO_ADDRESS>"
      chainId: 8453
      governorAddress: "<SECOND_TRADING_GOVERNOR_ADDRESS>"
      proposalScan:
        fromBlock: 23456789
  frequencyMinutes: 30
  requireAiApproval: true
  alertOnlyWithoutSigner: true
  inference:
    model: gpt-5.5
    reasoningEffort: medium
```

Every entry requires the Folio's current trading governor address and an independently chosen scan start block. The runtime verifies the configured governor against the Folio before evaluating or broadcasting a veto. Configure the chain-specific RPC Secret or Variable for every chain represented in the list.

Multi-Folio Defender remains one GitHub Actions job and invokes `node dist/cli/run-github-defender.js` once. The runtime combines scans by chain, keeps each Folio's review state isolated inside the single Defender state cache, and serializes signing. Do not create a workflow matrix or a job per Folio. The job deliberately continues to use one CLIProxyAPI sidecar, one encrypted OAuth cache, one Defender state cache, one signer Secret set, and the repository-wide concurrency lock.

## Proposer Setup

Authenticate CLIProxyAPI before running either setup helper. Both helpers package the contents of `~/.cli-proxy-api` by default; pass `--cliproxy-auth-dir <path>` when the auth directory is elsewhere. They upload that tar.gz as the base64-encoded `CLIPROXY_AUTH_TGZ_B64` Secret. On the first workflow run, this Secret bootstraps the auth directory; each run then encrypts the refreshed archive with AES-GCM/scrypt and saves it in GitHub Actions cache because Actions cannot write Secrets.

Generate a new proposer keystore and upload required Secrets:

```bash
scripts/setup-github-proposer \
  --generate-keystore-dir ~/.dtf-operator/example/proposer-keystores \
  --keystore-password-file ~/.dtf-operator/example/proposer-password \
  --cliproxy-auth-dir ~/.cli-proxy-api \
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
  --cliproxy-auth-dir ~/.cli-proxy-api \
  --from-email defender@example.com \
  --repo YOUR_GITHUB_OWNER/YOUR_PRIVATE_FORK
```

Mainline onchain veto mode reuses the proposer signer. Delegate enough optimistic voting weight to the proposer signer address printed by `scripts/setup-github-proposer` for every distinct governance voting token used by the configured governors. Governors that share one voting token recognize the same delegation.

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

For a separate Defender signer, delegate enough optimistic voting weight to the printed defender signer address for every distinct governance voting token used by the configured governors. This does not grant proposer authority. Multi-Folio operation still uses this one Defender signer Secret set; it does not select a different signer per Folio.

Only proposals containing a configured target Folio's `startRebalance()` call are vetted. Non-rebalance proposals are archived and skipped. The cached/signed review ledger keeps review state isolated by Folio and prevents invoking Codex more than once for the same optimistic proposal during normal scheduled operation.

### Pending Veto Transaction Recovery

An unresolved submitted veto transaction intentionally blocks all further Defender signer use across every configured Folio. This signer-wide block prevents another Folio from consuming the shared signer's nonce while the earlier transaction's outcome is unknown. Normal runs continue checking the submitted transaction and release the block after it resolves.

The manual `DTF Defender` workflow exposes `abandon_submitted_veto_tx_hash` only for exceptional recovery. Its value must be the exact `0x`-prefixed, 32-byte hash of the tracked submitted veto transaction; the runtime rejects malformed or non-matching values. Supplying it is a dangerous explicit operator assertion that the transaction can no longer consume the signer nonce. Use it only after independently verifying that the transaction was dropped or replaced, or that its nonce is otherwise safe. Do not use it merely because one RPC endpoint temporarily cannot find the transaction or receipt.

## Signer Overrides

Use the same account by default. Override only the signers that must diverge:

- Proposer account: `FOUNDRY_ACCOUNT`, `PROPOSER_KEYSTORE_JSON_B64`, `PROPOSER_KEYSTORE_PASSWORD`.
- Auction-launcher account: `AUCTION_LAUNCHER_FOUNDRY_ACCOUNT`, `AUCTION_LAUNCHER_KEYSTORE_JSON_B64`, `AUCTION_LAUNCHER_KEYSTORE_PASSWORD`.
- Defender account: `DEFENDER_FOUNDRY_ACCOUNT`, `DEFENDER_KEYSTORE_JSON_B64`, `DEFENDER_KEYSTORE_PASSWORD`.

The setup helpers support existing keystores with `--keystore` and `--keystore-password-file`, or new keystores with `--generate-keystore-dir`. They can import a private key from an environment variable with `--import-private-key-env <ENV_VAR>`.

## Operational Safety

- Keep workflows off `pull_request` and never use `pull_request_target` for signer or Codex automation.
- Run scheduled and manual workflows only from the fork default branch after GitHub Actions is enabled.
- Use `DTF_ACTIONS_ENABLED=true` as the final arming switch after config and Secrets are ready.
- Leave `DTF_OPERATOR_IMAGE` unset to follow the default `stable` operator channel, or pin it to a `sha-*` or `v*` tag when production needs explicit runtime changes only.
- Proposer and Defender share one repo-wide GitHub Actions concurrency lock, `dtf-codex-auth-${{ github.repository_id }}`. If either workflow is running, the other waits instead of running with the same CLIProxy OAuth credentials.
- A multi-Folio Defender run is intentionally one job with one runtime invocation and serial signing. Keep one sidecar, OAuth cache, Defender state cache, and signer Secret set for the whole configured list.
- GitHub Actions keeps at most one pending run for that shared lock. If scheduled runs pile up while another Proposer or Defender run is active, older pending runs can be replaced by newer scheduled runs.
- Do not expect this repository's Defender workflow to catch this repository's own Proposer workflow in parallel. Live Proposer safety must come from its own preflight checks, signer controls, and proposer self-check before broadcast.
- Treat `CLIPROXY_AUTH_TGZ_B64` and the encrypted CLIProxy auth cache as dedicated to this operator repository. Do not use the same OAuth login in another repo, runner, machine, or concurrent job stream, because refresh tokens may rotate during normal use and an old reused copy can later fail as revoked or already used.
- Keep this repository private because workflow artifacts and config can reveal operational details even when Secrets are not committed.

## Source Repository

Runtime source, image builds, tests, and local Docker Compose helpers live in `reserve-protocol/dtf-operator`. Open PRs there only for shared source, image, runtime-contract, or local-runner changes. Do not open per-operator config PRs upstream.
