# Collaborative coding app implementation plan

Written on November 17, 2026 against upstream commit `7219841f7b8b3cec6dc274398985877d7dee8039`.

Build a shared coding environment for Lucas and teammates using this fork, Daytona, Codex, and
Claude Code. A chat owns one remote checkout, one native agent conversation, and the subscription
account selected when the chat starts. Every participant's prompts enter that conversation with
their authorship recorded. The account binding persists when another teammate contributes.

The first usable release runs through the existing web interface. Lucas will direct a full frontend
redesign later. Desktop packaging follows that design. Early changes should expose the backend
capabilities through existing controls, artifacts, and the desktop viewer, with only the functional
UI needed to exercise them.

## Implementation decisions

- Start with the existing Cloudflare control plane and web deployment, and choose Daytona as the
  sandbox provider. This is the proposed hosting default from the architecture discussion. Account,
  domain, and resource sizing are deployment inputs.
- Reuse session queues, event persistence, participant authorization, diffs, native agent adapters,
  repository image builds, and PR artifacts. Extend the owner of each behavior.
- Use the creator's explicitly selected subscription for the entire chat. Attribute prompts and
  publishing actions separately from model usage. Subscription exhaustion produces a recoverable
  waiting state with the provider's reason.
- Give each chat an isolated repository checkout. Graphite can create several branch worktrees
  within that checkout's Git repository. A session coordinator serializes publishing and checkout
  changes against agent turns and manual edits.
- Keep the shared Chromium instance beside the agent in Daytona. Human and agent browser actions
  operate on the same profile and tabs.
- Use TTD's existing PR preview deployment and authentication mechanisms. Keep TTD-specific setup
  and credentials in the private TTD repository and deployment configuration. This public fork holds
  generic integration contracts.
- Deliver each numbered PR as one commit and one independently explainable concern. Aim for roughly
  250 reviewable changed lines, splitting further where a boundary needs more space. Include the
  behavioral regression with the behavior it protects.

## Existing implementation to reuse

| Concern                      | Owning implementation                                                                                                                                                                                                                              | Consequence for this plan                                                                                                                                                                         |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Daytona creation and restart | [Provider](../../packages/control-plane/src/sandbox/providers/daytona-provider.ts), [Terraform module](../../terraform/modules/daytona-infra/main.tf)                                                                                              | The provider advertises persistent resume. Session snapshot capture and restore are currently disabled in its capabilities. Validate stop/start first.                                            |
| Runtime image dependencies   | [Image build package](../../packages/sandbox-images/README.md), [tool pins](../../packages/sandbox-images/toolchain.json)                                                                                                                          | Build and verify the shared base image through the existing provider adapter.                                                                                                                     |
| Per-repository preparation   | [Repository hooks](../../packages/sandbox-runtime/src/sandbox_runtime/repository_hooks.py), [boot coordinator](../../packages/sandbox-runtime/src/sandbox_runtime/repository_boot.py)                                                              | `.openinspect/setup.sh` prepares dependencies and `.openinspect/start.sh` starts session services.                                                                                                |
| Subscription selection       | [Account resolution](../../packages/control-plane/src/session/provider-account-resolution.ts), [account guide](../OPENAI_MODELS.md)                                                                                                                | Selection is already persisted per session. Existing defaults can select API authentication, which needs a deployment policy for this product.                                                    |
| Shared chat                  | [Session architecture](../HOW_IT_WORKS.md#prompt-queuing)                                                                                                                                                                                          | Follow-ups queue behind the active turn and events stream to participants.                                                                                                                        |
| Changes                      | [Diff collector](../../packages/sandbox-runtime/src/sandbox_runtime/diff_collector.py), [refresh worker](../../packages/sandbox-runtime/src/sandbox_runtime/diff_capture.py)                                                                       | Reuse the existing uncommitted-file handling and idle refresh.                                                                                                                                    |
| Browser desktop              | [Desktop process owner](../../packages/sandbox-runtime/src/sandbox_runtime/browser_desktop.py), [viewer link](../../packages/web/src/components/sidebar/vnc-section.tsx)                                                                           | VNC exists. The inspected browser configuration selects a Chrome executable, so shared visible-browser attachment still needs verification and implementation.                                    |
| PR creation and state        | [Publishing service](../../packages/control-plane/src/session/pull-request-service.ts), [PR store](../../packages/control-plane/src/db/session-pull-request-store.ts), [refresh](../../packages/control-plane/src/session/pull-request-refresh.ts) | Multiple PRs and lifecycle refresh already exist. The current creation path pushes `HEAD` and creates a PR through the source-control provider. Graphite publication needs its own dispatch path. |

The root [AGENTS.md](../../AGENTS.md) describes the data plane primarily as Modal. The provider code
and [deployment guide](../GETTING_STARTED.md#daytona) establish the Daytona path used here.

## Stack sequence

Each stack begins after its prerequisites have landed. Keep active stacks short and merge completed
prefixes through the repository's normal review process. The numbers below identify planned work,
rather than existing PR numbers. Suggested branch names use `codex/` plus the title's subject.

| Stack                               | Planned PRs | Prerequisites                                             | Observable milestone                                                             |
| ----------------------------------- | ----------- | --------------------------------------------------------- | -------------------------------------------------------------------------------- |
| A — Daytona setup                   | 01–03       | Deployment accounts and selected resource names           | A private sandbox boots and preserves files across restart.                      |
| B — TTD workspace                   | 04–05       | A and GitHub App access to TTD                            | A fresh TTD checkout opens with working dependencies.                            |
| C — Subscriptions and collaboration | 06–09       | A, with provider integration decisions resolved for 07–08 | Two people continue the same conversation on its selected subscription.          |
| D — Shared browser                  | 10–13       | A and C                                                   | Both people and the agent operate the same browser with an explicit input owner. |
| E — TTD preview integration         | 14–17       | B, C, D, and an existing disposable TTD preview           | The agent can verify working-copy and deployed UI changes with a teammate.       |
| F — Graphite publishing             | 18–22       | B and C, plus E for automatic preview association         | A session publishes a small stack and tracks its top preview.                    |
| G — Frontend redesign and desktop   | Deferred    | Backend capabilities and Lucas's design direction         | A redesigned interface and macOS application.                                    |

Stack F can start alongside E after its prerequisites land. E initially accepts an existing PR
explicitly, so preview validation can proceed before automatic publishing is available.

## Stack A — Daytona setup

### PR 01 — Add this deployment's Daytona configuration example

Add a named, non-secret example beside the existing
[Terraform inputs](../../terraform/environments/production/terraform.tfvars.example) and a short
deployment guide. Set `sandbox_provider = "daytona"`, `web_platform = "cloudflare"`, team
enforcement, and explicit bot enablement. Start with web sessions and the optional Slack and Linear
classifiers disabled so the baseline can use subscription credentials alone.

- Record required Cloudflare, GitHub App, and Daytona credentials by setting name, required
  permission, and storage location. Values belong in ignored configuration or the deployment secret
  store.
- Select a unique sandbox image prefix and explicit memory, idle-stop, and archive settings from the
  existing configuration surface. Measure TTD memory use before settling on its larger image size.
- Validate formatting, Terraform input constraints, and that the planned resources use Daytona and
  the intended Cloudflare account.
- Acceptance is a reviewed configuration and resource plan that can build the first sandbox.
  Provisioning remains a deployment operation following the configured plan.

### PR 02 — Give the fork its own deployment state and workflow configuration

Wire the fork's state bucket/key and deployment variables through the existing
[Terraform workflow](../../.github/workflows/terraform.yml). Its current backend has fixed bucket
and key defaults, so local and CI initialization must select the same fork-owned state explicitly.
Reuse the current D1 and Durable Object bootstrap sequence.

- Reuse existing secrets checks, plan/apply jobs, image verification, and deployment outputs. Pass
  the selected state identity consistently in every initialization path.
- Test the workflow input contract and missing-configuration behavior. Keep secrets out of workflow
  output and saved plans exposed publicly.
- Acceptance is a deployment of the stock web app and control plane, owner/team bootstrap, and
  successful authorized sign-in using the published URL.
- Recovery is the prior application revision and retained image reference. Database changes follow
  forward-compatible migration rules.

### PR 03 — Add a bounded Daytona lifecycle smoke check

Extend existing provider verification with a session-level check for create, stop, start, and
cleanup. Reuse
[Daytona lifecycle handling](../../packages/control-plane/src/sandbox/daytona-lifecycle.ts) and
provider tests. Create a small temporary file and Git change, restart the same sandbox, and check
their contents and baseline revision.

- Exercise browser/terminal access renewal after restart and an interrupted start attempt. Persist
  the provider sandbox ID and report cleanup failures with that ID.
- Distinguish browser disconnect, idle stop, and worker failure. Any retry must reconcile the
  existing sandbox before creating another.
- Record boot duration, restart duration, resource configuration, and outcome using existing logs.
  This is a single bounded lifecycle run.
- Acceptance is preserved committed and uncommitted files, refreshed access, and cleanup of the
  check's temporary resources. Native agent-history recovery is covered in PR 09.

## Stack B — TTD workspace

### PR 04 — Prepare TTD through repository setup hooks

This is a companion PR in **ProDex-Labs/ttd**. Add `.openinspect/setup.sh` using TTD's pinned
toolchain and workspace resolver. The selected TTD revision's `.mise.toml`, Python workspace, and
frontend lockfiles own its dependencies. Open-Inspect's infrastructure Python and Node remain
available to its own worker.

- Reuse TTD's supported install steps and explicit Python pin. Read repository instructions before
  choosing which app dependencies to prepare.
- Verify a clean clone can import the workspace packages and start the frontend toolchain. Run a
  small representative lint/type check and unit test from normal entry points.
- Classify Docker-dependent integration tests separately and verify whether the chosen Daytona
  sandbox can support their required services. Record that capability before adding it to the
  advertised checks.
- Acceptance is a runnable TTD checkout through the existing repository hook lifecycle. TTD
  instructions, code, and credentials remain in its private repository.

### PR 05 — Configure TTD repository images and rebuild rules

Use the existing [image build workflow](../IMAGE_PREBUILD.md) to prepare a TTD image and enable
Daytona prebuilds for the deployment after validating a fresh restore. Store its repository revision
and verified image reference through existing records.

- Exercise setup from a clean base, startup from the prepared image, and fallback when an image is
  unavailable.
- Rebuild prepared images when the base toolchain or repository setup changes. The shared base-image
  build alone does not refresh repository images.
- Verify clones use fresh scoped GitHub credentials and that captured images contain the intended
  source/dependencies with credentials excluded.
- Acceptance is measured first-session readiness and a successful session started from the prepared
  TTD image. Reverting image selection to the preceding verified reference is the recovery path.

## Stack C — Subscriptions and collaboration

### PR 06 — Require explicit subscription binding for these coding sessions

Add a deployment policy at
[provider account resolution](../../packages/control-plane/src/session/provider-account-resolution.ts)
that requires a usable subscription connection for Codex and Claude sessions. Use the existing
account picker and persisted session binding. The creator selects their account when starting the
chat, and collaborators continue that binding.

- Validate the choice at session creation and revalidate availability before executing a queued
  turn. Cover disabled, expired, removed, and quota-limited connections.
- Preserve prompt author, session creator, and selected account as distinct facts. Return actionable
  connection/usage status through existing event and error surfaces.
- Test that participant changes and installation-default changes preserve an existing chat's
  account. Missing subscription credentials must prevent inference from silently selecting a paid
  API fallback.
- Acceptance is predictable account selection and a recoverable failure before model execution when
  the required connection is unavailable.

### PR 07 — Implement the supported Codex subscription login path

Use the existing
[Codex adapter](../../packages/sandbox-runtime/src/sandbox_runtime/harness/codex/__init__.py) and
account integration. Resolve the applicable hosted/self-hosted integration with OpenAI before
choosing the credential mechanism. The current OpenAI docs limit built-in app-server login for
hosted services and describe a separate Sign in with ChatGPT path.

- If that path applies, implement registration, protected token storage, serialized refresh, and the
  documented app-server provider configuration in the existing account adapter boundary.
- Preserve the native thread ID during access-token renewal. With the documented environment-token
  setup, restart app-server with the renewed token and resume the saved thread at a controlled turn
  boundary.
- Cover logout/revocation, expired credentials, model entitlement failure, and a provider limit
  reached during a turn. Retain a clear terminal or resumable state.
- Acceptance is one real subscription-backed Codex task followed by a refreshed-token continuation,
  with the selected account and usage attribution confirmed. Split credential lifecycle and runtime
  changes into adjacent PRs if review size requires it.

### PR 08 — Implement the supported Claude subscription login path

Use the published Claude Code binary and its official sign-in flow. Adapt the existing
[Claude adapter](../../packages/sandbox-runtime/src/sandbox_runtime/harness/claude.py) and
[environment preparation](../../packages/sandbox-runtime/src/sandbox_runtime/harness/claude_env.py)
to the approved hosted arrangement. Prefer direct login into the user's private CLI environment
where required by Anthropic's rules.

- Persist the CLI's private login state across sandbox stop/start. Keep login interaction out of
  shared chat history and browser recordings.
- Confirm the adapter's clean environment actually selects subscription auth. Exercise
  refresh/re-login and quota handling with the pinned SDK/binary.
- Treat central credential collection and sharing of a person's allowance as provider integration
  questions. The user's product decision is to pin the creator's account, while vendor permission
  remains an external dependency.
- Acceptance is a real Claude task and continuation drawing from the chosen subscription. Separate
  private login delivery from adapter changes if they form independently reviewable work.

### PR 09 — Verify multiplayer continuation and recovery

Extend the existing session integration and
[authenticated preview](../verification/authenticated-preview.md) coverage with two participants.
Send a second prompt while the first turn is active, reconnect one client, then stop and resume the
worker between turns.

- Check ordered prompt execution, authorship, one broadcast result per turn, consistent diffs, and
  the original provider-account binding for both harnesses.
- Exercise a disconnect after a prompt is accepted. Deduplicate the same client submission and
  reconcile an interrupted turn before any replay that could repeat edits or publication.
- Run the same bounded scenario on a real Daytona sandbox with both subscriptions. The local preview
  uses a sandbox fixture and proves the client/service contracts only.
- Acceptance is two people continuing the same native conversation, including uncommitted edits,
  after both client reconnect and a controlled sandbox restart.

## Stack D — Shared browser

### PR 10 — Start one visible Chromium profile per session

Extend
[browser desktop startup](../../packages/sandbox-runtime/src/sandbox_runtime/browser_desktop.py) and
its supervisor to own Chromium as well as the display/VNC processes. Use the pinned Chrome
executable and a session-specific persistent profile.

- Publish an internal browser connection descriptor through existing runtime state. Bind browser
  automation access to the sandbox and protect any externally reachable endpoint.
- Test browser startup, crash recovery, shutdown, and restart with retained profile data. Restrict
  profile captures to authorized session recovery.
- Acceptance is the existing Open Desktop viewer showing the owned browser and a page that survives
  a controlled browser restart where browser recovery permits it.

### PR 11 — Attach agent browser tools to that Chromium instance

Configure the existing `agent-browser` integration to attach to the owned Chromium instance and
verify the exact feature flags supported by its pinned release. Use the existing harness tool/skill
surfaces for both Codex and Claude.

- The browser command owner resolves the session's connection descriptor and returns current tab
  identity. Screenshots and navigation come from the visible instance.
- Verify the human opens a menu and the agent inspects that open menu, then the agent changes the
  page and the human sees the change.
- Test browser reconnect after restart and stale connection cleanup. Preserve existing
  screenshot/video artifact uploads.
- Acceptance is demonstrated shared URL, tab, cookies, and transient UI state through the current
  desktop viewer.

### PR 12 — Persist browser control ownership

Add a small session command/state contract for requesting and releasing browser control. Store the
participant or agent owner, ownership generation, and expiration in the session coordinator. Use
existing membership authorization and event broadcasting.

- Define disconnect and expiry behavior so a departed participant releases control predictably.
  Reject commands carrying an old ownership generation.
- Store the lease with the session, so a coordinator restart can reconcile it. Broadcast the current
  owner to reconnecting clients.
- Acceptance is a deterministic two-participant ownership test, including races, reconnects,
  release, and expiry.

### PR 13 — Apply browser ownership to human and agent input

Connect that state to the managed browser command path and authenticated desktop input. Provide a
minimal Take control / Release control action using the existing viewer surface.

- Before granting human control, pause or interrupt agent browser execution at a controlled boundary
  and drain tracked browser actions. The runtime must account for background browser commands as
  well as the main turn.
- Serve view-only input behavior to other viewers and validate ownership when admitting
  mouse/keyboard events. Route normal agent browser commands through the same ownership check.
- Treat this as coordination among authorized collaborators. Arbitrary shell/root access can bypass
  ordinary browser tooling, so tests must establish the managed interaction guarantee precisely.
- Acceptance is a teammate demonstrating a bug while the agent remains observant, followed by a
  successful handback and agent interaction. Full visual design is deferred to stack G.

## Stack E — TTD preview integration

### PR 14 — Associate an existing PR preview with a session

Add an explicit preview association using the existing session artifact machinery. Record repository
identity, PR number, target kind, requested head SHA, discovered preview URL, and observed status.
Validate association against the session's repository grants.

- Begin with an existing disposable preview selected by PR number. Derive its URL from trusted
  deployment output and the configured repository integration.
- Model preparing, available, stale, unavailable, and failed states. Treat a later head or a changed
  top PR as a new target generation so older results cannot replace it.
- Acceptance is a session opening the selected preview through the existing browser, with
  inaccessible and undeployed PRs producing useful state.

### PR 15 — Authorize the sandbox against a TTD preview

Implement the credential path using TTD's existing preview authentication owner. This includes a
companion private TTD PR if extracting a reusable launcher/auth entry point is necessary. The
current laptop workflow relies on `gcloud`/`kubectl` access and a minted session, so the remote
runner needs an explicitly scoped identity.

- Establish the actual GCP/GKE trust path and preview namespace scope in the private repository.
  Reuse the existing authentication implementation and its attributed user/factory context.
- Cover expiry, renewal, sandbox restart, preview recreation, and access removal. Keep tokens out of
  shared artifacts and public configuration.
- Use a dedicated preview test identity and suitable disposable data for the shared browser. Sign-in
  and logout behavior is validated through the deployed preview's real browser flow.
- Acceptance is both harnesses opening the same authenticated preview as a teammate, using the same
  intended identity and data.

### PR 16 — Run the TTD frontend with hot reload in Daytona

Add a companion TTD `.openinspect/start.sh` hook that launches the existing Vite remote-development
configuration against the associated preview backend. Extend the generic launch parameters only
where the current hooks cannot carry the selected target.

- Chromium can open the Vite server inside the sandbox. Configure HMR, HTTPS/cert handling, and host
  validation for that actual path before exposing optional remote URLs.
- Restart the server when its preview target or credentials change and surface failed startup
  through existing boot events. Preserve the ability to inspect the last served deployed preview.
- Acceptance is an uncommitted frontend edit appearing in the shared browser, a teammate interacting
  with it, and correct backend requests reaching the selected PR preview.

### PR 17 — Track the revision actually served by the preview

Extend the preview association to distinguish the current worktree, PR head, and verified served
revision. Read TTD's existing readiness/deployment evidence through the private integration,
including its served-commit annotation where needed.

- A successful workflow event triggers verification of the requested revision. Reconcile
  out-of-order events and a push arriving while an older deploy is finishing.
- Include preview sleep/wake, failed deployment, and an existing hot-sync overlay in the state.
  Capture the worktree revision with validation artifacts so evidence refers to the code exercised.
- Acceptance is a push changing the requested revision while the prior deployed revision stays
  accurately identified until rollout verification succeeds.

## Stack F — Graphite publishing

Step 5 from the earlier discussion is the work in this stack. Its user action is “publish these
changes” or “update the stack.” The result is an ordered set of Graphite PR links associated with
the chat, current-head CI state, and the top PR's preview. Review requests and merging require their
own explicit user instructions.

### PR 18 — Configure Graphite publishing for a repository

Introduce a repository publishing strategy at the existing PR-creation boundary. TTD selects
Graphite. Reuse its `.mise.toml` CLI pin and repository-owned submission wrapper. Authenticate the
publishing identity separately from the selected model subscription.

- Verify that Graphite has access to the target repository and that its Git credentials and
  repository remote agree. Define the publishing owner explicitly for the shared chat.
- Route all normal publish entry points, including the native harness PR tool and web action,
  through the selected strategy. For TTD this replaces the direct `HEAD` push/provider-create path.
- Acceptance is a dry-run contract test showing the correct dispatch and an actionable
  authentication error before any branch mutation.

### PR 19 — Submit the session's Graphite stack

Implement the worker operation that invokes the selected repository's submission wrapper. Capture
the intended ordered branches, parent relationships, and head SHAs before submitting. Use `gt` for
branch/commit/submission work and scoped restacking of branches owned by this session.

- Serialize publication with agent/manual Git writes. Require one commit per planned PR and a
  concise description for each concern. Reconcile remotely updated parents before retrying.
- Persist an operation ID and outcomes for each branch. A retry after partial submission discovers
  existing PRs and completes the remaining work against the recorded heads.
- Use the repository's existing submission behavior and review-state policy. For TTD, inspect and
  call its current `scripts/agent_submit.py` contract instead of duplicating it here.
- Acceptance is a two-PR stack created and then updated with correct bases, plus a forced
  mid-operation disconnect that recovers without duplicate PRs or lost edits.

### PR 20 — Register the published stack in session artifacts

Extend the tool result and existing PR records to represent all submitted PRs and their order. Reuse
[PR snapshot mapping](../../packages/control-plane/src/session/pull-request-snapshot.ts) and
canonical repository/PR identities. Add only the stack relationship fields absent from current
records.

- Reconcile by repository and PR identity, retain historical artifacts after restacking, and
  designate the current top PR explicitly.
- Use Graphite URLs for user-facing PR links while retaining provider identity for API refresh. Keep
  older single-PR records readable.
- Acceptance is a consistent stack shown to both participants after publish, reconnect, and a change
  to the top branch.

### PR 21 — Attach current-head CI status to the stack

Extend existing GitHub webhook and read-through refresh handling for check runs and workflow results
as required. Store observations against the specific PR head SHA and retain the provider update
ordering already used for PR lifecycle state.

- Read status through GitHub APIs. On failure, attach the job link and concise failure reason, with
  log inspection on demand.
- Cover queued, running, successful, failed, cancelled, skipped, and stale-head observations. Use
  bounded refresh as recovery for missed webhooks.
- Acceptance is an old commit's completed checks staying distinct from a newer commit's pending
  checks, including out-of-order delivery.

### PR 22 — Request and associate the top PR's preview

After stack registration, apply TTD's `deploy-preview` label to the verified top PR through its
source-control integration. Reuse the existing workflow for deployment and stack E for revision
verification.

- Make retries idempotent and correlate the request with the top PR and head generation. Reconcile a
  changed stack top before associating a deploy result.
- Show preparing/ready/failed preview status in the chat's existing artifact surfaces. Let TTD's
  established lifecycle own retirement of old previews.
- Acceptance is publish, deploy, and authenticated browser verification of the exact top revision,
  including a second push while the first deploy is running.

## Stack G — Deferred frontend redesign and desktop

Lucas's design direction will determine information architecture, typography, layout, navigation,
and interaction details. Plan this as a full redesign after the backend flows are usable. Existing
chat, diff, and browser surfaces provide functional validation until that work starts.

The eventual stack should first establish a reviewable visual prototype, then migrate the session
list, conversation, changes view, and shared preview in small vertical slices. Each slice should
preserve reconnect state, keyboard use, accessibility, and the implemented account/control
semantics. Avoid committing to the earlier three-pane sketch as a requirement.

Electron packaging follows the redesigned web experience. Its separate concerns are application
startup and browser auth, session deep links, native notifications, and signed distribution/updates.
Branding, app name, and signing identity are decisions for that phase.

## Validation and delivery

Run the affected package checks with the repository's pinned toolchain. Build shared TypeScript
types before dependent packages. Use focused regressions for newly introduced retry, concurrency,
credential, or ownership behavior, followed by the existing package suite and CI. The
[root instructions](../../AGENTS.md) and [preview guide](../verification/authenticated-preview.md)
define the actual commands.

| Change                            | Required evidence                                                                                                                           |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform and Daytona             | Format/validate/provider contract checks, reviewed resource plan, fresh image verification, and the bounded lifecycle run.                  |
| TypeScript contracts and services | Shared build, affected Vitest suites, control-plane integration tests for persisted state, typecheck, and lint.                             |
| Runtime changes                   | Frozen Python environment, affected pytest suites, repository lint/type checks, and a real sandbox exercise where process behavior matters. |
| Functional web controls           | Existing authenticated local preview with two personas, followed by a real two-person sandbox check.                                        |
| TTD companion PRs                 | TTD's own scoped checks and preview validation rules, with evidence linked from the corresponding implementation PR.                        |

For each milestone, record the fork commit, companion TTD commit if any, sandbox/image reference,
selected harness/account label, and observed result. Retain failures and the remaining verification
gap. Capture operation counts and outcomes through existing session events/logs first. Any new
product telemetry should be emitted once at the owner of the accepted operation or terminal outcome.

TTD preview checks should follow its existing traffic limits and stop conditions. Tests requiring
load approval remain separately scoped. Cloud account credentials, Graphite access, signing
identities, and preview data are supplied through their normal configuration paths when their
dependent phase begins.

## Decisions to resolve during implementation

1. Confirm the subscription integration permitted for this hosted deployment, including multiple
   participants contributing through the creator's account. This affects PRs 07–09 and can proceed
   alongside Daytona provisioning.
2. Select the Cloudflare account, domain, Daytona target, initial resource sizing, and GitHub App
   installation. The fork supports the proposed services, while the deployment values remain to be
   supplied.
3. Establish the TTD preview credential path and test identity before PR 15. Its current laptop
   authentication dependencies need explicit remote equivalents.
4. Verify browser attachment and VNC input enforcement against the pinned browser tools before
   finalizing PRs 11–13. The existing desktop viewer alone does not establish shared browser state
   or exclusive input.
5. Choose the GitHub/Graphite publishing identity for the team and preserve authorization to the
   target repository. A model subscription account does not supply Git publishing authority.

## External references

- [Daytona persistence](https://www.daytona.io/docs/en/persistence/) and
  [desktop access](https://www.daytona.io/docs/en/vnc-access/) define the underlying lifecycle and
  viewer capabilities.
- [OpenAI app-server authentication](https://developers.openai.com/codex/app-server/#auth-endpoints),
  [Sign in with ChatGPT integration](https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server),
  and
  [self-hosted VMs](https://developers.openai.com/siwc/token-sharing-open-source/self-hosted-vms)
  describe the distinct subscription integration paths.
- [Claude authentication](https://code.claude.com/docs/en/authentication) and
  [hosted-platform conditions](https://code.claude.com/docs/en/legal-and-compliance) define login
  and credential handling.
- [Graphite command reference](https://graphite.com/docs/command-reference) describes stack
  operations. TTD's repository instructions further constrain the publication workflow.
- [TTD frontend workflow](https://github.com/ProDex-Labs/ttd/blob/main/apps/frontend/README.md#running-against-a-backend),
  [preview workflow](https://github.com/ProDex-Labs/ttd/blob/main/.github/workflows/preview-deploy.yml),
  and
  [hot-sync behavior](https://github.com/ProDex-Labs/ttd/blob/main/k8s/overlays/pr-template/hot-sync/README.md)
  require private repository access.
