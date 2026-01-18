# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## First Steps for New Sessions

**IMPORTANT**: At the start of each new session, read `/home/vntbu/wtg/aidungscripting.md` to understand the AI Dungeon Scripting API (hooks, params, functions, return values).

## Git Configuration

Always push commits under the username **thedenial** using the SSH alias `github-thisisasetuptomyrap` for the account `thisisasetuptomyrap-cmyk`. Don't add new files to the repo when pushing:

```bash
git push git@github-thisisasetuptomyrap:thisisasetuptomyrap-cmyk/World-Time-Generator-2.0.git
```

The remote is already configured in `.git/config` to use the correct SSH alias.

## Project Overview

World Time Generator (WTG) 2.0 is a scripting system for AI Dungeon that tracks time progression and manages game entities. It uses AI Dungeon's scripting hooks (onInput, onModelContext, onOutput) to process player commands and AI responses.

---

## Version Variants - Detailed Breakdown

**CRITICAL: Never mix scripts from different versions. Each version is a complete, self-contained system.**

### 1. `wtg_2.0/` - Full (Normal) Mode

The complete WTG implementation with all features including AI-driven entity generation.

#### Input Hook (`input copy.js`)
- Processes player commands: `[settime]`, `[advance]`, `[sleep]`, `[reset]`
- Parses entity markers from player input: `(Character)`, `((Location))`
- Maintains an entity blacklist (pronouns, commands) to prevent false positives
- Tracks trigger mentions across storycards
- Sets cooldowns after sleep/advance commands (8 hours for sleep, 5 minutes for advance)
- Strips parentheses from display text unless Debug Mode is enabled

#### Context Hook (`context copy.js`)
- **Adventure Erasing Detection**: Compares current action with stored turn data; if mismatch detected with low keyword similarity, triggers cleanup of undiscovered characters
- **Time Calculation**: Base rate 1 min/700 chars, modified by:
  - Time Duration Multiplier setting
  - Keyword similarity (high similarity = 0.7x slower, low similarity = 1.3x faster)
- **AI Instruction Injection**: Adds formatting rules requiring AI to use:
  - `(CharacterName)` for characters on first mention
  - `((LocationName))` for locations on first mention
  - `(((Entity) description)))` for detailed descriptions
- **Dynamic Time Scratchpad**: When enabled, instructs AI to use `(sleep X units)` and `(advance X units)` commands
- Cleans up storycards with future timestamps
- Deprecates entities no longer detected in recent history

#### Output Hook (`output copy.js`)
- **Entity Detection**: Parses AI output for parentheses markers BEFORE any text manipulation
- **Storycard Creation**:
  - Single entity: Creates card with 2 sentences before/after the mention
  - Multiple entities: Creates a "Discovery Action" card linking all entities
  - Checks for duplicate/variation names before creating new cards
- **Triple Parentheses Processing**: `(((Entity) description)))` injects description into existing card
- **AI Command Processing**: Extracts and applies `(sleep X units)` / `(advance X units)` with cooldown enforcement
- Updates turn data with action type, text, timestamp, generated entities, trigger mentions

#### Library (`library copy.js`)
Core functions for time calculations, storycard management, turn data tracking. Contains ~1900 lines of utility code.

---

### 2. `wtg_2.0_lightweight/` - Lightweight Mode

Simplified time tracking without AI prompt injection. Recommended for free users and Llama models.

#### Key Differences from Full Mode
- **NO entity detection** from parentheses markers
- **NO AI instruction injection** for entity formatting
- **NO entity blacklist processing**
- **NO discovery cards or entity deprecation**
- **Simpler time calculation**: Fixed rate 1 min/700 chars (no keyword similarity adjustments unless Dynamic Time enabled)

#### Input Hook (`input copy.js`)
- Processes same player commands: `[settime]`, `[advance]`, `[sleep]`, `[reset]`
- No entity marker parsing
- Sets cooldowns for sleep/advance
- ~180 lines vs ~360 lines in full mode

#### Context Hook (`context copy.js`)
- Adventure erasing detection (simplified - no keyword similarity check for cleanup)
- Time calculation with optional Dynamic Time keyword similarity
- Adds basic instruction to not recreate system commands
- Dynamic Time scratchpad (optional)
- ~175 lines vs ~270 lines in full mode

#### Output Hook (`output copy.js`)
- **NO entity detection block** (the ~280 line entity detection section is absent)
- AI command extraction for Dynamic Time (sleep/advance)
- Time calculation based on character count
- Timestamp injection for storycards whose keywords are mentioned
- Turn data tracking (simpler - no generated entities or trigger mentions)
- ~280 lines vs ~850 lines in full mode

---

### 3. `wtg_2.0_scenario/` - Scenario Mode (Runtime Switching)

Contains BOTH lightweight and normal mode code with runtime switching via `[light]` and `[normal]` commands.

#### Input Hook (`input copy.js`)
- All commands from both modes plus: `[light]`, `[normal]`
- Mode stored in `state.wtgMode` (defaults to 'lightweight')
- Entity marker processing only runs when NOT in lightweight mode (`if (!isLightweightMode())`)
- Cooldown initialization only in normal mode
- ~375 lines

#### Context Hook (`context copy.js`)
- **Dual code paths**: `if (isLightweightMode()) { ... } else { ... }`
- Lightweight path: Simple time calculation, no keyword similarity adjustments
- Normal path: Full keyword similarity, AI formatting instructions, entity deprecation
- ~355 lines

#### Output Hook (`output copy.js`)
- **Dual code paths** for lightweight vs normal processing
- Lightweight: Simple time tracking, no entity detection
- Normal: Full entity detection, discovery cards, triple parentheses processing
- Both share: AI command extraction, timestamp injection, turn data tracking
- ~585 lines

---

### 4. `autocards+wtg 2.0/` - Combined WTG Lightweight + AutoCards

Combines WTG Lightweight time tracking with **AutoCards**, a separate storycard generation system created by another developer. WTG processes first for time consistency, then AutoCards processes the result.

#### What is AutoCards?
AutoCards is an independent AI Dungeon scripting system that:
- Automatically generates storycards from AI output (without requiring parentheses markers)
- Tracks characters, locations, items, factions, etc.
- Has its own configuration system with settings like:
  - Card creation cooldown (default: 22 turns)
  - Bulleted list format for entries
  - Generated entry length limits
  - Memory update settings
- Provides automatic memory injection based on relevance

#### Input Hook (`input.js`)
- WTG commands processed first
- Then calls `AutoCards("input", modifiedText)` to process result
- No entity marker parsing (AutoCards handles storycard generation differently)

#### Context Hook (`context.js`)
- WTG time tracking first
- Adds WTG instructions to context
- Then calls `AutoCards("context", modifiedText, stop)` which returns `[text, stop]`
- AutoCards can modify stop sequences

#### Output Hook (`output.js`)
- WTG time/command processing first
- Turn data tracking
- Then calls `AutoCards("output", modifiedText)` for storycard generation
- AutoCards handles detecting entities and creating cards automatically

#### Library (`library.js`)
- Contains BOTH WTG library functions (~first 990 lines)
- Contains complete AutoCards system (~6000+ additional lines)
- AutoCards function starts at line 991 with extensive configuration options:
  - `DEFAULT_DO_AC`: Whether AutoCards is enabled at adventure start
  - `DEFAULT_CARD_CREATION_COOLDOWN`: Minimum turns between card generation
  - `DEFAULT_GENERATED_ENTRY_LIMIT`: Max length for generated entries
  - Many more settings for card types, memory updates, etc.

#### Key Difference from Other Versions
- **Does NOT use parentheses markers** for entity detection
- AutoCards uses its own AI-based detection system
- WTG only provides time tracking; AutoCards provides storycard generation
- The two systems operate independently but are chained together

---

## Script Execution Flow

Each hook processes in order:
1. **onInput** (`input.js`): Player types action -> script processes -> modified text sent to AI
2. **onModelContext** (`context.js`): Before AI generation -> injects instructions/context
3. **onOutput** (`output.js`): AI generates response -> script processes -> modified text shown to player

## Key Data Structures

- **Turn Time Format**: `00y00m00d00h00n00s` (years, months, days, hours, minutes, seconds)
- **Entity Markers** (Full/Scenario modes only): `(CharacterName)`, `((LocationName))`, `(((Entity) description)))`
- **System Storycards**: "Current Date and Time", "WTG Data", "World Time Generator Settings", "WTG Cooldowns"

## State Variables

Key `state` properties used across hooks:
- `state.turnTime` - Current turn time object `{years, months, days, hours, minutes, seconds}`
- `state.startingDate`, `state.startingTime` - Adventure start point
- `state.currentDate`, `state.currentTime` - Computed current time
- `state.wtgMode` - 'lightweight' or 'normal' (scenario mode only)
- `state.settimeInitialized` - Whether user has set initial time
- Cooldown tracking in WTG Cooldowns storycard

## Testing

Scripts can only be tested on the AI Dungeon website:
1. Copy all 4 scripts into a scenario's scripting editor
2. Use the Script Test panel or start a new adventure
3. Console logs appear in the Console Log panel (15-minute retention)
4. Use the Inspect button to view recent model context and game state

## Common Modifications

When modifying time logic, update both `context` and `output` scripts as they both handle time calculations.

When adding new settings, update:
1. `getWTGSettingsCard()` in library for default value
2. `getWTGBooleanSetting()` calls where the setting is used

Regex patterns for turn time markers must not use `$` anchors (allows matching with trailing characters).

For AutoCards modifications in the combined version, the AutoCards function starts at line 991 in `library.js`.
