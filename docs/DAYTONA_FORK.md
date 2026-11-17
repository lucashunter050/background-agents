# Deploy the Daytona fork

This profile runs the stock web app and control plane on Cloudflare. Daytona runs sessions from a Terraform-managed base image. Sign-in starts with the GitHub user `lucashunter050`. Add teammate usernames to the allowlist when you're ready to grant access.

The example uses a 4 GiB base image for the pilot. Open-Inspect stops an idle web session after ten minutes when nobody is connected. Daytona's existing fallback timers stop an idle sandbox after two hours and archive a stopped sandbox after seven days. Repository-specific images can be enabled later.

## Accounts and secrets

Use the Daytona organization that holds the sandbox credits. Use the Cloudflare account selected for this deployment and create a dedicated GitHub App. The [upstream setup guide](GETTING_STARTED.md) covers provider setup in more detail.

| Provider | Create or collect |
| --- | --- |
| Cloudflare | Account ID, Workers subdomain, and an API token with Workers Scripts, KV, R2, D1, and Queues edit permissions. |
| R2 state | A state bucket and R2 object read/write credentials scoped to that bucket. |
| GitHub App | App ID, installation ID, PKCS#8 private key, OAuth client ID, and OAuth client secret. Grant Contents and Pull requests read/write, Metadata read, and user Email addresses read. Install it on the pilot repositories. Leave webhooks inactive for the first deployment. |
| Daytona | Organization API key with sandbox write and snapshot write/delete permissions. Set the target if the organization uses a specific one. |
| Application | Three persistent secrets for browser auth, GitHub token encryption, and repository secret encryption. Generate each independently with `openssl rand -base64 32`. |

For GitHub OAuth, set the callback URL to `https://open-inspect-web-lucashunter050-daytona.<subdomain>.workers.dev/api/auth/callback/github`. Keep user-to-server token expiration enabled.

## Local setup

Copy [the profile](../terraform/environments/production/daytona-fork.tfvars.example) to `terraform/environments/production/terraform.tfvars`, which Git ignores.

Set the following Terraform inputs using the ignored file or `TF_VAR_` environment variables.

- Cloudflare uses `cloudflare_account_id`, `cloudflare_worker_subdomain`, and `cloudflare_api_token`.
- GitHub uses `github_app_id`, `github_app_installation_id`, `github_app_private_key`, `github_client_id`, and `github_client_secret`.
- Daytona uses `daytona_api_key` and optionally `daytona_target`.
- The app uses `nextauth_secret`, `token_encryption_key`, and `repo_secrets_encryption_key`.

Follow the [state initialization instructions](GETTING_STARTED.md#configure-backendtfvars) and [two-phase deployment](GETTING_STARTED.md#step-6-deploy-with-terraform). With both bindings enabled, follow the [Owner bootstrap](GETTING_STARTED.md#step-9-bootstrap-the-workspace-owner) and [team setup](GETTING_STARTED.md#step-10-create-the-first-team-and-test-a-session). A first sign-in creates a member account. The bootstrap grants Owner privileges and the team setup grants access to specific repositories.

The base deployment can sign users in before any model account is connected. Subscription binding for shared coding sessions is tracked in the [implementation plan](plans/collaborative-coding-app.md#stack-c--subscriptions-and-collaboration).
