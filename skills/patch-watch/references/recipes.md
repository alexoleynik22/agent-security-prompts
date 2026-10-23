# Recipes

Read-only commands per platform. Everything here inspects; nothing changes state. Run with `sudo` where noted only if the user has granted it, and report checks you could not run as `unknown`.

## Any Linux host

```sh
cat /etc/os-release; uname -r; uptime
sshd -T 2>/dev/null | grep -Ei 'passwordauthentication|permitrootlogin|kbdinteractive|authorizedkeysfile'
ss -ltnp                                   # listening ports and owners
iptables -S INPUT; iptables -S DOCKER-USER  # firewall order matters
for u in $(cut -d: -f1 /etc/passwd); do crontab -l -u "$u" 2>/dev/null | sed "s/^/$u: /"; done
ls -la /etc/cron.* /etc/systemd/system; systemctl list-timers --all
ls -la /root/.ssh /home/*/.ssh             # authorized_keys AND authorized_keys2
```

## Debian / Ubuntu

```sh
apt list --upgradable 2>/dev/null
ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required.pkgs
needrestart -b 2>/dev/null                 # services still running old libraries
pro status 2>/dev/null                     # Ubuntu ESM coverage for old releases
apt changelog <pkg> 2>/dev/null | grep CVE-YYYY-NNNN
```

## RHEL / AlmaLinux / Rocky / CloudLinux / CentOS

```sh
dnf -C updateinfo list --security 2>/dev/null || yum updateinfo list security
needs-restarting -r; needs-restarting -s
rpm -q kernel openssh-server openssl
rpm -q --changelog <pkg> | head -60 | grep CVE-YYYY-NNNN
yum repolist | grep -i els                 # CentOS 7 gets nothing without an ELS repo
```

## cPanel / WHM

```sh
/usr/local/cpanel/cpanel -V
whmapi1 php_get_installed_versions
whmapi1 php_get_vhost_versions             # which PHP each site uses
whmapi1 php_get_handlers                   # cgi / lsapi / fpm / dso decides what needs a restart
cat /etc/userdatadomains                   # every vhost and its docroot
ls /var/cpanel/suspended/                  # suspended accounts still hold code
rpm -qa | grep -E '^ea-php[0-9]+-php-cli' | while read p; do echo "$p"; rpm -q --changelog "$p" | head -40 | grep -E 'CVE-YYYY-NNNN'; done
/usr/local/cpanel/3rdparty/bin/php -v      # cPanel's own internal PHP
csf -l 2>/dev/null | head; cat /etc/csf/csf.allow
imunify360-agent malware malicious list 2>/dev/null
imunify360-agent malware history list --limit 200 2>/dev/null
systemctl is-active cphulkd lfd
```

## WordPress (wp-cli, as the site's own user)

On cPanel servers call the PHP CLI binary explicitly, or wp-cli may run under CGI and refuse with "Only CLI access":

```sh
W="sudo -u <user> /opt/cpanel/ea-php82/root/usr/bin/php /usr/local/bin/wp --path=<docroot> --skip-plugins --skip-themes"
$W core version; $W core check-update --minor
$W core verify-checksums; $W plugin verify-checksums --all
$W plugin list --fields=name,status,version,update_version
$W user list --role=administrator --fields=ID,user_login,user_email,user_registered
$W option get updraft_central_localkeys   # report only whether it is non-empty
ls -la --time-style=long-iso <docroot>/wp-content/mu-plugins <docroot>/wp-content/plugins
find <docroot>/wp-content/uploads -name '*.php'
```

Skip `.wp-toolkit_*` clone directories when counting sites.

## Docker

```sh
docker ps --format '{{.Names}}\t{{.Image}}\t{{.CreatedAt}}\t{{.Ports}}'
docker images --format '{{.Repository}}:{{.Tag}}\t{{.CreatedAt}}'
for c in $(docker ps --format '{{.Names}}'); do
  printf '%s ' "$c"
  docker exec "$c" sh -c 'php -v 2>/dev/null | head -1; node -v 2>/dev/null; nginx -v 2>&1 | head -1; openssl version 2>/dev/null; grep PRETTY_NAME /etc/os-release; ls /lib/ld-musl* >/dev/null 2>&1 && echo musl || echo glibc' 2>/dev/null | tr '\n' ' '
  echo
done
# Next.js inside a container: version of next and sharp
docker exec <c> sh -c 'for d in /app /usr/src/app /home/node/app; do [ -f $d/node_modules/next/package.json ] && grep -m1 "\"version\"" $d/node_modules/next/package.json $d/node_modules/sharp/package.json; done'
trivy image <image>                        # if installed: OS and language packages in one pass
```

Never run `docker exec <c> env` or print `docker inspect` in full: both expose secrets.

## Python without running Python

```sh
find / -path '*/site-packages/*' -maxdepth 8 \( -name 'litellm*' -o -name 'cook.pth' -o -name 'certifi-*.dist-info' \) 2>/dev/null
```

Keep the search inside known Python locations on busy servers (`/usr/lib/python3*`, `/usr/local/lib/python3*`, virtualenvs, `~/.local`, uv and pipx tool directories). `sitecustomize.py` under `/usr/lib/python3*/` is normally owned by the distro: check with `dpkg -S` or `rpm -qf`.

## macOS (developer machine)

```sh
sw_vers; softwareupdate -l
HOMEBREW_NO_AUTO_UPDATE=1 brew outdated
ls ~/.nvm/versions/node ~/.fnm 2>/dev/null; volta list 2>/dev/null; node -v
ls ~/.local/share/uv/tools ~/.local/pipx/venvs 2>/dev/null
ls -la /Library/LaunchDaemons /Library/LaunchAgents ~/Library/LaunchAgents
ls -la /etc/ssl/cert.pem                   # system trust bundle date
for a in /Applications/*.app; do f="$a/Contents/Frameworks/Electron Framework.framework/Resources/Info.plist"; [ -f "$f" ] && echo "$(basename "$a") $(defaults read "$f" CFBundleVersion 2>/dev/null)"; done
```

## Windows (PowerShell)

```powershell
Get-ComputerInfo | Select-Object OsName, OsVersion
winget upgrade
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10
Get-ScheduledTask | Where-Object { $_.TaskPath -notlike '\Microsoft\*' }
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallDate
```

## From outside (your own hosts only)

```sh
curl -s -o /dev/null -w '%{http_code} %{content_type} %{size_download}\n' https://example.com/.git/HEAD
ssh -o PreferredAuthentications=password,keyboard-interactive -o PubkeyAuthentication=no -o ConnectTimeout=5 user@host true
nc -zv -w 3 host 3306
curl -s -o /dev/null -w '%{http_code}\n' --resolve example.com:443:<origin-ip> https://example.com/   # does the origin answer directly?
```

Run port and access checks from a machine that is not on any allowlist, and say which machine you used.
