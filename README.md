# agent-security-prompts

If you write, ship or run code, in any language, on any stack, give this to your coding agent.

Copy-paste prompts and a daily skill that turn Claude Code, Codex, Gemini CLI or any coding agent into a careful security checker for your computers, servers, containers, apps and websites. The agent inventories what you have, shows you a plan as a table, and changes nothing until you approve it. Every prompt is tested on real systems before it is published.

## What's here

- **[`skills/patch-watch`](skills/patch-watch/SKILL.md)**: a skill your agent can run every day. It keeps an inventory of what you run, pulls new advisories from primary sources (CISA's known-exploited list, the OSV database, vendor security pages), matches them against your real versions, runs the standing checks, and tells you what to patch first. Read-only unless you approve a fix. Its [sources list](skills/patch-watch/references/sources.md) doubles as a reference of where security advisories actually come from.
- **`prompts/`**: one-off, copy-paste prompts for a specific wave of vulnerabilities. No install needed.

## Prompts

| Prompt | What it covers | Last verified |
|---|---|---|
| [2026-10-patch-wave](prompts/2026-10-patch-wave.md) | The October 2026 wave: PHP "PHP2Shell" (CVE-2026-109618, CVE-2026-109619), Node.js, Next.js, the eMudhra CA keys, the LiteLLM 1.107.0 backdoor, GitLab, NGINX, Apache, WordPress (core 7.1.3, UpdraftCentral, Rank Math PRO), n8n, OpenSSH, OpenSSL, AOMEI, CISA known-exploited additions. Plus the checks people skip after patching. | October 23, 2026 |

New prompts are added as new files in `prompts/`. Dated prompts (`YYYY-MM-topic.md`) cover a specific wave of vulnerabilities and are not updated after their month ends, except for corrections. See [CHANGELOG.md](CHANGELOG.md).

## Install the daily skill

Claude Code:

```sh
git clone https://github.com/alexoleynik22/agent-security-prompts.git
mkdir -p ~/.claude/skills && cp -r agent-security-prompts/skills/patch-watch ~/.claude/skills/
```

Then ask Claude Code: "run patch-watch". The first run builds your inventory in `~/.patch-watch/` and asks about anything it cannot see.

Codex, Gemini CLI and other agents: tell the agent to "follow `agent-security-prompts/skills/patch-watch/SKILL.md`". It reads the `references/` folder on its own.

## Run it every day at 5 AM

Scheduled runs only report. They never apply fixes on their own: you read the report and approve fixes in a normal session.

macOS or Linux, with cron (`crontab -e`):

```cron
0 5 * * * mkdir -p "$HOME/.patch-watch" && cd "$HOME" && claude -p "Run the patch-watch skill. Read-only: do not change anything. Save the report to ~/.patch-watch/reports/ and print the 'Do today' section." --allowedTools "Read,Write,Grep,Glob,WebFetch,Bash(ssh:*),Bash(curl:*),Bash(docker:*)" >> "$HOME/.patch-watch/cron.log" 2>&1
```

Adjust `--allowedTools` to the commands your machines need. Keep it narrow: a scheduled agent should be able to read, not to deploy. With Codex, use `codex exec "Follow ~/agent-security-prompts/skills/patch-watch/SKILL.md. Read-only. Save the report."` in the same cron line.

On a Mac that sleeps at night, cron skips the run. Use a launchd job instead (it runs at wake if 5 AM was missed). Save this as `~/Library/LaunchAgents/com.patchwatch.daily.plist`, replace `YOURNAME`, then run `launchctl load ~/Library/LaunchAgents/com.patchwatch.daily.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.patchwatch.daily</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/zsh</string><string>-lc</string>
    <string>claude -p "Run the patch-watch skill. Read-only. Save the report to ~/.patch-watch/reports/." --allowedTools "Read,Write,Grep,Glob,WebFetch,Bash(ssh:*),Bash(curl:*),Bash(docker:*)"</string>
  </array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>5</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardOutPath</key><string>/Users/YOURNAME/.patch-watch/launchd.log</string>
  <key>StandardErrorPath</key><string>/Users/YOURNAME/.patch-watch/launchd.log</string>
</dict>
</plist>
```

Windows: create a Task Scheduler task that runs daily at 5:00 AM with the same `claude -p ...` command.

To get the report on your phone, add one line to the prompt telling the agent how you want it delivered (for example, "send the 'Do today' section to my Telegram bot with the `tg` command"), using a tool you already have set up.

## How to use a one-off prompt

1. Open the prompt file and copy everything below the line.
2. Paste it into your agent, running on the machine that has access to your systems (your laptop with SSH keys, a server, a CI runner).
3. Read the plan it shows you. Approve the parts you want. It asks again before anything risky in production.

## Safety, in every prompt

- The agent only looks at machines you own or explicitly name. It does not scan anyone else.
- It asks before updating production packages, restarting services, rebuilding containers or deploying.
- It never prints secrets and never "fixes" certificate errors by disabling verification.
- If it finds signs of compromise, it stops, preserves the evidence and tells you.

Versions in each prompt were checked against vendor advisories on the date shown at the top of that prompt. Fixes ship daily, so every prompt tells the agent to re-check them first. These are checklists, not guarantees. You are responsible for what runs on your systems.

## License

MIT
