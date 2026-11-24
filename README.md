# Project-Rouge

A D&D 5.5E-inspired roguelike with an **isometric diamond-shaped tile grid**, built with **Supabase** and **Vercel**.

## Included Antigravity Customizations
- `GEMINI.md` — Project & global working rules (auto-loaded on any PC).
- `.agents/skills/dnd-character-builder/` — D&D 5.5E Character Builder skill & book catalog.

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
