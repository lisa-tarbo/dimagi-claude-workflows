# Plugin Setup

1. Resolve paths from the manager directory determined by the SKILL.md that referred you here:
   - **Manager directory** (`<manager_dir>`): top-level. Holds `goals.md`, `.last_sync`, and any other standalone files the user adds over time.
   - **Daily directory** (`<daily_dir>`): `<manager_dir>/daily`. Holds daily journal entries (`YYYY-MM-DD.md`) from standup and shutdown.
2. Create directories if they don't exist:
   - If `<manager_dir>` doesn't exist, create it.
   - If `<daily_dir>` doesn't exist, create it.
   - If this is the first time the user is using the plugin (no journal files exist in `<daily_dir>` yet), mention where things are being saved so they know.
   - If the goals file (`<manager_dir>/goals.md`) doesn't exist yet, create it using the template at `${CLAUDE_PLUGIN_ROOT}/references/goals-template.md`.
3. The weekly review day is `${user_config.review_day}` (default: Friday).
