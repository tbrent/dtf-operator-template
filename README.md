# DTF Operator Template

Fork this repository into a private GitHub repository to operate a DTF/Folio.
The fork owns public operator configuration and GitHub Actions orchestration.
Credentials remain in GitHub Secrets or local keystores.

## Runtime image

Both workflows pin the cache-authoritative `dtf-operator` runtime by immutable
GHCR digest and verify its exact `org.opencontainers.image.revision` label
before execution.

## Production model

Proposer and Defender remain separate serial jobs and use distinct EVM signers:

| Role | Inference | Persistent runtime state | Concurrency |
| --- | --- | --- | --- |
| Proposer | Direct HTTPS recommended; CLIProxy optional | Per-run artifacts | `dtf-proposer-<repository_id>` |
| Defender | Direct HTTPS recommended; CLIProxy optional | Rolling GitHub Actions cache | `dtf-defender-<repository_id>` |

When both roles use direct inference, no CPA/GitStore URL, branch, token, OAuth
identity, fingerprint, or isolation configuration is required. Defender
persistence also requires no GitStore. The Defender workflow runs only from the
private fork's default branch and coordinates all configured Folios serially.

## Inference for both roles

### Recommended: direct HTTPS

Set these repository values:

```text
Variable: PROPOSER_INFERENCE_MODE=direct
Variable: PROPOSER_INFERENCE_BASE_URL=https://<host>/<optional-prefix>/v1
Secret:   PROPOSER_INFERENCE_API_KEY=<provider API key>

Variable: DEFENDER_INFERENCE_MODE=direct
Variable: DEFENDER_INFERENCE_BASE_URL=https://<host>/<optional-prefix>/v1
Secret:   DEFENDER_INFERENCE_API_KEY=<provider API key>
```

The runtime sends OpenAI-compatible `POST <base-url>/chat/completions` requests
with the corresponding role's API key as `Authorization: Bearer`. Supported
examples include OpenRouter (`https://openrouter.ai/api/v1`) and a remotely hosted
[`router-for-me/CLIProxyAPI`](https://github.com/router-for-me/CLIProxyAPI)
endpoint. Neither endpoint needs CPA GitStore variables or OAuth files in the
operator repository.

Use each setup helper with a separate provider-issued key:

```bash
PROPOSER_INFERENCE_API_KEY='<proposer-provider-key>' \
scripts/setup-github-proposer \
  --inference-base-url 'https://<host>/v1' \
  --generate-keystore-dir ~/.dtf-operator/production/proposer-keystores \
  --keystore-password-file ~/.dtf-operator/production/proposer-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>

RESEND_API_KEY='<resend-key>' \
DEFENDER_INFERENCE_API_KEY='<defender-provider-key>' \
scripts/setup-github-defender \
  --email '<operator-alert-address>' \
  --inference-base-url 'https://<host>/v1' \
  --generate-keystore-dir ~/.dtf-operator/production/defender-keystores \
  --keystore-password-file ~/.dtf-operator/production/defender-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>
```

Both helpers validate HTTPS, reject credential-bearing URLs, upload API keys
through standard input, and set the matching role's inference mode and base URL.

### Optional Tailscale access

For a direct endpoint reachable only through a tailnet, set both Secrets:

```text
TS_OAUTH_CLIENT_ID
TS_OAUTH_SECRET
```

They are optional but must be configured together. Each direct-mode job uses
`tailscale/github-action@v4` with `tag:ci` before probing and calling its endpoint.
Create a Tailscale OAuth client with the writable `auth_keys` scope and
permission to issue `tag:ci` nodes. Tailscale changes only network access; it
does not introduce a GitStore requirement.

### Optional legacy CLIProxy GitStore

Operators retaining a local CLIProxy OAuth sidecar configure the matching role:

```text
Variable: PROPOSER_INFERENCE_MODE=cliproxy
Variables: CPA_PROPOSER_GITSTORE_URL, CPA_PROPOSER_GITSTORE_BRANCH
Variable: CPA_PROPOSER_OAUTH_IDENTITY
Secret: CPA_PROPOSER_GITSTORE_TOKEN
Secret: CPA_PROPOSER_OAUTH_FINGERPRINT
Secret: PROPOSER_INFERENCE_API_KEY

Variable: DEFENDER_INFERENCE_MODE=cliproxy
Variables: CPA_DEFENDER_GITSTORE_URL, CPA_DEFENDER_GITSTORE_BRANCH
Variable: CPA_DEFENDER_OAUTH_IDENTITY
Secret: CPA_DEFENDER_GITSTORE_TOKEN
Secret: CPA_DEFENDER_OAUTH_FINGERPRINT
Secret: DEFENDER_INFERENCE_API_KEY
```

Seed only the role or roles using CLIProxy. When both roles use CLIProxy, use
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
Partial legacy configuration fails loudly. New operators configure none of the
triplet and receive a fresh journal automatically.

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
  --inference-base-url 'https://<host>/v1' \
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
and saved cache generation. Only then enable schedules:

```bash
gh variable set DTF_SCHEDULES_ENABLED \
  --repo <owner/private-operator-fork> \
  --body true
```

## GitHub configuration contract

Required Defender Variables:

```text
DEFENDER_INFERENCE_MODE             # direct recommended; cliproxy optional
DEFENDER_INFERENCE_BASE_URL         # direct only
DEFENDER_SIGNER_ADDRESS
DTF_SCHEDULES_ENABLED               # true only after acceptance
```

Required Proposer Variables:

```text
PROPOSER_INFERENCE_MODE             # direct recommended; cliproxy optional
PROPOSER_INFERENCE_BASE_URL         # direct only
PROPOSER_SIGNER_ADDRESS
```

Required Defender Secrets:

```text
DEFENDER_INFERENCE_API_KEY
DEFENDER_KEYSTORE_JSON_B64
DEFENDER_KEYSTORE_PASSWORD
DEFENDER_FOUNDRY_ACCOUNT
RESEND_API_KEY
DEFENDER_EMAIL_TO
```

Required Proposer Secrets:

```text
PROPOSER_INFERENCE_API_KEY
PROPOSER_KEYSTORE_JSON_B64
PROPOSER_KEYSTORE_PASSWORD
PROPOSER_FOUNDRY_ACCOUNT
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
transaction creates the signer-wide recovery blocker.

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
