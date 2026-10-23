# Changelog

## 2026-10-23
- Added `prompts/2026-10-patch-wave.md`: the October 2026 vulnerability wave (PHP, Node.js, Next.js, eMudhra, LiteLLM, GitLab, NGINX, Apache, WordPress, n8n, OpenSSH, OpenSSL, AOMEI, CISA KEV), every item checked against the vendor's own advisory.
- Added the `patch-watch` skill for daily checks, with its sources list, standing checks and per-platform command recipes.
- Tested on real Linux, cPanel/WordPress, Docker and macOS environments before publishing, and the gaps found were written back into the checks.
