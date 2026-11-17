# Deploy the Daytona fork

This profile runs the stock web app and control plane on Cloudflare. Daytona runs sessions from a
Terraform-managed base image. Sign-in starts with the GitHub user `lucashunter050`. Add teammate
usernames to the allowlist when you're ready to grant access.

The example uses a 4 GiB base image for the pilot. Open-Inspect stops an idle web session after ten
minutes when nobody is connected. Daytona's existing fallback timers stop an idle sandbox after two
hours and archive a stopped sandbox after seven days. Repository-specific images can be enabled
later.

## Accounts and secrets

Use the Daytona organization that holds the sandbox credits. Use the Cloudflare account selected for
this deployment and create a dedicated GitHub App. The [upstream setup guide](GETTING_STARTED.md)
covers provider setup in more detail.

| Provider    | Create or collect                                                                                                                                                                                                                                                         |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloudflare  | Account ID, Workers subdomain, and an API token with Workers Scripts, KV, R2, D1, and Queues edit permissions.                                                                                                                                                            |
| R2 state    | Create the `lucashunter050-open-inspect-state` bucket and issue R2 object read/write credentials scoped to it. The state key is `daytona/terraform.tfstate`.                                                                                                              |
| GitHub App  | App ID, installation ID, PKCS#8 private key, OAuth client ID, and OAuth client secret. Grant Contents and Pull requests read/write, Metadata read, and user Email addresses read. Install it on the pilot repositories. Leave webhooks inactive for the first deployment. |
| Daytona     | Organization API key with sandbox write and snapshot write/delete permissions. Set the target if the organization uses a specific one.                                                                                                                                    |
| Application | Three persistent secrets for browser auth, GitHub token encryption, and repository secret encryption. Generate each independently with `openssl rand -base64 32`.                                                                                                         |

For GitHub OAuth, set the callback URL to
`https://open-inspect-web-lucashunter050-daytona.<subdomain>.workers.dev/api/auth/callback/github`.
Keep user-to-server token expiration enabled.

## Local setup

Copy [the profile](../terraform/environments/production/daytona-fork.tfvars.example) to
`terraform/environments/production/terraform.tfvars`, which Git ignores.

Set the following Terraform inputs using the ignored file or `TF_VAR_` environment variables.

- Cloudflare uses `cloudflare_account_id`, `cloudflare_worker_subdomain`, and
  `cloudflare_api_token`.
- GitHub uses `github_app_id`, `github_app_installation_id`, `github_app_private_key`,
  `github_client_id`, and `github_client_secret`.
- Daytona uses `daytona_api_key` and optionally `daytona_target`.
- The app uses `nextauth_secret`, `token_encryption_key`, and `repo_secrets_encryption_key`.

Follow the [state initialization instructions](GETTING_STARTED.md#configure-backendtfvars) and
[two-phase deployment](GETTING_STARTED.md#step-6-deploy-with-terraform). With both bindings enabled,
follow the [Owner bootstrap](GETTING_STARTED.md#step-9-bootstrap-the-workspace-owner) and
[team setup](GETTING_STARTED.md#step-10-create-the-first-team-and-test-a-session). A first sign-in
creates a member account. The bootstrap grants Owner privileges and the team setup grants access to
specific repositories.

## Deploy from GitHub Actions

Set these repository variables in **Settings > Secrets and variables > Actions > Variables**.

| Variable                                                    | Value                                                            |
| ----------------------------------------------------------- | ---------------------------------------------------------------- |
| `TF_STATE_BUCKET`                                           | `lucashunter050-open-inspect-state`                              |
| `TF_STATE_KEY`                                              | `daytona/terraform.tfstate`                                      |
| `DEPLOYMENT_NAME`                                           | `lucashunter050-daytona`                                         |
| `WEB_PLATFORM`                                              | `cloudflare`                                                     |
| `SANDBOX_PROVIDER`                                          | `daytona`                                                        |
| `ALLOWED_USERS`                                             | `lucashunter050` or a comma-separated list including teammates   |
| `TEAMS_ENFORCEMENT`                                         | `on`                                                             |
| `ENABLE_SLACK_BOT`                                          | `false`                                                          |
| `DAYTONA_API_URL`                                           | `https://app.daytona.io/api`                                     |
| `DAYTONA_BASE_SNAPSHOT`                                     | `lucashunter050-open-inspect`                                    |
| `DAYTONA_BASE_SNAPSHOT_MEMORY_GIB`                          | `4`                                                              |
| `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_WORKER_SUBDOMAIN`      | Values from Cloudflare                                           |
| `GH_APP_ID`, `GH_APP_INSTALLATION_ID`, `GH_OAUTH_CLIENT_ID` | Values from the GitHub App                                       |
| `DAYTONA_TARGET`                                            | Optional. Leave unset to use the Daytona organization's default. |

Store these in **Actions > Secrets**.

- Cloudflare and R2 use `CLOUDFLARE_API_TOKEN`, `R2_ACCESS_KEY_ID`, and `R2_SECRET_ACCESS_KEY`.
- GitHub uses `GH_APP_PRIVATE_KEY` and `GH_OAUTH_CLIENT_SECRET`.
- Daytona uses `DAYTONA_API_KEY`.
- The app uses `NEXTAUTH_SECRET`, `TOKEN_ENCRYPTION_KEY`, and `REPO_SECRETS_ENCRYPTION_KEY`.

After these changes land on the fork's `main`, enable Actions in the fork if GitHub prompts you to.
Dispatch **Terraform** twice. Wait for the first run to succeed before starting the second. Use only
the second command for subsequent deploys.

```bash
gh workflow run terraform.yml --ref main -f bootstrap=true
gh workflow run terraform.yml --ref main -f bootstrap=false
```

The first apply creates the database, image, Workers, and Durable Object classes. The second
activates the bindings. It reuses the same Terraform state. Use `terraform output -raw web_app_url`
and `terraform output -raw d1_database_name` from the locally initialized production directory to
look up the deployed app and database.

The base deployment can sign users in before any model account is connected. Subscription binding
for shared coding sessions is tracked in the
[implementation plan](plans/collaborative-coding-app.md#stack-c--subscriptions-and-collaboration).
