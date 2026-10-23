# Sources

Use primary sources: the vendor's own advisory, the distro's own notice, CISA. News sites and social posts are leads, not evidence. Every item in a report links to the primary source you actually read. If you could only find a secondary source, say so.

## Always check

| Source | What it tells you | How to query |
|---|---|---|
| CISA Known Exploited Vulnerabilities (KEV) | Bugs attackers are actually exploiting. Anything here that you run goes to "Do today". | JSON feed: `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`. Filter `dateAdded` since `last_run`, then match `vendorProject` and `product` against the inventory. |
| OSV (osv.dev) | Known vulnerabilities AND known malicious packages (IDs starting `MAL-`) for npm, PyPI, Packagist (Composer), Go, Maven, RubyGems, crates.io, Debian, Ubuntu, Alpine and more. | `osv-scanner scan -r <project dir>` for lock files, or the API: `POST https://api.osv.dev/v1/querybatch` with `{"queries":[{"package":{"name":"litellm","ecosystem":"PyPI"},"version":"1.107.0"}]}`. |
| GitHub Advisory Database | Reviewed advisories with affected and patched version ranges, including many vendor GHSAs. | `gh api "/advisories?ecosystem=npm&affects=next@14.2.3"` or `gh api "/advisories?cve_id=CVE-YYYY-NNNN"`. |
| endoflife.date | Whether a version is still supported at all. End-of-life means no fix will ever come. | `https://endoflife.date/api/php.json` (also `nodejs`, `python`, `ubuntu`, `debian`, `almalinux`, `nginx`, `mysql`, `postgresql`, `redis`, `wordpress`, `gitlab` and many more). |
| EPSS (FIRST) | Probability a CVE gets exploited in the next 30 days. Use it to order the "This week" list. | `https://api.first.org/data/v1/epss?cve=CVE-YYYY-NNNN` |

## Scanners worth running (if installed, or offer to install)

- **Trivy** (`trivy image <image>`, `trivy fs <dir>`, `trivy rootfs /`): finds vulnerable OS packages and language packages inside container images and on hosts. The fastest way to check a whole Docker fleet.
- **Grype** (`grype <image>`): an alternative to Trivy.
- **osv-scanner**: lock files and SBOMs.
- Language audits: `npm audit`, `pnpm audit`, `pip-audit`, `composer audit`, `bundle audit`, `govulncheck ./...`, `cargo audit`.
- WordPress: `wp core check-update`, `wp plugin list --update=available`, `wp theme list --update=available`; vulnerability data from Wordfence Intelligence (free API with a key), Patchstack or WPScan.

## Operating systems (distros backport, so check THEIR notice)

| OS | Notices | On the machine |
|---|---|---|
| Ubuntu | `https://ubuntu.com/security/notices` (RSS: `/security/notices/rss.xml`); per CVE: `https://ubuntu.com/security/CVE-YYYY-NNNN` | `apt list --upgradable`, `/var/run/reboot-required`, `pro status` (ESM for old releases) |
| Debian | `https://www.debian.org/security/`; per CVE: `https://security-tracker.debian.org/tracker/CVE-YYYY-NNNN` | `apt list --upgradable`, `debsecan` |
| AlmaLinux / Rocky / RHEL | `https://errata.almalinux.org/`, Rocky errata, Red Hat CVE pages | `dnf updateinfo list --security`, `needs-restarting -r` |
| CloudLinux | CloudLinux blog and changelog | `yum updateinfo` |
| Alpine (containers) | `https://security.alpinelinux.org/` | `apk version -l '<'` inside the container |
| macOS | `https://support.apple.com/en-us/100100` (Apple security releases) | `softwareupdate -l` |
| Windows | Microsoft Security Update Guide, `https://msrc.microsoft.com/update-guide` | `winget upgrade`, Windows Update history |

## Runtimes and servers

| Product | Primary source |
|---|---|
| PHP | `https://www.php.net/ChangeLog-8.php`, advisories at `https://github.com/php/php-src/security/advisories`, supported versions at `https://www.php.net/supported-versions.php` |
| Node.js | `https://nodejs.org/en/blog/vulnerability`, release schedule at `https://github.com/nodejs/release#release-schedule` |
| Python | `https://www.python.org/news/security/`, `https://devguide.python.org/versions/` |
| Go | `https://go.dev/doc/devel/release`, `https://pkg.go.dev/vuln/` |
| Java (OpenJDK) | Oracle Critical Patch Updates, Adoptium release notes |
| NGINX | `https://nginx.org/en/security_advisories.html` |
| Apache HTTP Server | `https://httpd.apache.org/security/vulnerabilities_24.html` |
| OpenSSL | `https://openssl-library.org/news/vulnerabilities/` |
| OpenSSH | `https://www.openssh.com/security.html`, `https://www.openssh.com/releasenotes.html` |
| PostgreSQL | `https://www.postgresql.org/support/security/` |
| MySQL / MariaDB | Oracle CPU for MySQL, `https://mariadb.com/kb/en/security/` |
| Redis | `https://github.com/redis/redis/security/advisories` |
| Docker Engine, containerd, runc | `https://docs.docker.com/engine/release-notes/`, GitHub advisories for `moby/moby`, `containerd/containerd`, `opencontainers/runc` |
| Official Docker images | Docker Hub tags API, for example `https://hub.docker.com/v2/repositories/library/php/tags?name=8.3` (confirms a fixed tag is actually published) |

## Apps and platforms

| Product | Primary source |
|---|---|
| WordPress core | `https://wordpress.org/news/category/security/` |
| WordPress plugins and themes | The plugin's own changelog and incident page, Wordfence Intelligence, Patchstack database, WPScan |
| cPanel / WHM | cPanel security announcements and the EasyApache 4 changelog; `/usr/local/cpanel/cpanel -V` |
| Plesk | Plesk security advisories |
| GitLab (self-managed) | `https://about.gitlab.com/releases/categories/releases/` and `https://docs.gitlab.com/releases/patch-releases.xml` |
| n8n | `https://github.com/n8n-io/n8n/security/advisories` |
| Laravel, Symfony, Drupal, Magento, Joomla | Each project's security advisories page; `composer audit` covers most |
| Next.js / React | `https://github.com/vercel/next.js/security/advisories`; `npm audit` |
| Python AI tooling (LiteLLM, LangChain, etc.) | Project GitHub security advisories and blog; OSV `MAL-` entries for malicious releases |

## Certificate authorities and trust stores

- Mozilla security blog and the `dev-security-policy` group (CA distrust and incident decisions), `https://wiki.mozilla.org/CA`
- Python `certifi` releases on PyPI, Node.js release notes (bundled CA store), Go release notes, curl CA bundle (`https://curl.se/docs/caextract.html`)
- On macOS, the system `curl`, `git`, `python3`, `perl`, `ruby` and `openssl` read `/etc/ssl/cert.pem`, which only Apple updates.

## Early warning (leads, verify before acting)

- oss-security mailing list: `https://www.openwall.com/lists/oss-security/`
- Vendor and security-company blogs (Cloudflare, Rapid7, Wiz, Wordfence, Patchstack, Google Threat Intelligence)
- National CERTs (CISA alerts, NCSC, CERT-EU, Canadian Centre for Cyber Security)
