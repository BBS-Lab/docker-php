# Changelog

All notable changes to `bigbossstudio/php` will be documented in this file.

## 2026-10-29

### 🐛 Bug Fixes

- `FilesMatch "^\."` no longer blocks `/.well-known/*`. The 2026-10-21 hardening already exempted `.well-known` in `DirectoryMatch`, but Apache still matched the `.well-known` path segment as a hidden file and returned 403 before Laravel (or any front controller) could serve OAuth discovery, ACME challenges, or `security.txt`. The FilesMatch now uses `^\.(?!well-known$)`, same exemption as DirectoryMatch.

## 2026-10-21

### 💥 Breaking Changes

- `linux/arm/v7` (32-bit ARM) is no longer published. Images are built for `linux/amd64` and `linux/arm64/v8`, which cover Intel and Apple Silicon Macs, x86 and ARM 64-bit Linux. The existing `arm/v7` images stay on Docker Hub but are no longer updated.
- Access logs now use the `combined` format (referer, user agent) followed by the response time in microseconds, and AWS load balancer health checks coming from a private IP are no longer logged. Update your log parsers or CloudWatch metric filters if they rely on the previous format.

### 🔒 Security

- PHP errors are no longer displayed to visitors (`display_errors = Off`). The PHP default (`On`) applied before, leaking file paths. Errors are still logged to the container output.
- Every hidden file and directory (`.env`, `.user.ini`…) is now denied, except `.well-known`. Only `.git` and `.gitlab-ci.yml` were denied before.
- `Referrer-Policy` is now `strict-origin-when-cross-origin`: full URLs, query string included, are no longer sent to other sites.
- Removed `tj-actions/branch-names` from the publish workflow (vulnerable version, and not needed: `actions/checkout` already checks out the triggering branch).

### 🐛 Bug Fixes

- The weekly rebuild had stopped since 2026-06-01 (published images were stuck on PHP 8.5.6): GitHub disables scheduled workflows after 60 days without commits. The workflow now re-enables itself on each scheduled run.
- GD is now built with FreeType and JPEG support (`imagejpeg()`, `imagettftext()`…). Only PNG was supported before.
- PHP warnings were never logged under Apache: the vhost `LogLevel error` dropped them. The log level is now `warn`.
- PHP errors could be written to a `/error.log` file inside the container when `APACHE_LOG_DIR` was not set (e.g. cron jobs). The `error_log` setting is removed: PHP errors always go to stderr.
- `post_max_size` is now `20M`, so that a 16 MB upload with other form fields no longer drops the whole request.

### ✨ Features

- [PIE](https://github.com/php/pie) stays available in the image (`pie install vendor/extension`), copied from the official `ghcr.io/php/pie:1-bin` image. `phpredis` is now installed with PIE instead of PECL.

### ⚡ Performance

- Smaller image (−22 MB) and faster build (−30 %): apt cleanup now runs in the same layer as the install, and all extensions are compiled in a single pass.

### 🧹 Chores

- CI: bumped every action to its Node 24 version (Node 20 was removed from the runners on 2026-09-16), and a failing PHP version no longer cancels the build of the other one.
- Removed duplicated and unused Apache directives.
- Removed the PHP 7.4 / 8.0 specific packages. The default `PHP_VERSION` build argument is now `8.5`.
- README: documented logging (CloudWatch, Laravel), the client IP / Cloudflare behaviour and how to disable it, and the Laravel scheduler with `schedule:work`. Fixed the Puppeteer recipe (`fonts-kacst` no longer exists on Debian 13, wrong `npm` command), bumped the Node.js recipe to Node 24 and listed the real PHP modules.
