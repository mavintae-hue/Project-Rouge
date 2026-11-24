# Project-Rouge

A D&D 5.5E-inspired roguelike with an **isometric diamond-shaped tile grid**, built with **Supabase** and **Vercel**.

## Included Antigravity Customizations
- `GEMINI.md` — Project & global working rules (auto-loaded on any PC).
- `skills-lock.json` — Lockfile for community skills installed via `npx skills`.
- `.agents/skills/dnd-character-builder/` — D&D 5.5E Character Builder skill & book catalog.
- `.agents/skills/roguelike/` — Run structure, seeded RNG streams, energy turn loop, and symmetric shadowcasting FOV.
- `.agents/skills/pixijs/` & `.agents/skills/pixijs-3d/` — PixiJS v8 rendering reference skills.
- `.agents/skills/frontend-design/` — Intentional dark-fantasy RPG HUD & UI design skill.
- `.agents/skills/isometric-diamond-grid/` — Project-Rouge 2:1 diamond grid math, depth sorting, PixiJS input/camera, A*, and FOV contract.
- `.agents/skills/dnd-tactical-combat/` — Deterministic D&D 5.5E (2024) grid combat, TurnBudget, Weapon Masteries, and damage/condition rules.

## Multi-PC Setup
1. Clone the repository:
   ```bash
   git clone https://github.com/mavintae-hue/Project-Rouge.git
   cd Project-Rouge
   ```
2. Install dependencies (once the app is scaffolded):
   ```bash
   npm install
   ```
3. Pull environment variables from Vercel:
   ```bash
   vercel login
   vercel link
   vercel env pull .env.local
   ```
4. Sign in to the **Supabase** and **Vercel** plugins in Antigravity.
