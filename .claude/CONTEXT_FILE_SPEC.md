# Plugin Builder Context File Specification

## Purpose

The `.plugin-builder-context.json` file maintains persistent state across all builder phases, preventing directory context loss and ensuring consistent plugin generation.

## Location

**Always created at repository root**: `/.plugin-builder-context.json`

## Schema

```json
{
  "mode": "A" | "B",
  "workspace_dir": "string (repo-relative path)",
  "plugin_root": "string (repo-relative path to plugin)",
  "plugin_display_name": "string",
  "plugin_slug": "string (kebab-case)",
  "module_name": "string (MUST differ from plugin_slug)",
  "plugin_type": "ui_only" | "service_only" | "ui_plus_service",
  "phase": "start" | "map" | "plan" | "build" | "verify" | "package",
  "created_at": "ISO datetime",
  "last_updated_at": "ISO datetime",
  "reference_pack_version": "string",
  "plugin_spec": {
    "display_name": "string",
    "description": "string",
    "version": "string (semver)",
    "target_users": "string",
    "features": ["array of strings"],
    "ui_pages": ["array of strings"],
    "settings": ["array of strings"],
    "data_storage": "none" | "local plugin state" | "external database",
    "external_apis": ["array of strings"],
    "non_goals": ["array of strings"],
    "acceptance_criteria": ["array of strings"]
  },
  "template_source": "string (path to PluginTemplate)",
  "lifecycle_manager_line_count": "number",
  "theme_bridge_included": "boolean",
  "build_plan_path": "string (path to build-plan.json)"
}
```

## Field Definitions

### Core Fields

- **mode**: Operating mode
  - `"A"`: Repo-aware (BrainDrive Core detected)
  - `"B"`: Standalone (using bundled Reference Pack)

- **workspace_dir**: Workspace directory relative to repo root
  - Mode A: `"plugin-build"` (prevents collision with installed plugins)
  - Mode B: `"plugin-test"` (standalone workspace)

- **plugin_root**: Full path to plugin being generated
  - Format: `"{workspace_dir}/{plugin_slug}"`
  - Example: `"plugin-build/task-manager"`

- **plugin_display_name**: Human-readable plugin name
  - Example: `"Task Manager Pro"`

- **plugin_slug**: Kebab-case identifier
  - Must match package.json name
  - Must match folder name
  - Example: `"task-manager-pro"`

- **module_name**: Module namespace (MUST differ from slug)
  - Used in lifecycle_manager.py module_data
  - Example: `"TaskManagerPro"` (PascalCase recommended)
  - MUST NOT equal plugin_slug

- **plugin_type**: Plugin architecture type
  - `"ui_only"`: Frontend-only plugin
  - `"service_only"`: Backend-only plugin (rare)
  - `"ui_plus_service"`: Full-stack plugin

### Phase Tracking

- **phase**: Current builder phase
  - `"start"`: Phase 0-1 (interview)
  - `"map"`: Phase 2 (compatibility mapping)
  - `"plan"`: Phase 3 (file planning)
  - `"build"`: Phase 4 (file generation)
  - `"verify"`: Phase 5 (validation)
  - `"package"`: Phase 6 (ZIP creation)

### Timestamps

- **created_at**: ISO 8601 timestamp of context file creation
- **last_updated_at**: ISO 8601 timestamp of last modification

### Reference Pack

- **reference_pack_version**: Version of PluginTemplate used
- **template_source**: Path to template being used
  - Mode A: `"./BrainDrive/backend/plugins/shared/PluginTemplate/v1.0.0"`
  - Mode B: `"./.claude/reference-pack/PluginTemplate-REAL"`

### Validation Fields

- **lifecycle_manager_line_count**: Line count of generated lifecycle_manager.py
  - Minimum: 900 lines (enforced during verification)
  - Used to detect truncation

- **theme_bridge_included**: Whether theme bridge was auto-included
  - Mandatory for ui_only and ui_plus_service types

- **build_plan_path**: Path to build-plan.json artifact
  - Created in Phase 3 (plan)
  - Used in Phase 4 (build) for strict adherence

## Usage Rules

### Every Command MUST:

1. **Load context file first**
   ```bash
   # Check if context exists
   if [ ! -f .plugin-builder-context.json ]; then
     echo "ERROR: Context file not found. Run /start first."
     exit 1
   fi
   ```

2. **Resolve plugin_root before any operations**
   ```bash
   PLUGIN_ROOT=$(jq -r '.plugin_root' .plugin-builder-context.json)
   cd "$PLUGIN_ROOT" || exit 1
   ```

3. **Update phase when complete**
   ```bash
   jq '.phase = "map" | .last_updated_at = now' .plugin-builder-context.json > tmp.json
   mv tmp.json .plugin-builder-context.json
   ```

### Workspace Safety Rules

- **Mode A**: Never generate inside `backend/shared_plugins`
  - Use `plugin-build/{slug}` instead
  - Prevents overwriting installed plugins

- **Mode B**: Use dedicated `plugin-test` workspace
  - Clean separation from other work

### Fail-Fast Behavior

If context file is missing and command is not `/start`:
```
ERROR: Plugin builder context not found.

This usually means you haven't started a plugin build session yet.

Please run: /start

If you were in the middle of a build, the context file may have been
accidentally deleted. You'll need to restart from /start.
```

## Example Context File

```json
{
  "mode": "B",
  "workspace_dir": "plugin-test",
  "plugin_root": "plugin-test/task-manager",
  "plugin_display_name": "Task Manager",
  "plugin_slug": "task-manager",
  "module_name": "TaskManager",
  "plugin_type": "ui_only",
  "phase": "build",
  "created_at": "2026-01-21T10:30:00Z",
  "last_updated_at": "2026-01-21T11:45:00Z",
  "reference_pack_version": "1.0.0",
  "plugin_spec": {
    "display_name": "Task Manager",
    "description": "A simple task management plugin",
    "version": "1.0.0",
    "target_users": "everyone",
    "features": [
      "Add tasks",
      "Mark tasks complete",
      "Delete tasks"
    ],
    "ui_pages": ["Dashboard"],
    "settings": ["theme", "task_limit"],
    "data_storage": "local plugin state",
    "external_apis": [],
    "non_goals": ["Calendar integration", "Team collaboration"],
    "acceptance_criteria": [
      "Can add tasks",
      "Can mark tasks complete",
      "Can delete tasks",
      "Settings are persistent"
    ]
  },
  "template_source": "./.claude/reference-pack/PluginTemplate-REAL",
  "lifecycle_manager_line_count": 1025,
  "theme_bridge_included": true,
  "build_plan_path": "plugin-test/task-manager/build-plan.json"
}
```

## Implementation Notes

### Creating Context (Phase 0-1: /start)

1. Detect mode (A or B)
2. Determine workspace_dir based on mode
3. Collect plugin spec from user
4. Generate plugin_slug from display_name
5. Ensure module_name differs from plugin_slug
6. Write context file to repo root
7. Initialize phase as "start"

### Updating Context (All Commands)

Commands must update:
- `phase`: When transitioning to next phase
- `last_updated_at`: On every modification
- Phase-specific fields as applicable

### Reading Context (All Commands)

```python
import json
from pathlib import Path

def load_context():
    context_path = Path('.plugin-builder-context.json')
    if not context_path.exists():
        raise FileNotFoundError("Context file not found. Run /start first.")

    with open(context_path) as f:
        return json.load(f)

def save_context(context):
    from datetime import datetime
    context['last_updated_at'] = datetime.utcnow().isoformat() + 'Z'

    with open('.plugin-builder-context.json', 'w') as f:
        json.dump(context, f, indent=2)
```

## Migration from Old System

Old system relied on:
- Current working directory (unreliable)
- Conversation memory (gets lost)
- Manual path specification each time

New system guarantees:
- Persistent state across sessions
- No directory confusion
- Fail-fast on missing context
- Audit trail via timestamps
