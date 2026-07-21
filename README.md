# DTF Operator Template

Fork this repository into a private GitHub repository to operate a DTF/Folio.
The fork owns public operator configuration and GitHub Actions orchestration.
Credentials remain in GitHub Secrets or local keystores, while OAuth and
Defender state are persisted across two private Git repositories.

## Published operator identity

Every operator runtime execution is fixed to this reviewed production image:

```text
ghcr.io/reserve-protocol/dtf-operator@sha256:26e60f9ba0e4f4c968e7be3d705bbf93e5a03dd903f181856967bccd1b5afa07
```

The image must resolve to `linux/amd64`. Its
`org.opencontainers.image.revision` OCI label must equal:

```text
87bddadbc97eaa8dd006b8a048022db537c936c7
```

Both workflows pull that digest with an explicit platform and reject any
platform or source-revision mismatch. There is no image input or repository
variable override. The workflows do not check out operator source or build an
operator image per run.

## Production isolation model

Production separates the Proposer and Defender identities across two private
Git repositories and three logical Git write domains:

| Role | OAuth authority | Git write domain | EVM authority | Concurrency |
| --- | --- | --- | --- | --- |
| Proposer | Dedicated CPA login | `CPA_PROPOSER_GITSTORE_URL` + `CPA_PROPOSER_GITSTORE_BRANCH` | `PROPOSER_SIGNER_ADDRESS` | `dtf-proposer-<repository_id>` |
| Defender inference | Different dedicated CPA login | `CPA_DEFENDER_GITSTORE_URL` + `CPA_DEFENDER_GITSTORE_BRANCH` | None | `dtf-defender-<repository_id>` |
| Defender journal and veto | N/A | `CPA_DEFENDER_GITSTORE_URL` + `OPERATOR_STATE_GIT_BRANCH` | `DEFENDER_SIGNER_ADDRESS` | `dtf-defender-<repository_id>` |

The workflows require the two OAuth identity labels to differ, require
`CPA_OAUTH_ISOLATION=separate-gitstores-and-logins`, require separate Proposer
and Defender repositories, require distinct Defender OAuth and journal
branches, and require all three repository/branch write domains to be pairwise
distinct. The Proposer and Defender EVM addresses must also differ. If
infrastructure policy cannot provide these identity and write-domain
boundaries, do not enable production schedules.

Each CPA GitStore branch is authoritative only for its role's CLIProxy OAuth
records. A separate branch in the same private Defender repository is
authoritative only for the credential-free Defender journal. The Defender
workflow maps `CPA_DEFENDER_GITSTORE_URL` and `CPA_DEFENDER_GITSTORE_TOKEN` to
the runtime's `OPERATOR_STATE_GIT_URL` and `OPERATOR_STATE_GIT_TOKEN`, runs with
`OPERATOR_STATE_MODE=git-authoritative`, restores the journal before processing,
and CAS-persists journal mutations. Proposer artifacts remain per-run GitHub
artifacts; Actions cache is not an authority for OAuth, transaction state,
alert state, or Defender state.

The Defender OAuth branch and journal branch deliberately share one
repository-scoped token. Branch separation prevents data collision, but it is
not credential isolation: compromise of the Defender sidecar or token can
write either Defender branch. This is the explicit compactness tradeoff of the
two-repository model. Operators that require separate credential blast radii
must use branch-restricted credentials supported by their Git host or retain
separate repositories.

## Fresh production setup

The rollout supports fresh authoritative initialization only. It does not
import cached Defender files or provide a legacy state migration path.

### 1. Prepare the private repositories

Create or select two private repositories with three branches:

1. Proposer CPA GitStore repository and branch.
2. Defender repository with a CPA GitStore branch.
3. An absent Defender journal branch in that same Defender repository.

Use one scoped token for the Proposer repository and one scoped token for both
branches in the Defender repository. Never put a token, username/password pair,
or other credential in a Git URL. The Defender token can write both Defender
branches; scope it to no other repository. Protect both Defender branches with
policies that forbid force-pushes and deletion. Apply equivalent
least-privilege and retention controls to the Proposer CPA GitStore branch.

### 2. Create separate CPA OAuth logins

On a trusted operator workstation, complete the CLIProxy Codex login flow once
for Proposer and once for Defender. Use different OAuth accounts and separate
local auth directories. Choose non-secret identity labels that operators can
audit, such as distinct account aliases; the labels must represent the actual
different logins.

Seed the Proposer GitStore:

```bash
GITSTORE_GIT_TOKEN='<proposer-scoped-token>' \
scripts/seed-cliproxy-gitstore \
  --role proposer \
  --oauth-identity '<proposer-account-label>' \
  --cliproxy-auth-dir ~/.cli-proxy-api-proposer \
  --git-url https://github.com/<owner>/<proposer-cpa-repo>.git \
  --git-branch <proposer-cpa-branch> \
  --repo <owner/private-operator-fork>
```

Seed the Defender GitStore with the different login in the Defender repository:

```bash
GITSTORE_GIT_TOKEN='<defender-scoped-token>' \
scripts/seed-cliproxy-gitstore \
  --role defender \
  --oauth-identity '<defender-account-label>' \
  --cliproxy-auth-dir ~/.cli-proxy-api-defender \
  --git-url https://github.com/<owner>/<defender-repo>.git \
  --git-branch <defender-cpa-branch> \
  --repo <owner/private-operator-fork>
```

The helper requires exactly one Codex OAuth JSON record per role, refuses a
target that already contains OAuth records, uploads each role's token through
standard input, and verifies that the record survives a clean GitStore clone.
It hashes the stable OAuth `account_id` and email into a role-specific GitHub
Secret without printing either field; workflows reject equal fingerprints. It
does not alter the source OAuth directory.

Do not delete retained OAuth archives, including an existing
`CLIPROXY_AUTH_TGZ_B64` secret or offline encrypted copies, during rollout.
The production workflows do not consume them, but removal is a separate
operator decision after production acceptance and backup retention are
confirmed. The setup helpers never delete GitHub Secrets or local archives.

### 3. Initialize the Defender journal

The target journal branch must not exist. Initialize an empty authoritative
journal from the exact published image:

```bash
CPA_DEFENDER_GITSTORE_TOKEN='<same-defender-scoped-token>' \
scripts/initialize-defender-journal \
  --git-branch <defender-journal-branch> \
  --repo <owner/private-operator-fork>
```

The initializer reads `CPA_DEFENDER_GITSTORE_URL` from the operator fork,
reuses `CPA_DEFENDER_GITSTORE_TOKEN`, and sets only
`OPERATOR_STATE_GIT_BRANCH`; there is no duplicate journal URL or token
configuration. It imports no prior runtime files, refuses existing branch
state, performs a clean authoritative restore and validation, and never
force-pushes or deletes a private branch. If initialization stops after its
first persisted generation, it reports that the branch was intentionally left
intact; inspect and resolve that state explicitly rather than deleting it
automatically.

### 4. Configure distinct EVM signers

Generate or upload the Proposer signer:

```bash
scripts/setup-github-proposer \
  --generate-keystore-dir ~/.dtf-operator/production/proposer-keystores \
  --keystore-password-file ~/.dtf-operator/production/proposer-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>
```

The helper sets `PROPOSER_KEYSTORE_JSON_B64`,
`PROPOSER_KEYSTORE_PASSWORD`, `PROPOSER_FOUNDRY_ACCOUNT`,
`PROPOSER_INFERENCE_API_KEY`, and the `PROPOSER_SIGNER_ADDRESS` repository
variable. Fund the address and grant the optimistic proposer permissions needed
for the configured Folio. Configure a separate auction-launcher keystore only
when the deployment intentionally separates that role.

Generate or upload the different Defender signer:

```bash
RESEND_API_KEY='<resend-key>' \
scripts/setup-github-defender \
  --email '<operator-alert-address>' \
  --generate-keystore-dir ~/.dtf-operator/production/defender-keystores \
  --keystore-password-file ~/.dtf-operator/production/defender-password \
  --init-keystore-password \
  --repo <owner/private-operator-fork>
```

The helper sets `DEFENDER_KEYSTORE_JSON_B64`,
`DEFENDER_KEYSTORE_PASSWORD`, `DEFENDER_FOUNDRY_ACCOUNT`,
`DEFENDER_INFERENCE_API_KEY`, email Secrets, and the
`DEFENDER_SIGNER_ADDRESS` repository variable. Delegate enough optimistic
voting power to this signer for every configured governor and fund it on every
required chain. It must not be the Proposer signer.

The two role-specific inference keys are gateway credentials, not CPA OAuth
identities. CPA identity isolation comes from the two independently created
OAuth logins and their dedicated GitStores.

### 5. Configure Folios and RPC access

Edit [`.github/dtf-operator.yml`](.github/dtf-operator.yml):

- Set top-level `folioAddress` and `chainId` for Proposer.
- Set `proposer.optimisticProposerAddress` to the configured Proposer signer.
- Set `proposer.proposalScan.fromBlock` before production operation.
- Add every Defender target under `defender.folios`, including its current
  `governorAddress` and `proposalScan.fromBlock`, or use the top-level target for
  single-Folio operation.
- Keep `defender.maxVetoDelayPlusPeriod` at `1209600s`, exactly 14 days.

Configure `RPC_URL`, `MAINNET_RPC_URL`, `BSC_RPC_URL`, or `BASE_RPC_URL` as
GitHub Secrets whenever the URL contains authentication. Never store an
authenticated RPC URL in a repository variable, workflow, config file, log, or
artifact.

The runtime reads governor behavior onchain and rejects any optimistic veto
delay plus voting period above the configured 1,209,600-second maximum.
Equality is accepted. This runtime release does not use a repository-configured
governor implementation allowlist.

## Multi-Folio Defender behavior

A non-empty `defender.folios` list replaces the top-level target for Defender
only. Proposer still operates the top-level Folio. Example:

```yaml
folioAddress: "<PROPOSER_FOLIO_ADDRESS>"
chainId: 8453

defender:
  maxVetoDelayPlusPeriod: 1209600s
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
```

Defender remains one GitHub Actions job and invokes
`node dist/cli/run-github-defender.js` once. The runtime selects its serial
multi-Folio coordinator, isolates each Folio's review state inside the one
authoritative journal, and serializes use of the one Defender signer. Do not
create a workflow matrix or a job per Folio. Scheduled and manual Defender
dispatches are one-shot; continuous polling is not part of the template surface.

## GitHub configuration contract

Repository Variables:

```text
CPA_PROPOSER_GITSTORE_URL
CPA_PROPOSER_GITSTORE_BRANCH
CPA_PROPOSER_OAUTH_IDENTITY
CPA_DEFENDER_GITSTORE_URL
CPA_DEFENDER_GITSTORE_BRANCH
CPA_DEFENDER_OAUTH_IDENTITY
CPA_GITSTORE_GIT_USERNAME          # optional
CPA_OAUTH_ISOLATION                # separate-gitstores-and-logins
OPERATOR_STATE_GIT_BRANCH
PROPOSER_SIGNER_ADDRESS
DEFENDER_SIGNER_ADDRESS
DTF_SCHEDULES_ENABLED              # set to true only after acceptance
```

Repository Secrets:

```text
CPA_PROPOSER_GITSTORE_TOKEN
CPA_DEFENDER_GITSTORE_TOKEN
CPA_PROPOSER_OAUTH_FINGERPRINT
CPA_DEFENDER_OAUTH_FINGERPRINT
PROPOSER_INFERENCE_API_KEY
DEFENDER_INFERENCE_API_KEY
PROPOSER_KEYSTORE_JSON_B64
PROPOSER_KEYSTORE_PASSWORD
PROPOSER_FOUNDRY_ACCOUNT
DEFENDER_KEYSTORE_JSON_B64
DEFENDER_KEYSTORE_PASSWORD
DEFENDER_FOUNDRY_ACCOUNT
RESEND_API_KEY
DEFENDER_EMAIL_TO
DEFENDER_EMAIL_FROM                # optional
RPC_URL / MAINNET_RPC_URL / BSC_RPC_URL / BASE_RPC_URL
```

Auction-launcher Secrets are optional and remain separate from the required
Proposer and Defender identities:

```text
AUCTION_LAUNCHER_KEYSTORE_JSON_B64
AUCTION_LAUNCHER_KEYSTORE_PASSWORD
AUCTION_LAUNCHER_FOUNDRY_ACCOUNT
```

## Validation and rollout

Run the authoritative validator and its focused negative regressions:

```bash
scripts/validate-authoritative-template
scripts/validate-authoritative-template-regressions
```

The validator checks the exact operator digest and revision, `linux/amd64`
selection, two private repositories with three independent Git domains, two
OAuth identities, distinct EVM signers and concurrency groups,
Git-authoritative Defender mode, the schedule gate, one serial Defender
coordinator, the exact 14-day maximum, and removal of cache-backed,
mutable-image, disposable-branch, source-build, and obsolete governor
configuration.

Manual `workflow_dispatch` runs are available in a private fork even when
`DTF_SCHEDULES_ENABLED` is absent or false. Start with Proposer `dry-run`, then
`observe-only`. Review preflight output and artifacts before confirming a
`live-one-cycle` dispatch. Run Defender once and verify that its journal advances
in the protected private branch. These manual checks do not enable cron.

Only after signer authorities, RPCs, OAuth restoration, journal persistence,
notification routing, and manual behavior are accepted should an operator set:

```bash
gh variable set DTF_SCHEDULES_ENABLED \
  --repo <owner/private-operator-fork> \
  --body true
```

Removing or changing that variable disables future cron jobs without disabling
manual dispatch.

## Operational security

- Never print or commit GitHub tokens, OAuth JSON, keystores or passwords,
  private keys, authenticated RPC URLs, Resend credentials, inference gateway
  keys, or plaintext signed transactions.
- Never put credentials in Git URLs. All helper scripts send GitHub Secrets
  through standard input and pass Git credentials through environment-backed
  askpass handling.
- Treat OAuth files and retained archives as credentials. Keep role-specific
  copies separate and encrypted at rest.
- Treat the Defender journal as private operational state. The runtime rejects
  secret values and plaintext signed transactions in journal content, but the
  repository still requires strict access control and retention.
- Do not force-push or delete any persistence branch during routine operation.
  Resolve conflicts and incomplete initialization explicitly.
