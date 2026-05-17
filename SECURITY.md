# Security

Do not commit secrets to this repository.

The helpers may read runtime credentials from:

- `gh auth token`
- `GH_TOKEN`
- `GITHUB_TOKEN`
- your local Crabbox provider configuration

Those values are used at runtime only and should stay in your shell, keychain, secret manager, or provider CLI.

If you find a security issue, please open a GitHub security advisory if available, or open an issue with enough detail to reproduce the risk without exposing live credentials.
