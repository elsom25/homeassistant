# AI Agent Guide for Home Assistant Configuration

This document provides guidance for AI agents (like ChatGPT, Claude, etc.) working on this Home Assistant configuration repository.

## Repository Overview

**Purpose**: Personal Home Assistant configuration  
**Owner**: Jesse McGinnis  
**Scope**: Home automation for a two-person household (Jesse + Andrew)  
**Scale**: 24 automations, 15 scripts, 7 blueprints, 800+ lines of automation config

## Configuration Standards (2025.12)

### Modern Syntax Requirements

This repository uses **Home Assistant 2024.8+ modern syntax**. Always use:

```yaml
# ✅ CORRECT - Modern syntax
triggers:
  - trigger: state
    entity_id: sensor.temperature
conditions:
  - condition: state
    entity_id: binary_sensor.presence
    state: "on"
actions:
  - action: light.turn_on
    target:
      entity_id: light.living_room
```

```yaml
# ❌ WRONG - Deprecated syntax (DO NOT USE)
trigger:
  - platform: state
    entity_id: sensor.temperature
condition:
  - condition: state
    entity_id: binary_sensor.presence
    state: "on"
action:
  - service: light.turn_on  # Use "action:" not "service:"
    entity_id: light.living_room
```

### Required Fields

All automations MUST have:
- `id` - Unique identifier (enables debugging traces and UI editing)
- `alias` - Human-readable name
- `description` - Brief explanation of what it does

All actions SHOULD have:
- `alias` - Descriptive label (shows up in automation traces for debugging)

### Notify Configuration

Use `services:` syntax with `service:` prefix for each member:

```yaml
# ✅ CORRECT
notify:
  - name: ALL_DEVICES
    platform: group
    services:
      - service: mobile_app_jesse_iphone
      - service: mobile_app_andrew_iphone

# ❌ WRONG - members: syntax does not exist
notify:
  - name: ALL_DEVICES
    platform: group
    members:
      - mobile_app_jesse_iphone
```

## Repository Structure

```
.
├── automations.yaml          # 24 automations, organized by category
├── scripts.yaml              # 15 reusable scripts
├── configuration.yaml        # Main config, input_selects, zones
├── templates.yaml            # Template sensors (e.g., sensor.home_status)
├── customize.yaml            # Entity customizations
├── scenes.yaml               # Scene definitions
├── groups.yaml               # Entity groups
├── secrets.dist.yaml         # Example secrets file (DO NOT edit secrets.yaml)
├── blueprints/
│   ├── automation/
│   │   └── homeassistant/
│   │       ├── presence_state_machine.yaml  # Custom blueprint
│   │       ├── motion_light.yaml
│   │       └── notify_leaving_zone.yaml
│   ├── script/
│   └── template/
├── themes/                   # UI themes
└── esphome/                  # ESPHome device configs
```

## Key Automation Categories

The `automations.yaml` file is organized into these sections:

1. **Light Mode Automations** - Adaptive lighting control (Manual, Sleep, Sun modes)
2. **Scheduled Tasks** - Time-based automations (e.g., litter robot cleaning)
3. **Notifications** - Alert automations (water leaks, doors left open)
4. **Presence State Machines** - Blueprint-based presence tracking (Jesse, Andrew)
5. **Seasonal Mode** - Date-based seasonal detection (Halloween, Christmas)
6. **Sun Phase Tracking** - Elevation-angle-based sun tracking (6 phases)
7. **Lighting Automations** - Sun-phase-triggered lighting
8. **Vacation Presence Simulation** - Realistic activity simulation when away
9. **Presence-Triggered Lighting** - Coming home / leaving automations
10. **Sunrise Alarm** - Winter-only bedroom sunrise simulation
11. **Eight Sleep Bed Control** - Vacation mode automation
12. **Assist Speaker Ducking** - Voice assistant audio management

## Important Patterns & Conventions

### 1. Presence States

This home uses a **5-state presence system** for each person:

- `Home` - Settled at home
- `Just Arrived` - Recently arrived (10min transition)
- `Just Left` - Recently left (10min transition)
- `Away` - Gone for a while
- `Extended Away` - Vacation mode (after 20 hours)

**Entities**:
- `input_select.jesse_status_dropdown`
- `input_select.andrew_status_dropdown`
- `sensor.home_status` (template sensor aggregating both people)

### 2. Sun Phase System

Uses **elevation angles** (not simple sunrise/sunset) for accurate seasonal lighting:

- **Night**: < -6° (full darkness)
- **Nautical Dawn**: -12° to -6° rising (early lightening)
- **Dawn**: -6° to 0° rising (civil twilight)
- **Day**: > 0° (full daylight)
- **Golden Hour**: 6° to 0° falling (pre-sunset)
- **Dusk**: 0° to -6° falling (sunset, civil twilight)

**Entity**: `input_select.sun_phase`

This is MORE accurate than simple sunrise/sunset because twilight duration changes with seasons.

### 3. Seasonal Mode

Automatically detected based on date:

- **Halloween**: October 31 (outdoor lights stay OFF at sunset)
- **Christmas**: December 1-25
- **Christmas Season**: November, December, January
- **Normal**: All other times

**Entity**: `input_select.seasonal_mode`

### 4. Reusable Scripts

Common patterns are extracted into scripts:

- `script.outdoor_lights_on` / `script.outdoor_lights_off` - Centralized outdoor light control
- `script.christmas_lights_on` - Conditional seasonal lights
- `script.good_morning` - Morning routine
- `script.reset_speakers` - Audio system reset
- `script.bedroom_sunrise` - Multi-stage sunrise alarm

**Always use existing scripts** instead of duplicating light entity lists.

### 5. Blueprints

The custom `presence_state_machine.yaml` blueprint manages per-person presence:
- Configurable delays (Just Arrived, Just Left, Away to Extended)
- Quick return logic (if left briefly, skip "Just Arrived")
- Vacation mode skip
- Uses `mode: restart` to auto-cancel timers

**Used by**: `presence_jesse` and `presence_andrew` automations

## Common Tasks

### Adding a New Automation

1. Choose the appropriate section in `automations.yaml`
2. Add with required fields:
   ```yaml
   - id: unique_automation_id
     alias: "Human-Readable Name"
     description: Brief explanation
     triggers:
       - trigger: state
         entity_id: sensor.example
     actions:
       - alias: "Descriptive action name"  # ← ALWAYS add this
         action: light.turn_on
   ```
3. Use existing scripts where possible (e.g., `script.outdoor_lights_on`)
4. Add inline comments for complex logic

### Adding a New Script

1. Add to `scripts.yaml` with modern syntax:
   ```yaml
   my_new_script:
     alias: "Script Name"
     description: What it does
     mode: restart  # or single, parallel, queued
     icon: mdi:icon-name
     sequence:
       - alias: "Step 1 description"
         action: light.turn_on
   ```

### Modifying Presence Logic

**DO NOT** directly modify `input_select.*_status_dropdown` states from automations.

**Instead**: The `presence_state_machine.yaml` blueprint handles all state transitions automatically based on `person.jesse` and `person.andrew` entity states.

**Only modify**: Blueprint parameters in the automation instances if you need different timings.

### Working with Seasonal Automations

Use the `seasonal_mode` input_select to conditionally trigger actions:

```yaml
- if:
    - "{{ states('input_select.seasonal_mode') in ['Christmas', 'Christmas Season'] }}"
  then:
    - action: script.christmas_lights_on
```

### Sun-Phase-Based Automations

Use `input_select.sun_phase` for lighting decisions:

```yaml
variables:
  is_bright: "{{ states('input_select.sun_phase') == 'Day' }}"
  is_dim: "{{ states('input_select.sun_phase') in ['Night', 'Nautical Dawn', 'Dawn', 'Golden Hour', 'Dusk'] }}"
```

## Anti-Patterns (DO NOT DO)

❌ **DO NOT use deprecated `service:` syntax** - Use `action:` instead  
❌ **DO NOT use deprecated `trigger:`/`condition:`/`action:` keys** - Use plural forms  
❌ **DO NOT duplicate entity lists** - Use scripts like `outdoor_lights_on`  
❌ **DO NOT create automations without `id` and `alias`**  
❌ **DO NOT skip `alias` on actions** - Makes debugging impossible  
❌ **DO NOT use simple sunrise/sunset triggers** - Use sun_phase elevation angles  
❌ **DO NOT edit `secrets.yaml`** - It's gitignored; use `secrets.dist.yaml` as template  
❌ **DO NOT commit changes without verifying YAML syntax**

## Debugging Tips

### Automation Traces

Since all automations have `id` fields, you can view execution traces in:
**Settings → Automations & Scenes → [Automation] → Traces**

The `alias` fields on actions show up in traces, making it easy to see which step failed.

### LSP Diagnostics

Before committing, verify YAML syntax. The LSP may show false positives for YAML tags like `!include` and `!secret` - these can be ignored.

### Testing Automations

When testing changes:
1. Check automation traces for execution flow
2. Monitor Home Assistant logs for errors
3. Test edge cases (e.g., arriving home at different times of day)

## Migration Notes

### From Node-RED (December 2024)

This repository recently migrated from Node-RED to native Home Assistant automations:

- **Node-RED "Set house status" flow** → `presence_state_machine.yaml` blueprint
- **Node-RED "Time- & Presence-based" flow** → Multiple automations in categories above
- **Improvements made**:
  - Replaced simple sunrise/sunset with elevation-angle sun phases
  - Consolidated vacation routines into single automation
  - Extracted reusable scripts for DRY code
  - Modern syntax throughout

The old Node-RED flows are documented in git history but should not be referenced for new work.

## Contact & Ownership

**Repository Owner**: Jesse McGinnis  
**Household**: Two people (Jesse + Andrew)  
**Home Location**: Zone configured in `configuration.yaml` (lat/long in secrets)

## Version History

- **2024.8+**: Migrated to modern `triggers`/`actions` syntax
- **2025.12**: Current Home Assistant version (as of this writing)
- **December 2024**: Removed Node-RED dependency
- **January 2025**: Added comprehensive action aliases and modern notify syntax

## Additional Resources

- [Home Assistant Automation Docs](https://www.home-assistant.io/docs/automation/)
- [Home Assistant Script Docs](https://www.home-assistant.io/docs/scripts/)
- [Home Assistant Blueprint Docs](https://www.home-assistant.io/docs/blueprint/)
- [Modern Syntax Guide](https://www.home-assistant.io/docs/automation/yaml/)

---

**Last Updated**: January 2025  
**For AI Agents**: Always follow the modern syntax patterns shown in this guide. When in doubt, examine existing automations in the same category for reference patterns.
