# Minecraft-Website — CraftOracle

AI-powered Minecraft companion: generate, explain & fix Java Edition commands, find mods on Modrinth, and build modpacks with compatibility analysis.

**Live:** https://aathara-v.github.io/Minecraft-Website/

## Features
- ⚙️ **Command Generator** — free-text & guided modes, plus Explain and Fix tools
- 🧩 **Mod Finder** — Modrinth search with AI-optimized queries
- 📦 **Modpack Builder** — add mods, AI compatibility report, JSON/TXT export
- 🕐 **History** — last 50 queries per tool, re-run & delete

## Stack
- 100% client-side: a single `index.html`, no build step, no dependencies
- AI: NVIDIA NIM API (`z-ai/glm-5.2`) via `integrate.api.nvidia.com`
- Mods: Modrinth v2 API
- Deploys automatically via GitHub Pages

## Usage
Open the site and start typing — a hosted API key is bundled so it works out of the box.
You can also click **⚙ Key** to enter your own NVIDIA NIM key (saved to your browser's localStorage only).

> ⚠️ **Security note:** the bundled API key is visible to anyone who can view this repo/site.
> If you plan to keep the repo public, rotate/restrict that key in the NVIDIA build console,
> or remove the fallback and let each visitor supply their own key.
