# Read Vault Secrets for GitHub Actions

## Example Workflow

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
