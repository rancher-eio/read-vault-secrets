# Read Vault Secrets for GitHub Actions

This Action allows (authorized) repositories to fetch secrets from a deployment of Vault. It does this by wrapping around `hashicorp/vault-action` and providing the appropriate configuration for how our Vault is set up, so you don't have to.

All you have to do is tell it which secrets to fetch, and where to put them.

## Example Workflow

This is a complete example of a workflow that includes everything: permissions, reading secrets, and using them. A more detailed explanation follows after the example.

```yaml
---
name: Example Workflow
on:
  push:
    branches:
    - "**"
jobs:
  example-job:
    permissions:
      contents: read
      id-token: write # this is important, it's how we authenticate with Vault
    runs-on: ubuntu-22.04
    steps:
    - name: "Read some Secrets"
      uses: rancher-eio/read-vault-secrets@main
      with:
        secrets: |
          secret/data/github/repo/${{ github.repository }}/dockerhub/credentials username | DOCKER_USERNAME ;
          secret/data/github/repo/${{ github.repository }}/dockerhub/credentials password | DOCKER_PASSWORD
    - name: "Use those Secrets"
      run: |
        echo "${{ env.DOCKER_USERNAME }}
```

## Details

### Required Permissions

We rely on JWT tokens issued by GitHub's OIDC provider ([docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)) for authentication with Vault.

In order for this to work, the job needs the `id-token: write` permission. While this can be configured for the entire workflow, the recommended approach is to configure this permission at the job level. This is preferred for a few reasons:

1. It clearly identifies which jobs require access to secrets.
2. It limits the exposure of secrets to subsequent steps in those jobs.
3. Defining _any_ permissions **replaces** the [default permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token), according to the [docs](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#modifying-the-permissions-for-the-github_token):

    > When the `permissions` key is used, all unspecified permissions are set to no access, with the exception of the `metadata` scope, which always gets read access.

```yaml
jobs:
  example-job:
    permissions:
      id-token: write
      # ... and any other permissions the job requires.
```

### Reading Secrets

To read secrets from Vault, you need to provide the following:

1. The path to a secret in Vault, and the name of the value you want read.
2. The name to export it as in the Environment for any subsequent steps in the job.

These are combined like so:

```handlebars
{{ SECRET_PATH }} {{ SECRET_NAME }} | {{ ENV_NAME }}
```

If you need multiple values from a secret (such as `username` and `password`), separate them with semicolons (`;`) like so:

```handlebars
{{ SECRET_PATH }} {{ THIS_SECRET_NAME }} | {{ ENV_NAME_FOR_THIS_SECRET }} ;
{{ SECRET_PATH }} {{ THAT_SECRET_NAME }} | {{ ENV_NAME_FOR_THAT_SECRET }}
```

You can use any references [available in the current context](https://docs.github.com/en/actions/learn-github-actions/contexts#context-availability) as well. So the path to the `dockerhub/credentials` secret for the current repo could be written as:

```handlebars
secret/data/github/repo/${{ github.repository }}/dockerhub/credentials
```

And to fetch two values (`username` and `password`) from that secret and export them to `DOCKER_USERNAME` and `DOCKER_PASSWORD` in the environment, you could do this:

```yaml
    steps:
    - uses: rancher-eio/read-vault-secrets@main
      with:
        secrets: |
          secret/data/github/repo/${{ github.repository }}/dockerhub/credentials username | DOCKER_USERNAME ;
          secret/data/github/repo/${{ github.repository }}/dockerhub/credentials password | DOCKER_PASSWORD
```

### Using Secrets

Once you have successfully fetched your secret(s) from Vault, they are exported into the environment for all subsequent steps in that job, and can be referenced like any other environment variable. So to write the `DOCKER_USERNAME` from above to STDOUT (this is a terrible idea, and would actually be redacted), you could do this:

```yaml
    steps:
    # ... see above
    - run: |
        echo "${{ env.DOCKER_USERNAME }}
```

And that's it!

You can stop reading now, unless... you want to know more about the paths? 👉👈

---

#### Understanding the Path

The secret path is constructed like so for repo-level secrets:

```handlebars
secret/data/github/repo/{{ repoOwner }}/{{ repoName }}/some/secret
```

And like so for org-level secrets:

```handlebars
secret/data/github/org/{{ orgName }}/some/secret
```

In the examples above, the `secret/data` prefix is required because `secret` is the mount path of the secret engine in Vault, and `data` tells Vault that you want the contents (and not the metadata) of the secret. We could have called this anything, but `secret` seemed both intuitive and descriptive for what it is.

The next few segments of the path can be thought of as both a hierarchy and a namespace.

- the first segment (`github`) indicates that secrets under this prefix are related to GitHub.
- the next segment (`org` or `repo`) indicates the scope of the secret
- the next segment(s) indicate the resource assocated with that scope
  - `{{ orgName }}` for `org` secrets
  - `{{ ownerName }}/{{ repoName }}` for `repo` secrets
- the rest is arbitrary and should describe the secret, such as `dockerhub/credentials`.

We use this for policy management, and while the resulting path is long, it allows intuitive and flexible policy management for both humans and machines. A few simple examples:

- `github/*` applies to all github secrets.
- `github/org/rancher-eio/*` applies to all **org-level** secrets for the `rancher-eio` org.
- `github/repo/rancher-eio/*` applies to all **repo-level** secrets for the `rancher-eio` org.
- `github/repo/rancher-eio/read-vault-secrets/*` applies to all secrets for the `rancher-eio/read-vault-secrets` repo.
