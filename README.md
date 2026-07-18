# dtf-operator-template

Forkable deployment template for running Reserve DTF proposer and defender automation from private GitHub Actions.

This template is currently intended for internal operators and testing. Fork it into a private repository. The private fork owns non-secret config, GitHub Secrets and Variables, schedules, and workflow artifacts; authoritative OAuth and runtime state live in separate private Git repositories. Runtime code is not built here; authoritative workflows require a digest-pinned `DTF_OPERATOR_JOURNAL_IMAGE`.

Do not put operator secrets in public repositories. Do not open deployment-config PRs against `reserve-protocol/dtf-operator` or `reserve-protocol/dtf-operator-template`.

## Runtime Model

- `DTF Proposer` wakes on a schedule, checks onchain lifecycle state, and exits unless a rebalance lifecycle action is due.
- `DTF Defender` wakes on a schedule, checks the configured Folio target or targets for new optimistic proposals, and exits unless a new `startRebalance()` proposal needs review.
- Non-secret operator configuration lives in `.github/dtf-operator.yml`.
- Secret material lives in GitHub Secrets or local Foundry keystores, never in the repository.
- Both workflows run the same Docker runtime image by default.

This template intentionally does not include Docker Compose files. Local Docker and self-hosted Compose operation are documented in `reserve-protocol/dtf-operator`, which owns the image contract and local runtime helpers.

The authoritative workflows require distinct Proposer and Defender signers so their independent GitHub concurrency locks cannot race on one account nonce. The Proposer signer needs optimistic proposer authority and chain gas; it may also launch auctions when it has `AUCTION_LAUNCHER_ROLE`. The Defender signer needs chain gas and enough optimistic voting weight to veto malicious optimistic proposals.

## Quick Start

1. Fork this template into a private repository and enable GitHub Actions in the fork.
2. Leave `DTF_ACTIONS_ENABLED` unset until config and secrets are ready.
3. Choose the top-level DTF/Folio that Proposer will operate and, if needed, the additional Folios that Defender will protect.
4. Edit `.github/dtf-operator.yml` for those targets.
5. Authenticate CLIProxyAPI locally with its `-codex-login -no-browser` flow; the resulting auth directory defaults to `~/.cli-proxy-api`.
6. Install and authenticate `gh`, and install Foundry so `cast` is available.
7. Upload required GitHub Secrets with `scripts/setup-github-proposer` and/or `scripts/setup-github-defender`.
8. Complete the authoritative Git cutover below while `DTF_ACTIONS_ENABLED` remains unset.
9. Set repository variable `DTF_ACTIONS_ENABLED=true` only after cutover validation succeeds.
10. Run manual `dry-run` or alert-only tests before relying on scheduled live runs.

The scheduled jobs are guarded by `DTF_ACTIONS_ENABLED=true` and skip in `reserve-protocol/dtf-operator-template` itself.

## Authoritative Git cutover (required for live operation)

`git-authoritative` is the only mode in which these workflows execute.
CPA GitStore replaces the OAuth cache for both workflows. The private operator
journal replaces cached Defender runtime state; Proposer transaction persistence
is not part of this initial cutover. Caches are not restored or saved by the
authoritative workflows and must not be revived as a rollback source.

Create three private repositories or otherwise demonstrably isolated write
domains: one `OPERATOR_STATE_GIT_URL` journal consumed by the runtime, one
CPA GitStore for Proposer, and one CPA GitStore for Defender. The two CPA
stores must contain different OAuth logins. This is what lets Defender run on
its own schedule while a four-hour Proposer job is active. Sharing a rotating
OAuth login, CPA GitStore branch, or signer is unsafe: use a single external
writer/lock in that case, accepting that the shared resource cannot be
scheduled independently. Do not silently run the two writers concurrently.

Configure these GitHub Variables (URLs contain no credentials):

```text
OPERATOR_AUTHORITY_MODE=git-authoritative
DTF_OPERATOR_JOURNAL_IMAGE=ghcr.io/reserve-protocol/dtf-operator@sha256:<journal-compatible-digest>
OPERATOR_STATE_GIT_URL
OPERATOR_STATE_GIT_BRANCH
CPA_PROPOSER_GITSTORE_URL
CPA_PROPOSER_GITSTORE_BRANCH
CPA_DEFENDER_GITSTORE_URL
CPA_DEFENDER_GITSTORE_BRANCH
CPA_GITSTORE_GIT_USERNAME (optional; only when the Git host requires it)
CPA_OAUTH_ISOLATION=separate-gitstores-and-logins
PROPOSER_SIGNER_ADDRESS
DEFENDER_SIGNER_ADDRESS
```

Configure these scoped, write-only GitHub Secrets. Grant each token access only
to its named private repository and never put it in a URL, config file, or log:

```text
OPERATOR_STATE_GIT_TOKEN
CPA_PROPOSER_GITSTORE_TOKEN
CPA_DEFENDER_GITSTORE_TOKEN
INFERENCE_API_KEY
DEFENDER_KEYSTORE_JSON_B64
DEFENDER_KEYSTORE_PASSWORD
DEFENDER_FOUNDRY_ACCOUNT
```

CLIProxyAPI GitStore is verified against the v7.2.83 sidecar used here. Its
actual environment contract is `GITSTORE_GIT_URL` (enables GitStore), optional
`GITSTORE_GIT_USERNAME`, `GITSTORE_GIT_TOKEN`, optional
`GITSTORE_GIT_BRANCH`, and optional `GITSTORE_LOCAL_PATH`. It clones/updates
the selected branch and owns `auths/` plus `config/config.yaml` in that
repository. The workflows map the repository-specific variables/secrets above
to those exact sidecar environment names; GitHub Secret names are deliberately
template-specific, not assumed CLIProxyAPI names.

Bootstrap each empty CPA repository once from a trusted operator workstation:

```bash
GITSTORE_GIT_TOKEN=<scoped-token> scripts/seed-cliproxy-gitstore \
  --git-url https://github.com/OWNER/private-cpa-proposer.git --git-branch main \
  --cliproxy-auth-dir ~/.cli-proxy-api
```

Repeat using a different OAuth login and the Defender CPA repository. The seed
script passes the token only as the documented process environment, imports
JSON through the authenticated management API, and does not generate an auth
archive or credential-bearing config. Verify the private repository content
and OAuth health before enabling the workflows. `CLIPROXY_AUTH_TGZ_B64` and
the old encrypted Actions cache are bootstrap/forensic artifacts only; remove
the Secret after a successful seed and never restore either after cutover.

Before seeding operator state, configure a branch rule for
`OPERATOR_STATE_GIT_BRANCH` that forbids force-pushes and deletion, permits only
the scoped journal writer and the migration operator to push, and does not
require pull requests. Treat both the protected branch and the configured
branch name as part of the authority trust anchor: do not retarget the variable
or rewrite history after cutover.

With every legacy Defender writer paused and drained, run the journal-compatible
image from a trusted workstation. Use an absolute path containing the legacy
Defender state to migrate, and enter the token without putting it in a command
line or Git URL:

```bash
export DTF_OPERATOR_JOURNAL_IMAGE='ghcr.io/reserve-protocol/dtf-operator@sha256:<digest>'
export LEGACY_DEFENDER_DATA='/absolute/path/to/legacy-defender-data'
export OPERATOR_STATE_GIT_URL='https://github.com/OWNER/private-operator-state.git'
export OPERATOR_STATE_GIT_BRANCH='operator-state'
export OPERATOR_STATE_MODE='git-authoritative'
read -rsp 'Operator-state Git token: ' OPERATOR_STATE_GIT_TOKEN && printf '\n'
export OPERATOR_STATE_GIT_TOKEN

operator_state() {
  docker run --rm \
    -e OPERATOR_STATE_GIT_URL \
    -e OPERATOR_STATE_GIT_BRANCH \
    -e OPERATOR_STATE_GIT_TOKEN \
    -e OPERATOR_STATE_MODE \
    -v "$LEGACY_DEFENDER_DATA:/data" \
    "$DTF_OPERATOR_JOURNAL_IMAGE" \
    node dist/cli/run-defender-state.js "$@" --state-root /data
}

operator_state seed       # captures allowlisted legacy files and persists shadow generation 1
operator_state validate
operator_state cutover    # refuses unresolved legacy submissions without exact signed bytes
operator_state restore    # round-trip the authoritative branch into the local state root
operator_state validate
operator_state status
unset OPERATOR_STATE_GIT_TOKEN
```

If a shadow journal was created earlier without Git-authoritative environment
variables, run `operator_state persist` before `operator_state cutover`. Do not
edit journal JSON by hand. Confirm that cutover reports `lifecycle:
"authoritative"`, a monotonic generation, and a Git commit before enabling any
new writer.

Perform the one-way cutover in this order:

1. Pause scheduled and manual legacy jobs and let every running job drain.
2. Reconcile each signer’s nonce, all pending/replaced transactions, and every
   pending Defender review before copying any state.
3. Seed and verify the CPA GitStores, then seed the private operator-state
   journal using the journal-compatible runtime’s migration procedure.
4. Pin `DTF_OPERATOR_JOURNAL_IMAGE` by digest and set the variables and
   secrets above. Authority mode is supplied by the workflow, not the public YAML.
5. Enable exactly one writer during initial reconciliation. Only after the
   journal proves clean should the separately credentialed Defender schedule be
   enabled.
6. Set `DTF_ACTIONS_ENABLED=true` last. The workflow guards reject all modes
   except `OPERATOR_AUTHORITY_MODE=git-authoritative`.

Legacy rollback is intentionally non-broadcasting. It may observe or export
evidence, but must not submit transactions unless a separately reviewed reverse
migration has restored nonce, pending-transaction, review, and journal state.

### Runtime integration assumption

The Defender workflow passes `OPERATOR_STATE_GIT_URL`,
`OPERATOR_STATE_GIT_BRANCH`, `OPERATOR_STATE_GIT_TOKEN`, and
`OPERATOR_STATE_MODE=git-authoritative` into the `dtf-operator` container.
The runtime reads `defender.maxVetoDelayPlusPeriod` and
`defender.supportedGovernorImplementations` directly from the mounted YAML,
then resolves the corresponding internal environment contract. The Proposer
workflow does not consume Defender authority inputs. The journal schema remains
owned by the runtime; use only a digest-pinned image that implements atomic
Defender-journal coordination.

## Image Selection

The legacy `stable` channel is not valid for authoritative operation. Required
runtime image form:

```text
ghcr.io/reserve-protocol/dtf-operator@sha256:<journal-compatible-digest>
```

Set `DTF_OPERATOR_JOURNAL_IMAGE` to a digest-pinned image that implements the
runtime integration described above. The workflows reject tags, including
`stable`, `main`, `sha-*`, and `v*`, in authoritative mode.

The workflows separately pin the tested CLIProxyAPI sidecar by immutable image digest. Leave `CLI_PROXY_IMAGE` unset for that default. Set it only when deliberately testing another digest; authoritative workflows reject mutable tags.

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
INFERENCE_API_KEY
```

Scout ETL endpoint, provider, and short-term API key access are owned by the published runtime image. Forks do not configure Scout ETL.

Required repository variable to enable workflows:

```text
DTF_ACTIONS_ENABLED=true
```

Optional repository variables:

```text
DTF_OPERATOR_JOURNAL_IMAGE
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

`INFERENCE_API_KEY` is not an OpenAI key and does not grant access to a model account. In GitStore mode it is the sidecar management password; the Git-backed config keeps `api-keys` empty so no gateway secret is committed. Proxy access is limited to the job's private Docker network and its localhost-bound runner port. Both setup helpers generate and upload a fresh value when the environment variable `INFERENCE_API_KEY` is unset; set that environment variable only when you deliberately want to choose the value.

`CLIPROXY_AUTH_TGZ_B64` was the legacy cache bootstrap mechanism. It is not
read by authoritative workflows. Use `scripts/seed-cliproxy-gitstore` for the
explicit one-time CPA migration instead.

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

Required Defender veto signer:

```text
DEFENDER_KEYSTORE_JSON_B64
DEFENDER_KEYSTORE_PASSWORD
DEFENDER_FOUNDRY_ACCOUNT
```

The authoritative workflows require this signer to differ from the Proposer signer. Alert-only operation can still be run outside the authoritative live workflow, but it is not a substitute for a configured veto signer at cutover.

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
  # Total lifetime, vetoDelay + vetoPeriod; not vetoPeriod alone.
  maxVetoDelayPlusPeriod: 14d
  # Mandatory allowlist; replace placeholders with verified deployed values.
  supportedGovernorImplementations:
    - chainId: 8453
      implementationAddress: "0x..."
      codeHash: "0x..."
  proposalScan:
    fromBlock: 12345678
  requireAiApproval: true
  alertOnlyWithoutSigner: true
  inference:
    model: gpt-5.5
    reasoningEffort: medium
```

`defender.maxVetoDelayPlusPeriod` defaults to exactly `14d` and means the
maximum total proposal lifetime, `vetoDelay + vetoPeriod`, not `vetoPeriod`
alone. The runtime resolves that default to 1,209,600 seconds and uses it both
to reject unsupported governor timing and to bound normal proposal discovery.

`defender.supportedGovernorImplementations` is mandatory in authoritative
mode. It is a chain-scoped allowlist of verified implementation addresses and
runtime code hashes, not Folio governor proxy addresses. Include every
supported implementation on every chain Defender protects. The runtime rejects
an empty, malformed, or duplicate allowlist before scanning or acting.

Top-level `folioAddress` and `chainId` always identify the Proposer target. They are also the backward-compatible single-Folio Defender target when `defender.folios` is omitted or empty. `proposer.rebalanceCadence` is the deterministic auction cadence. For example, `30d` means the next rebalance auction should start at least 30 days after the first auction from the latest successful rebalance.

`defender.frequencyMinutes` is configurable and may be raised above `30`. It gates the workflow's 15-minute wakeups; it is not a hard maximum delay. GitHub Actions schedules are best-effort, so an eligible run can start later than the configured interval during platform congestion. Defender no longer waits behind the Proposer lock when the documented independent OAuth and signer requirements are met.

Set `proposer.proposalScan.fromBlock` before enabling scheduled runs. In legacy single-Folio Defender mode, also set `defender.proposalScan.fromBlock`; in multi-Folio mode, set `proposalScan.fromBlock` on every `defender.folios` entry instead. Defender treats this as a deployment floor and normally starts at the later of that block and the conservative timestamp-derived lifetime cutoff. Explicit exact-proposal recovery may scan outside the normal discovery window.

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
  maxVetoDelayPlusPeriod: 14d
  supportedGovernorImplementations:
    - chainId: 56
      implementationAddress: "0x<BNB_GOVERNOR_IMPLEMENTATION>"
      codeHash: "0x<BNB_GOVERNOR_IMPLEMENTATION_CODE_HASH>"
    - chainId: 8453
      implementationAddress: "0x<BASE_GOVERNOR_IMPLEMENTATION>"
      codeHash: "0x<BASE_GOVERNOR_IMPLEMENTATION_CODE_HASH>"
  requireAiApproval: true
  alertOnlyWithoutSigner: true
  inference:
    model: gpt-5.5
    reasoningEffort: medium
```

Every entry requires the Folio's current trading governor address and an independently chosen scan start block. The runtime verifies the configured governor against the Folio before evaluating or broadcasting a veto. Configure the chain-specific RPC Secret or Variable for every chain represented in the list.

Multi-Folio Defender remains one GitHub Actions job and invokes `node dist/cli/run-github-defender.js` once. The journal-compatible runtime keeps each Folio's review state isolated in the private operator-state Git journal and serializes signing. Do not create a workflow matrix or a job per Folio. Each job has one sidecar and one dedicated CPA GitStore; it does not use an Actions cache.

## Proposer Setup

Authenticate CLIProxyAPI before the one-time `scripts/seed-cliproxy-gitstore` migration. The older setup helpers still support legacy test forks, but their archive upload is not consumed by authoritative workflows.

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

Authoritative onchain veto mode uses a distinct Defender signer. Generate its keystore and upload the veto secrets:

```bash
RESEND_API_KEY=<key> scripts/setup-github-defender \
  --email alerts@example.com \
  --generate-keystore-dir ~/.dtf-operator/example/defender-keystores \
  --keystore-password-file ~/.dtf-operator/example/defender-password \
  --init-keystore-password \
  --foundry-account example-defender \
  --repo YOUR_GITHUB_OWNER/YOUR_PRIVATE_FORK
```

Delegate enough optimistic voting weight to the printed Defender signer address for every distinct governance voting token used by the configured governors. This does not grant proposer authority. Multi-Folio operation still uses this one Defender signer Secret set; it does not select a different signer per Folio.

Only proposals authoritatively classified by the governor as optimistic and containing a configured target Folio's `startRebalance()` call are vetted. Standard and non-rebalance proposals are archived and skipped. The journal-backed signed review ledger keeps review state isolated by Folio and prevents invoking Codex more than once for the same optimistic proposal during normal scheduled operation.

### Pending Veto Transaction Recovery

An unresolved veto transaction intentionally blocks all further Defender signer use across every configured Folio. Before first broadcast, the runtime persists the exact signed transaction bytes in encrypted form. Normal runs rebroadcast those same bytes when appropriate and verify the canonical transaction, receipt, finality depth, governor safety, and vote effect before releasing the signer-wide block.

There is no manual abandonment escape hatch. A missing receipt or temporarily unknown RPC outcome remains blocking. Replacement support must preserve the signer, chain, nonce, recipient, value, calldata, and proposal authorization identity; automated fee bumping is intentionally not part of this cutover.

## Signer Configuration

Use distinct Proposer and Defender accounts in authoritative operation. The auction launcher may reuse the Proposer account unless an operational split is required:

- Proposer account: `FOUNDRY_ACCOUNT`, `PROPOSER_KEYSTORE_JSON_B64`, `PROPOSER_KEYSTORE_PASSWORD`.
- Auction-launcher account: `AUCTION_LAUNCHER_FOUNDRY_ACCOUNT`, `AUCTION_LAUNCHER_KEYSTORE_JSON_B64`, `AUCTION_LAUNCHER_KEYSTORE_PASSWORD`.
- Defender account: `DEFENDER_FOUNDRY_ACCOUNT`, `DEFENDER_KEYSTORE_JSON_B64`, `DEFENDER_KEYSTORE_PASSWORD`.

The setup helpers support existing keystores with `--keystore` and `--keystore-password-file`, or new keystores with `--generate-keystore-dir`. They can import a private key from an environment variable with `--import-private-key-env <ENV_VAR>`.

## Operational Safety

- Keep workflows off `pull_request` and never use `pull_request_target` for signer or Codex automation.
- Run scheduled and manual workflows only from the fork default branch after GitHub Actions is enabled.
- Use `DTF_ACTIONS_ENABLED=true` as the final arming switch after config and Secrets are ready.
- Set `DTF_OPERATOR_JOURNAL_IMAGE` to a journal-compatible digest; mutable channels are rejected.
- Proposer and Defender have independent signer locks and separate CPA GitStores/OAuth logins so Defender cannot be starved by a long Proposer job. If either resource is shared, use one external writer coordinator instead.
- A multi-Folio Defender run is intentionally one job with one runtime invocation and serial signing. Its review/journal state belongs in the private operator-state Git repository.
- Do not expect this repository's Defender workflow to catch this repository's own Proposer workflow in parallel. Live Proposer safety must come from its own preflight checks, signer controls, and proposer self-check before broadcast.
- Treat each CPA GitStore and OAuth login as dedicated to one workflow. Refresh-token rotation makes a copied or shared OAuth record unsafe.
- Keep this repository private because workflow artifacts and config can reveal operational details even when Secrets are not committed.

## Source Repository

Runtime source, image builds, tests, and local Docker Compose helpers live in `reserve-protocol/dtf-operator`. Open PRs there only for shared source, image, runtime-contract, or local-runner changes. Do not open per-operator config PRs upstream.
