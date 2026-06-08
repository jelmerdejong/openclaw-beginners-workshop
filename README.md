# OpenClaw Beginner Workshop

Beginner-friendly, security-first workshop notes for setting up OpenClaw on a VPS.

This repository packages a participant-facing workshop for people who are new to OpenClaw and want a practical, safer default setup they can follow step by step.

Start here: [workshop.md](./workshop.md)

## What This Covers

The workshop walks through:

- securing a fresh Ubuntu VPS before installing OpenClaw
- installing OpenClaw with the current stable Linux path
- running the Gateway as an always-on service
- safely accessing the Control UI over an SSH tunnel
- setting up starter workspace files, memory habits, a daily operating loop, model routing, Telegram, web search, skills, and lightweight automation
- using a trust ladder and draft-and-approve approval queue for risky actions
- optional add-ons for Tailscale hardening and Obsidian-backed knowledge sharing
- maintaining, updating, backing up, and auditing a personal Gateway

## Who This Is For

- absolute beginners to OpenClaw
- people comfortable following copy/paste terminal steps
- people running a personal OpenClaw Gateway on Ubuntu 22.04+ or 24.04+
- Mac users for the laptop-side commands in the guide

## What You'll End Up With

- a secured VPS running an OpenClaw Gateway
- root SSH login disabled, fail2ban active, and UFW allowing SSH only
- a working Control UI accessed through SSH tunneling
- a starter workspace with safe defaults, memory habits, and first-week feedback loop
- a draft-and-approve safety posture for external or irreversible actions
- Telegram connected with pairing-based access, if configured
- a baseline setup for skills, search, heartbeat, cron, and weekly maintenance

## Prerequisites

- an Ubuntu 24.04 VPS recommended, or Ubuntu 22.04+ with a public IP and root access
- a Mac with Terminal
- an Anthropic API key
- optional: an OpenAI API key for embeddings, images, or direct OpenAI API usage
- optional: a Telegram account and bot token
- optional: a web search API key, such as Brave or Tavily
- optional: a Tailscale account and/or Obsidian if you want those add-ons

## How To Use This Repo

Read [workshop.md](./workshop.md) from top to bottom and work through the sections in order. Replace placeholders like `YOUR_IP_ADDRESS` before running commands, and keep the security defaults unless you have a clear reason to change them.

This is a living guide.

- Last reviewed: `2026-06-08`
- Stable target: `OpenClaw v2026.6.1`
- Fresh VPS E2E status: `pending in this workspace`

The guide has been refreshed against the official OpenClaw docs and current GitHub release. A full live run still requires a disposable Ubuntu VPS plus temporary model, Telegram, and search credentials.

If a future PR changes version-sensitive instructions, update those notes in the same PR.

## Contributing

Pull requests are welcome for typo fixes, clarity improvements, broken commands, safer defaults, live validation results, and OpenClaw version refreshes.

For larger restructures or scope changes, open an issue or draft PR first so the workshop stays stable for new readers.

## License

This workshop is licensed under [CC BY-NC 4.0](./LICENSE).
