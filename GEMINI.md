# Project-Rouge — Workspace & Global Working Rules

This file is automatically loaded by Antigravity on every PC that clones this repository.

## 1. Modern Web Stack & Infrastructure
- **Source Control (GitHub):** Repository under `mavintae-hue`. Always push completed work to `main` (or a feature branch when WIP across PCs).
- **Backend & Database (Supabase):** Use Supabase for PostgreSQL, Auth, Storage, and Edge Functions. Do not create a separate backend server. Keep schema changes as SQL migrations in `supabase/migrations/`.
- **Hosting & Secrets (Vercel):** Deploy via Vercel connected to GitHub `main`. Manage all secrets and environment variables (`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, etc.) in the Vercel Dashboard. Never hardcode secrets or commit `.env*` files.
- **No Localhost in Client Assets (when targeting shared LAN server):** Never hardcode `localhost` or `127.0.0.1` in client-facing links/APIs for XAMPP/LAN setups; use relative paths or environment-configured URLs.

## 2. Game Architecture: Project-Rouge
- **Genre:** D&D 5.5E-inspired turn-based Roguelike.
- **Map & Grid:** Isometric tile map with a **diamond-shaped grid** (`gridToScreen` / `screenToGrid` coordinate transforms, painter's algorithm depth sorting by `x + y`, A* pathfinding, and line-of-sight / fog-of-war).
- **Rules & Mechanics Reference:** Use the bundled `dnd-character-builder` skill in `.agents/skills/dnd-character-builder/` for D&D 5.5E classes, ability scores, action economy (Action, Bonus Action, Reaction, Movement), and combat formulas.

## 3. Code Quality & Data Standards
- **No Hardcoded User Paths:** Never hardcode machine-specific user paths (e.g. `C:\Users\...`). Use relative paths from project root or `os.path.join` / Node `path.resolve`.
- **Thai Language Encoding:**
  - CSV exports: always use `encoding='utf-8-sig'`.
  - JSON exports: always use `ensure_ascii=False` (or UTF-8 without escaping Thai characters).
- **File Backup:** When modifying existing files outside a clean Git commit, keep at most 1 `.bak` backup (overwriting the previous `.bak`). On a clean Git repo, rely on Git commits/diffs.
- **Documentation:** Summarize major completed milestones in an Obsidian-ready note (Name, Function, Logic, Maintenance).

## 4. Multi-PC Sync Workflow
1. **Before leaving PC 1:** `git add . && git commit -m "..." && git push origin main`
2. **First time on PC 2:**
   - `git clone <repo-url>`
   - `npm install`
   - `vercel login && vercel link && vercel env pull .env.local`
   - Sign in to Supabase & Vercel MCP plugins in Antigravity.
3. **Resuming on PC 2:** `git pull origin main`
