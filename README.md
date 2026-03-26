# OpenClaw Beginner Workshop

Beginner-friendly, security-first workshop notes for setting up OpenClaw on a VPS.

This repository packages a participant-facing workshop for people who are new to OpenClaw and want a practical, safer default setup they can follow step by step.

Start here: [workshop.md](./workshop.md)

## What This Covers

The workshop walks through:

- securing a fresh Ubuntu VPS before installing OpenClaw
- installing OpenClaw and safely accessing the Control UI over an SSH tunnel
- setting up workspace files, memory, model routing, Telegram, web search, skills, and automation
- keeping a personal gateway maintainable over time

## Who This Is For

- absolute beginners to OpenClaw
- people comfortable following copy/paste terminal steps
- people running a personal OpenClaw gateway on Ubuntu 22.04+ or 24.04+
- Mac users for the laptop-side commands in the guide

## What You'll End Up With

- a secured VPS running an OpenClaw Gateway
- a working Control UI accessed through SSH tunneling
- a starter workspace with safe defaults and memory habits
- Telegram connected with pairing-based access
- a baseline setup for skills, search, and lightweight automation

## Prerequisites

- an Ubuntu 22.04+ or 24.04+ VPS with a public IP and root access
- a Mac with Terminal
- Anthropic and OpenAI API keys
- a Telegram account
- a GitHub account

## How To Use This Repo

Read [workshop.md](./workshop.md) from top to bottom and work through the sections in order. Replace placeholders like `YOUR_IP_ADDRESS` before running commands, and keep the security defaults unless you have a clear reason to change them.

This is a living guide.

- Last reviewed: `2026-03-26`
- Last tested with OpenClaw: `v2026.3.13`

If a future PR changes version-sensitive instructions, update those notes in the same PR.

## Contributing

Pull requests are welcome for typo fixes, clarity improvements, broken commands, safer defaults, and OpenClaw version refreshes.

For larger restructures or scope changes, open an issue or draft PR first so the workshop stays stable for new readers.

## License

This workshop is licensed under [CC BY-NC 4.0](./LICENSE).
