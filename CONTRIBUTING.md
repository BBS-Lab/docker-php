# Contributing

Contributions are welcome. To keep the image green:

1. Fork and branch from `main`.
2. Build every supported version locally before opening a PR (requires a `docker buildx` builder named `mybuilder`):
   ```bash
   ./dev.sh        # builds the versions listed in dev.sh
   ./dev.sh 8.4    # builds a single version
   ```
3. Keep the supported versions in sync between `dev.sh` and `.github/workflows/workflow-docker-publish.yml`.
4. Every image must build for `linux/amd64` and `linux/arm64/v8`.
5. Update the PHP modules list in the [README](README.md) when an extension is added or removed, and add an entry to the [CHANGELOG](CHANGELOG.md).

## Security

Please email `paris@big-boss-studio.com` for security issues instead of the issue tracker.
