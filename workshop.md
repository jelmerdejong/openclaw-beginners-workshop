# OpenClaw Beginner Workshop
**Participant guide for securely setting up OpenClaw on a VPS**

> - **Last reviewed:** 2026-05-15
> - **Stable target:** OpenClaw v2026.5.12
> - **Fresh VPS E2E status:** pending in this workspace. This guide was command-reviewed against the official OpenClaw docs and current stable GitHub release, but a disposable VPS and temporary provider credentials are still required for a full live run.
> - **Update policy:** if a PR changes version-sensitive setup, update this note in the same PR.

> **What you will leave with:** a secured Ubuntu VPS, an always-on OpenClaw Gateway, the Control UI opened through an SSH tunnel, a starter workspace, safe model and memory habits, Telegram pairing, optional web search, one reviewed skill, and a simple maintenance routine.

---

## How to use this guide

- Follow the sections in order.
- Replace placeholders like `YOUR_IP_ADDRESS` before running commands.
- Keep the loopback and pairing defaults unless you have a specific reason to widen access.
- Commands are labeled by where you run them: **Mac** means your laptop terminal, **VPS** means the remote Ubuntu server.

## Workshop prerequisites

You need these before the workshop starts:

- An Ubuntu VPS, preferably Ubuntu 24.04 LTS. Ubuntu 22.04+ should also work.
- Public IP address and root login from the VPS provider.
- A Mac with Terminal.
- An Anthropic API key for the default model setup.
- Optional but useful: an OpenAI API key for embeddings, images, and direct OpenAI API usage.
- A Telegram account if you want phone access.
- A Brave Search, Tavily, or other supported search API key if you want web search.
- A password manager for server passwords and API keys.

Recommended VPS size:

- Minimum: 2 vCPU / 2 GB RAM.
- Better: 2 vCPU / 4 GB RAM if you plan to use browser tools, multiple channels, or larger plugins.
- Disk: SSD-backed storage. The Gateway owns long-lived state and transcripts.

Cost reminder: model and search usage can grow quickly if an agent loops. Set provider spend limits before giving the agent broad tool access.

## Part 1: What OpenClaw is

OpenClaw is a self-hosted Gateway that connects chat channels such as Telegram, WhatsApp, Slack, Discord, and others to an AI agent runtime and a web Control UI.

"Self-hosted" means:

- You run the Gateway.
- You own the state directory and workspace files.
- Your selected model provider still receives prompts and tool outputs through its API or account integration.

The Gateway is the source of truth for:

- configuration
- credentials
- pairings
- sessions
- cron jobs
- workspace files

Treat the VPS as the machine where your assistant lives.

## Part 2: Security model

### One trusted operator boundary

The default OpenClaw model is personal-assistant security: one trusted operator boundary per Gateway.

That means:

- Use one VPS or OS user per person or trust boundary.
- Do not run one shared Gateway for mutually untrusted people.
- Anyone who can edit `~/.openclaw/` should be treated as an operator.
- If several people can message one tool-enabled agent, they can steer the same tool authority.

For a company team, a shared agent can be acceptable only when everyone is in the same business trust boundary and the runtime uses dedicated accounts, not anyone's personal browser or password manager.

### The baseline we want

For beginners, use this posture:

- Gateway bound to loopback.
- Control UI reached through SSH tunnel.
- Gateway auth enabled.
- DM access through pairing or explicit allowlist.
- Groups require mention.
- No public Gateway port.
- No random community skills without review.
- `openclaw security audit --deep` after setup and after major config changes.

### Main risks

1. Network exposure: binding the Gateway to a public interface or opening port `18789` to the internet.
2. Secrets leakage: storing API keys in workspace files, git repos, screenshots, or group chats.
3. Tool blast radius: letting untrusted messages drive shell, browser, filesystem, or message-send tools.
4. Prompt injection: web pages, emails, attachments, or other users can try to trick the model into ignoring your intent.

What helps:

- Keep inbound access tight.
- Prefer stronger current-generation models for tool-enabled work.
- Use web/search content as untrusted input.
- Keep dangerous tools disabled unless needed.
- Use sandboxing or approval prompts for risky execution.

## Part 3: Secure the VPS

Do this before installing OpenClaw.

### Step 3.1 - First SSH login

On your Mac:

```bash
ssh root@YOUR_IP_ADDRESS
```

If this is your first connection to the server, type `yes` when SSH asks whether to trust the host key.

When you type a password in Terminal, nothing appears on screen. That is normal. Paste or type the password and press Enter.

If you see a stale host-key error for a reused IP:

```bash
ssh-keygen -R YOUR_IP_ADDRESS
ssh root@YOUR_IP_ADDRESS
```

### Step 3.2 - Create the OpenClaw OS user

On the VPS, as root:

```bash
adduser openclaw
usermod -aG sudo openclaw
id openclaw
```

The `id` output should include `sudo` in the group list.

### Step 3.3 - Verify the new user before locking root

Open a second Terminal tab on your Mac:

```bash
ssh openclaw@YOUR_IP_ADDRESS
sudo -v
```

Only continue if both commands work. Keep this second session open while you change SSH settings.

### Step 3.4 - Disable root SSH login

On the VPS:

```bash
sudo nano /etc/ssh/sshd_config
```

Find or add these lines:

```text
PermitRootLogin no
MaxAuthTries 5
LoginGraceTime 60
```

Save with `Ctrl+X`, then `Y`, then Enter.

Validate and restart SSH:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

On your Mac, confirm root is blocked:

```bash
ssh root@YOUR_IP_ADDRESS
```

This should fail with `Permission denied`.

### Step 3.5 - Add fail2ban

On the VPS, logged in as `openclaw`:

```bash
sudo apt update
sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
```

Paste:

```ini
[sshd]
enabled  = true
port     = ssh
filter   = sshd
maxretry = 5
findtime = 600
bantime  = 3600
```

Save, then run:

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### Step 3.6 - Enable the firewall

On the VPS:

```bash
SSH_PORT=$(echo "$SSH_CONNECTION" | awk '{print $4}')
SSH_PORT=${SSH_PORT:-22}
echo "Allowing SSH on port ${SSH_PORT}"
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow "${SSH_PORT}/tcp"
sudo ufw enable
sudo ufw status verbose
```

This detects the SSH port used by your current session before enabling UFW. If your provider uses a non-standard SSH port, this avoids locking yourself out.

Important: do not open OpenClaw's Gateway port. We will use an SSH tunnel.

### Step 3.7 - Updates and reboot

On the VPS:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo reboot
```

After the reboot, reconnect from your Mac:

```bash
ssh openclaw@YOUR_IP_ADDRESS
```

## Part 4: Install and onboard OpenClaw

Official current requirements: Node 24 is recommended, and Node 22.14+ is supported. The installer handles Node for you on normal Linux installs.

### Step 4.1 - Install OpenClaw

On the VPS as `openclaw`:

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
```

If the installer does not launch onboarding automatically, run:

```bash
openclaw onboard --flow quickstart --install-daemon
```

### Step 4.2 - Onboarding choices

The wizard changes over time, so use intent rather than memorizing exact menu wording. For this workshop, you are running onboarding on the VPS itself, so choose the local Gateway path. In OpenClaw onboarding, local means local to the machine where the command runs.

Choose:

- QuickStart. It uses the safe defaults this workshop wants.
- Local Gateway on this VPS, not Remote mode.
- Default workspace location, unless you have a clear reason to change it.
- Gateway port `18789`.
- Loopback/local bind. Do not bind to a public interface.
- Gateway token auth enabled. QuickStart auto-generates a token even for loopback.
- Tailscale exposure off for now.
- Install/manage the Gateway daemon when asked.
- Anthropic as the initial model provider.
- Anthropic API key as the auth method. On a VPS, this is the simplest production path.
- A current Sonnet model as the default if offered.
- Skip channels for now. Telegram is set up later.
- Skip web search for now. Search is set up later with `openclaw configure --section web`.
- Skip recommended skills/plugins for now. We review skills manually later.

Do not choose Remote mode in this VPS session. Remote mode is for configuring the machine you are typing on to connect to a Gateway somewhere else; it does not install or modify the remote Gateway host.

If onboarding detects existing config, choose Keep or Modify only if you know where that config came from. On a fresh workshop VPS there should be no existing config. Do not choose Reset on a machine that already has useful OpenClaw state unless you have a backup.

Useful current defaults to recognize:

- `tools.profile: "coding"` for fresh local setups.
- `session.dmScope: "per-channel-peer"` when unset.
- token auth for the Gateway, even on loopback.
- pairing/allowlist-oriented DM defaults for chat channels.
- a final health check that starts the Gateway and verifies it is reachable.

### Step 4.3 - Verify the install

On the VPS:

```bash
openclaw --version
node --version
openclaw doctor
openclaw gateway status
openclaw health
```

Expected:

- `openclaw --version` should show `2026.5.12` or later stable.
- `node --version` should show Node 24 or at least Node 22.14+.
- `openclaw doctor` should not report critical migration or auth failures.
- `openclaw gateway status` should show the Gateway running.

If the Gateway is not running:

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway status
```

If commands feel slow on a tiny VPS, add OpenClaw's recommended startup cache environment:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

### Step 4.4 - Smoke test one agent turn

On the VPS:

```bash
openclaw agent --agent main --message "Reply with the active model and one sentence confirming the Gateway works."
```

If this fails, run:

```bash
openclaw doctor
openclaw models status
openclaw gateway status
```

The usual causes are missing model auth, the Gateway not running, or an onboarding step that did not finish.

## Part 5: Open the Control UI safely

### Step 5.1 - Start the SSH tunnel

On your Mac:

```bash
ssh -N -L 18789:127.0.0.1:18789 openclaw@YOUR_IP_ADDRESS
```

Leave this Terminal tab open. It will look idle while the tunnel is active.

### Step 5.2 - Open the UI

On your Mac, open:

```text
http://127.0.0.1:18789/
```

If you need an authenticated dashboard URL, run this on the VPS:

```bash
openclaw dashboard
```

Copy the URL it prints and open that URL on your Mac while the SSH tunnel is active.

If the UI asks for a token or password, use the Gateway auth value created during onboarding. If you cannot find it:

```bash
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
```

If a secret is redacted, do not paste config contents into chat or a public issue. Run `openclaw doctor` first and use `openclaw dashboard` to get a usable local URL.

### Step 5.3 - Run the security audit

On the VPS:

```bash
openclaw security audit --deep
```

If the audit reports permission issues:

```bash
openclaw security audit --fix
```

Then run:

```bash
openclaw security audit --deep
```

Manual permission baseline:

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
```

Treat everything under `~/.openclaw/` as sensitive. It can contain config, credentials, transcripts, auth profiles, pairing state, and tool output.

## Part 6: Build the starter workspace

OpenClaw works best when important context is written to files. The model may forget chat context after compaction, but workspace files persist.

Common workspace files:

| File | Purpose |
| --- | --- |
| `AGENTS.md` | Operating rules, priorities, and standing instructions |
| `SOUL.md` | Tone, style, boundaries, and personality |
| `USER.md` | Facts and preferences about you |
| `TOOLS.md` | Your tool conventions, accounts, IDs, and workflows |
| `IDENTITY.md` | Agent name and short self-description |
| `HEARTBEAT.md` | Small periodic checklist for heartbeat |
| `MEMORY.md` | Curated durable facts and decisions |
| `memory/YYYY-MM-DD.md` | Daily working notes |
| `LEARNINGS.md` | Optional rules learned from mistakes |

`LEARNINGS.md` is useful, but do not assume every custom file is automatically injected into every prompt. If a rule must be present every session, put the compact version in `AGENTS.md`.

### Step 6.1 - Confirm the workspace

On the VPS:

```bash
ls -la ~/.openclaw/workspace
```

If it does not exist:

```bash
mkdir -p ~/.openclaw/workspace/memory
```

### Step 6.2 - Introduce yourself

In the Control UI, send:

```text
I am [your name]. I will call you [agent name]. You are my personal AI assistant for [main use case]. Save the durable facts to USER.md and IDENTITY.md. Keep the files short and show me what changed.
```

### Step 6.3 - Run a guided onboarding interview

In the Control UI:

```text
Interview me one question at a time to build a concise starter workspace. Learn my goals, timezone, preferred tone, important tools, recurring work, and what you should be proactive about. Then draft or update AGENTS.md, SOUL.md, USER.md, TOOLS.md, MEMORY.md, HEARTBEAT.md, and LEARNINGS.md for review. Do not store secrets.
```

Review the files before trusting them. Short, explicit rules beat long prose.

### Step 6.4 - Make SOUL.md useful

In the Control UI:

```text
Read SOUL.md and make it specific. Remove corporate filler. Keep it under one page. Make the style direct, useful, and concise. Add a rule that you should not open with filler like "Great question" or "Absolutely"; just answer. Show me the diff before saving.
```

## Part 7: Models and cost control

### How selection works

OpenClaw selects:

1. The primary model.
2. Fallback models in order.
3. Provider auth failover inside a provider before moving to the next model.

Useful commands on the VPS:

```bash
openclaw models status
openclaw models list
openclaw models set anthropic/claude-sonnet-4-6
openclaw models fallbacks list
openclaw models aliases list
```

Useful chat commands:

```text
/model
/model list
/model status
/model anthropic/claude-opus-4-6
/usage tokens
```

### Beginner model ladder

Use the strongest model you can afford for tool-enabled or ambiguous work. Use cheaper models only for low-risk tasks.

| Tier | Example | Use for |
| --- | --- | --- |
| Deep reasoning | `anthropic/claude-opus-4-6` | Important decisions, ambiguous planning, security-sensitive tool use |
| Standard work | `anthropic/claude-sonnet-4-6` | Daily assistant work, writing, summarization, light research |
| Cheap background | provider's current small model | Simple heartbeat checks and low-risk cron summaries |
| Code/Codex | `openai-codex/*` or current OpenAI Codex route | Coding work when you have Codex auth configured |

Model names change. Before teaching a participant to pin a model, run:

```bash
openclaw models list
```

### OpenAI and Codex note

OpenAI API-key usage and Codex subscription usage are separate surfaces in current OpenClaw.

- Use OpenAI API-key onboarding for direct API usage, embeddings, images, speech, and similar non-agent OpenAI surfaces.
- Use Codex/OpenAI Code auth when you want ChatGPT/Codex subscription-backed coding agents.
- If a model route looks wrong after an update, run `openclaw update`, `openclaw doctor`, and `openclaw models status`.

Set up an OpenAI API key if you want OpenAI-backed memory embeddings:

```bash
openclaw onboard --auth-choice openai-api-key
openclaw models status
```

## Part 8: Memory that works

### Rules

1. If it must persist, write it.
2. Keep `AGENTS.md`, `SOUL.md`, and `USER.md` short.
3. Use daily notes for messy context.
4. Curate `MEMORY.md` weekly.
5. Do not store secrets in workspace files.
6. Search memory before answering non-trivial questions about past decisions.

Add this rule to `AGENTS.md`:

```text
Before answering questions about prior decisions, preferences, people, projects, or facts that may already be known, search memory/workspace first. If nothing relevant is found, say that briefly.
```

### Commands

On the VPS:

```bash
openclaw memory status
openclaw memory index
openclaw memory search "what did we decide about pricing?"
```

OpenClaw can auto-detect embedding providers from available API keys. Codex OAuth is not an embedding provider; use an OpenAI, Gemini, Voyage, Mistral, Bedrock, local, or other supported embedding setup for vector search.

### Optional session memory

Session transcript indexing is still experimental. It can help the agent find facts from old conversations, but it also means transcripts are more searchable. Treat disk access as the trust boundary.

In the Control UI:

```text
Check whether experimental session memory search is enabled. If not, explain the privacy tradeoff and show me the config change before enabling it. I want normal memory files enabled either way.
```

## Part 9: Telegram

### Step 9.1 - Create the bot

In Telegram:

1. Open `@BotFather`.
2. Run `/newbot`.
3. Choose a display name and username.
4. Save the bot token in your password manager.

Telegram's default bot privacy mode is fine for beginner setups where the bot only responds to mentions or commands. Disable privacy mode only if you deliberately need the bot to read all group messages.

### Step 9.2 - Add Telegram in OpenClaw

On the VPS:

```bash
openclaw onboard
```

Choose the path for adding a channel, select Telegram, and paste the bot token.

Keep:

- DM policy: `pairing`
- Groups: require mention
- Write actions: conservative defaults

### Step 9.3 - Pair your DM

Send a DM to your Telegram bot. It should reply with a pairing code.

On the VPS:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

Pairing codes are short-lived. If one expires, DM the bot again and approve the new code.

Important: DM pairing grants DM access only. It does not automatically authorize group commands. For groups, configure explicit group policy and allowlists.

### Step 9.4 - Safe group behavior

For beginner workshops:

- Require `@botname` mention in groups.
- Do not allow the bot in large public groups.
- Do not enable broad message-send or delete actions until you understand the blast radius.
- Put your numeric Telegram user ID in the allowlist if you want durable owner authorization.

To find your ID without third-party bots, DM your bot and inspect OpenClaw logs:

```bash
openclaw logs --follow
```

Look for Telegram's `from.id`.

## Part 10: Web search

Web search is optional. It is useful for cited research and current facts, but search results are untrusted input.

### Step 10.1 - Configure a provider

On the VPS:

```bash
openclaw configure --section web
```

Choose one provider. Brave and Tavily are straightforward beginner options.

Brave uses:

```text
BRAVE_API_KEY
```

Tavily uses:

```text
TAVILY_API_KEY
```

Prefer storing keys through onboarding/configuration or SecretRef. Avoid pasting keys into workspace files.

### Step 10.2 - Test search

In the Control UI:

```text
Search the web for the latest stable OpenClaw release. Return sources and dates.
```

If search fails:

```bash
openclaw doctor
openclaw configure --section web
```

Check that the Gateway process can see the relevant environment variable if you used env storage.

## Part 11: Skills

A skill is a folder with a `SKILL.md` file that teaches the agent how to use a tool, API, or workflow.

### Best practices

- Read third-party skills before enabling them.
- Keep skills short and operational.
- No hardcoded secrets.
- Read operations can be automatic.
- Write operations require explicit confirmation.
- Document rate limits, pagination, and common errors.
- Prefer sandboxed or tightly scoped execution for risky tools.

### Skill locations

OpenClaw loads bundled, managed, personal, project, and workspace skills. For this workshop, use workspace skills:

```text
~/.openclaw/workspace/skills/<skill-name>/SKILL.md
```

Workspace skills take precedence over lower-priority bundled or shared skills with the same name.

### Create one starter skill

In the Control UI:

```text
Create a simple OpenClaw skill for [SERVICE]. It should support [READ-ONLY TASKS]. Use env var [ENV_VAR] for auth. Never hardcode secrets. Any write action must require explicit confirmation. Put it under workspace/skills/[service]/SKILL.md and show me the file before I use it.
```

Review the file. Make the agent fix vague or unsafe instructions before using it.

### ClawHub

Use native OpenClaw commands:

```bash
openclaw skills search "calendar"
openclaw skills install <skill-slug>
openclaw skills update --all
```

ClawHub is public. Treat it as discovery, not trust. Check owner, source, version, required permissions, scan/moderation status, and the `SKILL.md` before enabling anything.

## Part 12: Automation

OpenClaw has several automation mechanisms. Beginners only need two:

- Heartbeat: periodic main-session check. Good for awareness and lightweight monitoring.
- Cron: precise scheduled tasks. Good for reminders, daily reports, weekly reviews, and isolated work.

### Heartbeat

Heartbeat runs periodic agent turns in the main session. Keep `HEARTBEAT.md` tiny.

Starter `HEARTBEAT.md` content:

```markdown
# HEARTBEAT

- Check whether anything in today's memory log needs a short follow-up.
- Report only important changes.
- Do not perform write actions unless I explicitly approved them earlier.
- If nothing needs attention, stay quiet.
```

### Cron

Cron jobs are stored on the Gateway and survive restarts.

Useful commands:

```bash
openclaw cron list
openclaw cron show <JOB_ID>
openclaw cron runs --id <JOB_ID>
```

Example one-shot reminder:

```bash
openclaw cron add \
  --name "Reminder" \
  --at "$(date -d 'tomorrow 09:00' '+%Y-%m-%dT%H:%M:%S%:z')" \
  --session main \
  --system-event "Reminder: review OpenClaw workshop notes" \
  --wake now \
  --delete-after-run
```

Prefer cron over heartbeat when exact timing matters or when you want isolated execution.

## Part 13: Secrets

Plaintext config still works, but SecretRefs are safer for longer-running setups.

Supported SecretRef shape:

```json
{ "source": "env", "provider": "default", "id": "OPENAI_API_KEY" }
```

Useful commands:

```bash
openclaw secrets audit
openclaw secrets configure
openclaw secrets apply
openclaw secrets reload
```

For a beginner workshop:

- It is acceptable to use onboarding and environment variables.
- Do not put secrets in `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `MEMORY.md`, or `skills/*/SKILL.md`.
- After the workshop, migrate long-lived keys to SecretRefs or a password-manager-backed `exec` provider.

## Part 14: Maintenance

Weekly on the VPS:

```bash
openclaw update
openclaw doctor
openclaw gateway restart
openclaw health
openclaw security audit --deep
sudo apt update
sudo apt upgrade -y
```

Before major upgrades:

```bash
openclaw backup create
openclaw backup verify
```

After updates, always read `openclaw doctor` output. It catches config migrations, stale auth routes, DM policy issues, and health problems.

## Troubleshooting

### Control UI does not load

Check:

```bash
openclaw gateway status
```

Confirm your Mac tunnel is still running:

```bash
ssh -N -L 18789:127.0.0.1:18789 openclaw@YOUR_IP_ADDRESS
```

Open exactly:

```text
http://127.0.0.1:18789/
```

### Unauthorized or token errors

Run:

```bash
openclaw dashboard
openclaw doctor
```

Use the dashboard URL printed by the Gateway while connected through the SSH tunnel.

### Gateway is not running

Run:

```bash
openclaw gateway status
openclaw gateway start
openclaw doctor
```

If using systemd user services:

```bash
systemctl --user status openclaw-gateway.service
```

### Telegram does not reply

Check:

```bash
openclaw gateway status
openclaw pairing list telegram
openclaw logs --follow
```

Common causes:

- Bot token is wrong.
- Gateway was not restarted after config.
- DM pairing was not approved.
- Bot was added to a group but group access was not configured.
- Telegram privacy mode prevents the bot from seeing non-mentioned group messages.

### Agent forgot something

Ask:

```text
Search memory for this topic. If it is not saved, write the durable decision to MEMORY.md and today's daily log.
```

Then run:

```bash
openclaw memory index
```

### Security audit reports critical findings

Fix in this order:

1. Public Gateway exposure or missing auth.
2. Open DM/group access combined with tools.
3. File permissions under `~/.openclaw`.
4. Browser or node execution exposure.
5. Unreviewed plugins or skills.

Then rerun:

```bash
openclaw security audit --deep
```

## End-of-workshop checklist

- [ ] You can SSH as `openclaw`.
- [ ] Root SSH login is disabled.
- [ ] `sudo -v` works for `openclaw`.
- [ ] fail2ban is running.
- [ ] UFW is enabled and only SSH is open.
- [ ] `openclaw --version` shows `2026.5.12` or later stable.
- [ ] `node --version` shows Node 24 or at least Node 22.14+.
- [ ] `openclaw gateway status` is healthy.
- [ ] `openclaw doctor` is clean or every warning is understood.
- [ ] Control UI works through SSH tunnel.
- [ ] `openclaw security audit --deep` is clean or every finding is understood.
- [ ] Workspace files exist and were reviewed.
- [ ] Model status and fallbacks are understood.
- [ ] Memory commands work.
- [ ] Telegram DM pairing works, if configured.
- [ ] Web search works, if configured.
- [ ] At least one skill was reviewed before use.
- [ ] You know how to update and back up the Gateway.

## Appendix: Advanced later

These are useful after the workshop, not during the first setup:

- Switch SSH from passwords to Ed25519 keys and then disable password auth.
- Use Tailscale Serve instead of SSH tunnels if you want easier private remote access.
- Move all long-lived secrets to SecretRefs.
- Add a private GitHub backup workflow for `~/.openclaw/workspace`.
- Enable sandboxing for risky tools.
- Use isolated cron jobs for daily reports and weekly reviews.
- Explore session memory indexing only after understanding transcript privacy.
- Evaluate QMD, dreaming, or third-party memory plugins only after the file-based memory setup is stable.
- Pair local Mac/iOS/Android nodes only when you understand that node execution is operator-level capability on that device.

## Sources reviewed for this refresh

- OpenClaw latest stable release: https://github.com/openclaw/openclaw/releases/latest
- Install docs: https://docs.openclaw.ai/install
- Onboarding docs: https://docs.openclaw.ai/start/onboarding-overview, https://docs.openclaw.ai/start/wizard, and https://docs.openclaw.ai/start/wizard-cli-reference
- Linux/VPS docs: https://docs.openclaw.ai/vps and https://docs.openclaw.ai/platforms/linux
- Security docs: https://docs.openclaw.ai/gateway/security
- Updating docs: https://docs.openclaw.ai/install/updating
- Pairing docs: https://docs.openclaw.ai/channels/pairing
- Telegram docs: https://docs.openclaw.ai/channels/telegram
- Models docs: https://docs.openclaw.ai/concepts/models and https://docs.openclaw.ai/providers/openai
- Memory docs: https://docs.openclaw.ai/reference/memory-config
- Web search docs: https://docs.openclaw.ai/tools/web and https://docs.openclaw.ai/tools/brave-search
- Skills and ClawHub docs: https://docs.openclaw.ai/tools/skills and https://docs.openclaw.ai/tools/clawhub
- Automation docs: https://docs.openclaw.ai/automation/index and https://docs.openclaw.ai/cli/cron
- Secrets docs: https://docs.openclaw.ai/gateway/secrets
