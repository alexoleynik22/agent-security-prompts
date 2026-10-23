# Standing checks

Run these every time. Version matching alone misses most of what goes wrong after a patch. These are the gaps that most often survive a patch.

First do a one-line inventory per machine and skip checks that cannot apply (no Docker, no WordPress, no Windows). Mark anything you could not verify as `unknown`, never `clean`, and say why (no sudo, timed out, no access).

## 1. The patched code is the code that is running

- **Containers.** For every running container, read the versions inside it: `docker exec <c> php -v`, `node -v`, `nginx -v`, `openssl version`, and `/etc/os-release`. For Python and certifi, list `site-packages/*.dist-info` instead of running Python. Report one line per container. Sidecars count: a phpMyAdmin container is a PHP install.
- **Pulled but not recreated.** Compare each container's creation time with its image's creation time. A fixed image that was pulled but never used to recreate the container fixes nothing.
- **Unpinned and old images.** `FROM node:20-alpine`, `FROM nginx:alpine` and `:latest` tags only pick up fixes when rebuilt with `docker build --pull --no-cache`. Report images built before the fix date, pinned or not.
- **Long-running processes.** After a PHP, Node or Python update, confirm the live process restarted: compare `ps -o lstart= -p <pid>` with the package install time, or check `needrestart -b` (Debian/Ubuntu) and `needs-restarting` (RHEL/Alma). PHP-FPM pools, LSAPI, mod_php, Node apps and queue workers all keep old code in memory. Reading these is always allowed and needs no approval.
- **Pending reboot.** `/var/run/reboot-required` on Debian/Ubuntu, `needs-restarting -r` on RHEL/Alma. On some hosts (for example provider-managed netboot kernels) `rpm -q kernel` is empty, so compare `uname -r` and uptime instead. Before proposing a reboot, look for that host's known boot problems and confirm out-of-band console access.
- **Live check over HTTP.** Where possible, confirm a website runs the patched runtime by a request it serves, not by the CLI.

## 2. Versions that will never be fixed

List everything on an end-of-life version (check `https://endoflife.date/api/<product>.json`): PHP below the oldest supported branch, Node odd-numbered and old even lines, Python, Ubuntu 20.04 without ESM, CentOS 7 without an ELS repo, Debian oldoldstable, MySQL 5.7, Elasticsearch on EOL bases. These are not "behind on updates". They are permanently vulnerable until moved or taken offline. Give them their own section in the report.

## 3. Vendor builds that carry fixes without changing the version

cPanel EasyApache, CloudLinux alt-php, Imunify360 "hardened" PHP, TuxCare ELS and distro packages often backport fixes and keep the old version number. Before calling a version vulnerable, check the package changelog for the CVE: `rpm -q --changelog <pkg> | head -60 | grep CVE-YYYY-NNNN` or `apt changelog <pkg> | grep CVE-YYYY-NNNN`. A hit counts as patched (report it as a vendor claim). Also check the vendor's own notice. After switching a panel PHP build by hand, re-check after the next nightly update, because a higher-epoch vendor package can silently replace it.

## 4. Forgotten copies

Staging sites, old subdomains, `.wp-toolkit_*` clones, `/old` and `/backup` copies of a site, suspended hosting accounts that still hold code, a second server nobody logs into. List every vhost (on cPanel: `/etc/userdatadomains`), check which still resolve to the server in public DNS, and report the ones that serve code but no longer matter. Check `/var/cpanel/suspended/<user>` before treating a broken site as unknown.

## 5. Things exposed to the internet that should not be

From outside, against the user's own domains only, check for: `/.git/HEAD`, `/.env`, `/wp-config.php.bak`, `phpinfo.php`, backup archives (`.zip`, `.sql`, `.tar.gz`) in web roots, `wp-content/debug.log`, `error_log`, phpMyAdmin, Adminer, Docker or Kubernetes dashboards, n8n, database ports (3306, 5432, 6379, 27017, 9200). A hit only counts if the body is the real file, not the app's catch-all page (single-page apps return 200 with HTML for everything; compare content type and size). For domains behind Cloudflare, find which origin actually serves them before fixing anything.

## 6. Ports that bypass the firewall

On Linux, `docker run -p 5432:5432` is reachable from the internet even when UFW or firewalld says the port is closed, because Docker writes its own iptables rules. Run `docker ps --format '{{.Names}} {{.Ports}}'`, read `iptables -S DOCKER-USER` and `iptables -S INPUT`, and for every `0.0.0.0` port say which rule blocks it, or that none does. Test from a machine that is NOT in any allowlist and name that vantage point. Testing from an allowlisted IP gives false "open" results. If you have no clean vantage point, say `unknown`.

## 7. Firewall allowlists that drifted

Read the allowlist of every firewall in use: CSF (`/etc/csf/csf.allow`), the Imunify360 whitelist, UFW, firewalld, iptables, cloud security groups, Cloudflare IP access rules. Flag any range wider than /24, residential or foreign home IPs nobody can name, proxy networks, and entries with no named person or service. Attackers who get in once often add themselves here, and it survives a cleanup.

## 8. Persistence

- Cron (all users, `/etc/cron.*`), systemd units and timers, launchd agents and daemons (macOS, including `/Library/LaunchDaemons`), Windows scheduled tasks.
- `authorized_keys` AND `authorized_keys2` for root and every user (`sshd -T | grep authorizedkeysfile` shows which files count). Compare key fingerprints across servers; the same unknown key on several boxes is a lead.
- New users with sudo, admin or `sshusers` membership.
- WordPress administrators created recently, especially random-looking logins, users named `root` or `admin`, and throwaway emails such as `@example.com`. Must-use plugins and plugin folders that imitate real plugin names.
- Remote-access tools you did not install (Splashtop, AnyDesk, TeamViewer, ScreenConnect, RustDesk).
- Before calling something persistence, check whether a package owns it (`rpm -qf`, `dpkg -S`): some products install their own sudoers entries and mu-plugins.

## 9. Logins and logs

- Count successful logins per user and source IP from `/var/log/auth.log`, `/var/log/secure*` or `journalctl -u ssh`, not only for root. Flag unexpected countries and keys.
- A stale `wtmp`/`last` is normal on automation hosts: sessions without a terminal (CI, scripts, agents) never write it. It is only a finding if the auth log also stops.
- State the earliest timestamp of every log you rely on. If it is later than the exposure window, say the window cannot be checked.
- If Imunify360 or ImunifyAV is present, read its detections: `imunify360-agent malware malicious list` and `imunify360-agent malware history list`. A malware scanner's history is often the best evidence available.

## 10. SSH and panel logins

- Check the EFFECTIVE SSH config with `sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|kbdinteractive'`. Drop-ins in `sshd_config.d/` (for example `50-cloud-init.conf`) are read first and can override the main file.
- Confirm from outside: `ssh -o PreferredAuthentications=password,keyboard-interactive -o PubkeyAuthentication=no user@host` must be refused.
- cPanel/WHM, Plesk and WordPress admin logins should have two-factor authentication. Check that brute-force protection (cPHulk, fail2ban, CSF lfd) is actually running, not just installed.

## 11. Origins that bypass the proxy

If a site is behind Cloudflare or another proxy, check whether the origin IP also answers directly for that hostname. If it does, the WAF can be bypassed. Recommend restricting ports 80 and 443 to the proxy's IP ranges. An expired origin certificate matters only if the proxy validates it (Full strict) or the domain points straight at the server.

## 12. Code: disabled certificate checks and committed secrets

Search the user's repos for `verify=False`, `NODE_TLS_REJECT_UNAUTHORIZED=0`, `rejectUnauthorized: false`, `curl -k`, `InsecureSkipVerify`, and for keys, tokens and passwords committed to git, including history. Report locations, never values. Secrets also leak through `crontab -l`, `ps` arguments, systemd unit files and SSH command lines: mask them there too.

## 13. Dependencies

Run the native audit per project (`npm audit`, `pnpm audit`, `pip-audit`, `composer audit`, `bundle audit`, `govulncheck`, `cargo audit`) or `osv-scanner`. For container images, `trivy image`. For Next.js, check the `next` version AND the `sharp` version inside the built image, and whether the image uses glibc or musl, because some image-optimization bugs only affect glibc builds. For WordPress: core minor updates (`wp core check-update --minor`; a site on an older branch is fine if it has the latest minor of that branch, but report unsupported branches), plugin and theme updates, and plugins closed on WordPress.org.

## 14. Trust stores

Report the CA bundle date of each OS, container base and language runtime (`ca-certificates` package, `certifi` version, Node version). To check for a distrusted CA, look for it in the bundle (`openssl crl2pkcs7 -nocrl -certfile <bundle> | openssl pkcs7 -print_certs -noout | grep -i <ca name>`) or connect to a test host that is known to use that CA's chain and confirm the client rejects it. On macOS, the system `curl`, `git`, `python3` and `openssl` use `/etc/ssl/cert.pem`, which only Apple updates.

## 15. Backups and mail

- Each important site and database has a recent backup that lives off the server, ideally versioned or immutable. Report the newest backup date. A restore test usually needs a human: if you cannot do it, say `unverified`.
- The server's own mail (contact forms, password resets, system notices) is authenticated with SPF and DKIM, actually delivered, and the server is not an open relay. A frozen or bouncing mail queue is a symptom worth reporting.

## 16. Safe to run at scale

Never run `find /` or deep recursive searches across all of `/home` on a busy shared server; it can stall the box. Restrict searches to the directories that matter (for example `site-packages`, `node_modules/next`, `wp-content`). On macOS use targeted paths or Spotlight. Homebrew commands can modify state: set `HOMEBREW_NO_AUTO_UPDATE=1` before `brew outdated`.
