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
