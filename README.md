# DTF Operator Template

This public operator repository owns public DTF configuration and GitHub Actions
orchestration. Credentials remain in GitHub Secrets or local keystores.
Public runs do not upload Proposer artifacts; compact step summaries remain public.

## Runtime image

Both workflows pin the cache-authoritative `dtf-operator` runtime by immutable
GHCR digest and verify its exact `org.opencontainers.image.revision` label
before execution.

## Production model

Proposer and Defender remain separate serial jobs and use distinct EVM signers:

This deployment hard-codes every scheduled and manual Proposer run to dry-run
mode. Defender remains independently scheduled and scans its five configured
Folio/governor pairs every 15 minutes.

| Role | Inference | Persistent runtime state | Concurrency |
| --- | --- | --- | --- |
| Proposer | Direct HTTPS recommended; CLIProxy optional | Public deployment: no uploaded artifacts | `dtf-proposer-<repository_id>` |
| Defender | Direct HTTPS recommended; CLIProxy optional | Rolling GitHub Actions cache | `dtf-defender-<repository_id>` |

When both roles use direct inference, no CPA/GitStore URL, branch, token, OAuth
identity, fingerprint, or isolation configuration is required. Defender
persistence also requires no GitStore. The Defender workflow runs only from the
repository's default branch and coordinates all configured Folios serially.

## Inference for both roles

### Recommended: direct HTTPS

Set these repository values:

```text
Variable: PROPOSER_INFERENCE_MODE=direct
Variable: PROPOSER_INFERENCE_PROVIDER=openai-compatible|anthropic
Secret:   PROPOSER_INFERENCE_BASE_URL=https://<host>/<optional-prefix>
Secret:   PROPOSER_INFERENCE_API_KEY=<provider API key>

Variable: DEFENDER_INFERENCE_MODE=direct
Variable: DEFENDER_INFERENCE_PROVIDER=openai-compatible|anthropic
Secret:   DEFENDER_INFERENCE_BASE_URL=https://<host>/<optional-prefix>
Secret:   DEFENDER_INFERENCE_API_KEY=<provider API key>
```

The workflows append `/v1` when the configured URL does not already end in it.
`openai-compatible` sends Chat Completions with Bearer authentication and
supports native OpenAI, OpenRouter (`https://openrouter.ai/api/v1`), and a
remotely hosted [`router-for-me/CLIProxyAPI`](https://github.com/router-for-me/CLIProxyAPI)
endpoint. `anthropic` sends native Messages requests with `x-api-key`,
`anthropic-version`, effort controls, and `output_config.format` structured
outputs. Use it with `https://api.anthropic.com/v1` or a Claude-backed remote
CLIProxyAPI endpoint. Direct endpoints need no CPA GitStore variables or OAuth
files in the operator repository.

Provider selection does not rewrite public model configuration. For Anthropic,
set the matching `proposer.inference.model` or `defender.inference.model` in
`.github/dtf-operator.yml` to a Claude model that supports structured outputs,
such as `claude-fable-5`. For OpenRouter, use its vendor-prefixed model ID.

Use each setup helper with a separate provider-issued key:

```bash
PROPOSER_INFERENCE_API_KEY='<proposer-provider-key>' \
scripts/setup-github-proposer \
  --inference-provider openai-compatible \
  --inference-base-url 'https://<host>' \
  --generate-keystore-dir ~/.dtf-operator/production/proposer-keystores \
  --keystore-password-file ~/.dtf-operator/production/proposer-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>

RESEND_API_KEY='<resend-key>' \
DEFENDER_INFERENCE_API_KEY='<defender-provider-key>' \
scripts/setup-github-defender \
  --email '<operator-alert-address>' \
  --inference-provider openai-compatible \
  --inference-base-url 'https://<host>' \
  --generate-keystore-dir ~/.dtf-operator/production/defender-keystores \
  --keystore-password-file ~/.dtf-operator/production/defender-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>
```

Both helpers validate HTTPS, reject URL userinfo, query parameters, and
fragments, upload API keys and base URLs through standard input, and set the
matching role's inference mode.

### Optional Tailscale access

For a direct endpoint reachable only through a tailnet, set both Secrets:

```text
TS_OAUTH_CLIENT_ID
TS_OAUTH_SECRET
```

They are optional but must be configured together. Each direct-mode job uses
`tailscale/github-action@v4` with `TAILSCALE_TAGS` when configured, defaulting
to `tag:github-actions`, before probing and calling its endpoint. Create a
Tailscale OAuth client with the writable `auth_keys` scope and permission to
issue that tag. Tailscale changes only network access; it does not introduce a
GitStore requirement.

### Optional legacy CLIProxy GitStore

Operators retaining a local CLIProxy OAuth sidecar configure the matching role:

```text
Variable: PROPOSER_INFERENCE_MODE=cliproxy
Variables: CPA_PROPOSER_GITSTORE_URL, CPA_PROPOSER_GITSTORE_BRANCH
Variable: PROPOSER_INFERENCE_PROVIDER       # openai-compatible for Codex; anthropic for Claude
Variable: CPA_PROPOSER_OAUTH_IDENTITY
Secret: CPA_PROPOSER_GITSTORE_TOKEN
Secret: CPA_PROPOSER_OAUTH_FINGERPRINT
Secret: PROPOSER_INFERENCE_API_KEY

Variable: DEFENDER_INFERENCE_MODE=cliproxy
Variables: CPA_DEFENDER_GITSTORE_URL, CPA_DEFENDER_GITSTORE_BRANCH
Variable: DEFENDER_INFERENCE_PROVIDER       # openai-compatible for Codex; anthropic for Claude
Variable: CPA_DEFENDER_OAUTH_IDENTITY
Secret: CPA_DEFENDER_GITSTORE_TOKEN
Secret: CPA_DEFENDER_OAUTH_FINGERPRINT
Secret: DEFENDER_INFERENCE_API_KEY
```

Seed only the role or roles using CLIProxy. The seeder accepts one Codex or
Claude OAuth record and configures the matching inference provider
automatically. Create the source record with CLIProxyAPI's `-codex-login` or
`-claude-login` flow. When both roles use CLIProxy, use
different logins and repositories and set
`CPA_OAUTH_ISOLATION=separate-gitstores-and-logins`:

```bash
GITSTORE_GIT_TOKEN='<defender-scoped-token>' \
scripts/seed-cliproxy-gitstore \
  --role defender \
  --oauth-identity '<defender-account-label>' \
  --cliproxy-auth-dir ~/.cli-proxy-api-defender \
  --git-url https://github.com/<owner>/<defender-cpa-repo>.git \
  --git-branch <defender-cpa-branch> \
  --repo <owner/private-operator-fork>
```

These GitStores contain only optional inference OAuth state. Defender journal
state never writes to them.

## Defender Actions-cache persistence

The workflow restores and saves only:

```text
defender-data/operator-state
```

The container mounts `defender-data` at `/data`, so every atomic journal write
immediately reaches the runner host. The cached directory contains the complete
Defender journal and matching checkpoint, including:

- AI review reuse;
- notification and inference-auth bookkeeping;
- proposal obligations and reorg-aware scan checkpoints;
- frequency gates and successful-rebalance context;
- encrypted signed-veto transaction outbox and finality state.

Per-Folio run output under `/data/defender-folios` is not authoritative and is
not cached.

Actions caches are immutable. Every run restores the newest cache matching:

```text
dtf-defender-state-v1-<repository_id>-
```

and saves a new generation keyed by repository ID, run ID, and run attempt. A
separate `actions/cache/save@v6` step runs after ordinary runtime failures when
the mounted journal still validates. State validation occurs before save, and
cleanup occurs after save.

Operational limitation: runner cancellation, runner loss, or cache-service
failure before upload can leave the previous generation as the newest state.
The next run resumes proposal discovery from that older reorg-aware checkpoint.
If a signed or submitted transaction existed only in the lost generation, a
later transaction may conflict, replace it, or revert. This is the accepted
tradeoff for the simpler cache-backed setup. Cache-save failures remain visible
as failed workflow runs.

### Existing Git journal migration

On the first cache miss only, the workflow checks the legacy triplet:

```text
CPA_DEFENDER_GITSTORE_URL
OPERATOR_STATE_GIT_BRANCH
CPA_DEFENDER_GITSTORE_TOKEN
```

When all three are present, it performs a read-only `run-defender-state restore`
into the mounted directory before starting cache-authoritative Defender. This
preserves reviews, checkpoints, notifications, and signed-veto recovery state.
`OPERATOR_STATE_GIT_BRANCH` is the explicit migration marker. When it is
absent, the workflow starts a fresh journal even if Defender uses the CPA URL
and token for CLIProxy inference. When the marker is present, a missing legacy
URL or token fails loudly.

After a migrated run successfully saves its first cache generation,
`OPERATOR_STATE_GIT_BRANCH` is no longer required. Keep the Defender CPA URL and
token only when `DEFENDER_INFERENCE_MODE=cliproxy` still uses them.

## Fresh setup

### 1. Fork and configure the public Folio settings

Fork this repository privately and edit [`.github/dtf-operator.yml`](.github/dtf-operator.yml):

- Set top-level `folioAddress` and `chainId` for Proposer.
- Set `proposer.optimisticProposerAddress` and `proposer.proposalScan.fromBlock`.
- Add every Defender target under `defender.folios`, including its
  `governorAddress` and `proposalScan.fromBlock`, or use the top-level target for
  single-Folio operation.
- Keep `defender.maxVetoDelayPlusPeriod` at `1209600s`.

The runtime rejects governors and proposals whose optimistic veto delay plus
voting period exceeds 1,209,600 seconds. Equality is accepted.

### 2. Configure Proposer

Configure direct Proposer inference and its signer:

```bash
PROPOSER_INFERENCE_API_KEY='<proposer-provider-key>' \
scripts/setup-github-proposer \
  --inference-base-url 'https://<host>' \
  --generate-keystore-dir ~/.dtf-operator/production/proposer-keystores \
  --keystore-password-file ~/.dtf-operator/production/proposer-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>
```

### 3. Configure Defender direct inference and signer

Use the direct setup command shown above. Delegate enough optimistic voting
power to the Defender signer for every configured governor and fund it on every
required chain. The template requires the Proposer and Defender EVM addresses
to differ.

### 4. Configure RPC and notifications

Use GitHub Secrets for authenticated RPC URLs:

```text
RPC_URL
MAINNET_RPC_URL
BSC_RPC_URL
BASE_RPC_URL
```

Defender live operation also requires:

```text
RESEND_API_KEY
DEFENDER_EMAIL_TO
DEFENDER_EMAIL_FROM  # optional
```

### 5. Publish and pin the new runtime

Replace the runtime publication placeholders only after the runtime CI has
published the reviewed `linux/amd64` image. Both workflows verify the OCI
revision label before execution.

### 6. Validate and run manually

```bash
scripts/validate-authoritative-template
scripts/validate-authoritative-template-regressions
```

Manual `workflow_dispatch` runs are available only from the private fork's
default branch. Run Defender once and confirm the workflow reports a validated
and saved cache generation. Then enable its independent schedule:

```bash
gh variable set DEFENDER_SCHEDULES_ENABLED \
  --repo <owner/private-operator-fork> \
  --body true
```

Proposer scheduling is independent and every scheduled or manual Proposer run
is hard-coded to dry-run. To enable scheduled dry-runs, set its schedule gate:

```bash
gh variable set PROPOSER_SCHEDULES_ENABLED \
  --repo <owner/private-operator-fork> \
  --body true
```

This deployment does not expose a live Proposer mode or pass proposer signing
material into the workflow.

## GitHub configuration contract

Required Defender Variables:

```text
DEFENDER_INFERENCE_MODE             # direct recommended; cliproxy optional
DEFENDER_INFERENCE_PROVIDER         # openai-compatible (default) or anthropic
DEFENDER_SIGNER_ADDRESS
DEFENDER_SCHEDULES_ENABLED          # true only after Defender acceptance
```

Required Proposer Variables:

```text
PROPOSER_INFERENCE_MODE             # direct recommended; cliproxy optional
PROPOSER_INFERENCE_PROVIDER         # openai-compatible (default) or anthropic
PROPOSER_SIGNER_ADDRESS
PROPOSER_SCHEDULES_ENABLED          # optional; enables scheduled Proposer runs
```

Required Defender Secrets:

```text
DEFENDER_INFERENCE_BASE_URL         # direct only; variable fallback is migration-only
DEFENDER_INFERENCE_API_KEY
DEFENDER_KEYSTORE_JSON_B64
DEFENDER_KEYSTORE_PASSWORD
DEFENDER_FOUNDRY_ACCOUNT
RESEND_API_KEY
DEFENDER_EMAIL_TO
```

Required Proposer Secrets:

```text
PROPOSER_INFERENCE_BASE_URL         # direct only; variable fallback is migration-only
PROPOSER_INFERENCE_API_KEY
```

Optional direct-network Secrets:

```text
TS_OAUTH_CLIENT_ID
TS_OAUTH_SECRET
```

CPA/GitStore variables and Secrets are required only for a role configured with
`*_INFERENCE_MODE=cliproxy`. `OPERATOR_STATE_GIT_BRANCH` is legacy-import-only
and is not part of fresh cache-backed setup.

## Multi-Folio Defender behavior

A non-empty `defender.folios` list replaces the top-level target for Defender
only. One GitHub Actions job scans governors by chain, durably records relevant
proposals before advancing checkpoints, and evaluates proposals serially with
one signer. Do not create a workflow matrix or one job per Folio.

Scheduled and manual Defender dispatches are one-shot. Failed proposals remain
retryable without blocking unrelated Folios unless an unresolved signed-veto
transaction creates the signer-wide recovery blocker. Proposal discovery and
durable queueing continue while signing is blocked. After five minutes, the
runtime may persist and broadcast a same-nonce, same-intent replacement with a
12.5% fee bump; unrelated signing resumes only after one transaction in the
replacement chain finalizes or reverts.

## Operational security

- Never print or commit provider keys, GitHub tokens, OAuth JSON, keystores,
  passwords, private keys, authenticated RPC URLs, Tailscale credentials,
  Resend credentials, or plaintext signed transactions.
- Keep direct inference URLs credential-free; authentication belongs in the
  separate `PROPOSER_INFERENCE_API_KEY` and `DEFENDER_INFERENCE_API_KEY` Secrets.
- Treat the Actions cache journal as private operational state. Signed veto
  bytes are encrypted, but cache access still grants operational metadata.
- Do not delete retained OAuth archives merely because either role moved to direct
  inference; archive deletion is a separate operator decision.
