# mod-shared-professions

An **AzerothCore** module that synchronizes profession skills and recipes account-wide. All characters on the same account benefit from the highest skill level, tier, and learned recipes across all characters, provided they have the profession learned.

## Project Overview

- **Purpose:** Automate the sharing of profession progression (primary and optionally secondary) across all characters on a single account.
- **Technologies:** 
  - **C++:** Core module logic using the AzerothCore scripting API.
  - **SQL (MySQL/MariaDB):** Persistent storage for account-wide skill and spell data.
  - **AzerothCore:** The underlying World of Warcraft server emulator framework.
- **Architecture:**
  - **Hooks:** Uses `PlayerScript` hooks (`OnPlayerLogin`, `OnPlayerLearnSpell`, `OnPlayerUpdateSkill`) and `WorldScript` hooks (`OnAfterConfigLoad`).
  - **Data Model:** Uses two tables in the `acore_characters` database:
    - `shared_professions_account_skills`: Stores the highest `value`, `max_value`, and `step` (tier) for each skill per account.
    - `shared_professions_account_spells`: Stores learned recipe IDs per account.
  - **Concurrency:** Employs a `SyncGuard` (mutex-protected set of GUIDs) to prevent recursive synchronization events during login or training.

## Building and Running

This module must be built as part of an AzerothCore source tree.

### Installation
1. Clone/Place this repository into the `modules/` directory of your AzerothCore source.
2. Re-run CMake to include the module in the build configuration.
3. Build the `worldserver` target.

### Setup
1. Apply the SQL scripts found in `data/sql/db-characters/` to your characters database (usually `acore_characters`).
2. Copy `conf/mod_shared_professions.conf.dist` to your server's configuration directory (e.g., `etc/`) and rename it to `mod_shared_professions.conf`.

### Configuration
Adjust settings in `mod_shared_professions.conf`:
- `SharedProfessions.Enable`: Set to `1` (default) to enable the module.
- `SharedProfessions.SyncSecondary`: Set to `1` (default) to include Cooking, First Aid, and Fishing in the synchronization.

## Development Conventions

- **Hook Management:** Always use the `SyncGuard` pattern when performing batch updates to a player's skills or spells to avoid triggering other hooks recursively.
- **Database Operations:**
  - Use `CharacterDatabase.Execute` for updates/inserts and `CharacterDatabase.Query` for fetches.
  - Leverage `INSERT ... ON DUPLICATE KEY UPDATE` with `GREATEST()` to ensure that only progression (increases) is saved to the account store.
- **Skill Identification:** Use `IsPrimaryProfessionSkill` (from AC core) and the internal `IsSyncedProfessionSkill` helper to filter which skills should be synchronized.
- **Recipe Logic:** Recipes are synchronized by mapping `spell_id` to its associated `skillId` via `sSpellMgr->GetSkillLineAbilityMapBounds`.

## Key Files
- `src/SharedProfessions.cpp`: Main implementation containing the `PlayerScript` and `WorldScript` logic.
- `src/mod_shared_professions_loader.cpp`: Entry point for AzerothCore to load the module's scripts.
- `data/sql/db-characters/*.sql`: Database schema definitions for account-wide storage.
- `conf/mod_shared_professions.conf.dist`: Template for module configuration.
