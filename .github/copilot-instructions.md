# Copilot Instructions for MusicName SourceMod Plugin

## Repository Overview

This repository contains **MusicName**, a SourceMod plugin for Source engine games that displays the names of currently playing music/songs in chat. The plugin hooks into ambient sound events to detect music playback and announces song names to players based on configurable mappings.

### Key Features
- Automatic music detection via ambient sound hooks
- Per-map configuration files for song name mappings
- Player toggle controls for music name display
- Multi-language support via translation files
- Cooldown system to prevent spam
- Admin commands for configuration management

## Technical Environment

### Core Technologies
- **Language**: SourcePawn
- **Platform**: SourceMod 1.12+ (minimum required version)
- **Build System**: SourceKnight (modern SourceMod build tool)
- **Compiler**: Latest SourcePawn compiler via SourceKnight
- **Dependencies**:
  - `multicolors` - For colored chat messages
  - `utilshelper` - Utility functions
  - `sourcemod` - Core SourceMod API

### Development Tools
- **Build Tool**: `sourceknight` (configured via `sourceknight.yaml`)
- **CI/CD**: GitHub Actions (`.github/workflows/ci.yml`)
- **Package Management**: Automated dependency resolution via SourceKnight

## Project Structure

```
├── addons/sourcemod/
│   ├── scripting/
│   │   └── MusicName.sp          # Main plugin source code
│   └── translations/
│       └── MusicName.phrases.txt # Multi-language translations
├── .github/
│   └── workflows/
│       └── ci.yml                # CI/CD pipeline
├── sourceknight.yaml            # Build configuration
└── README.md                     # Documentation
```

### Configuration Files (Runtime)
The plugin expects configuration files at runtime in:
```
configs/musicname/<mapname>.cfg   # Per-map song name mappings
```

## Code Style & Standards

### SourcePawn Best Practices
```sourcepawn
#pragma newdecls required        // Always use new-style declarations
#pragma semicolon 1              // Require semicolons

// Variable naming conventions:
char g_sCurrentSong[256];        // Global strings: g_s + PascalCase
bool g_bConfigLoaded;            // Global booleans: g_b + PascalCase
StringMap g_songNames;           // Global objects: g_ + PascalCase
float g_fCooldownTime;           // Global floats: g_f + PascalCase

// Function naming:
public void OnPluginStart()      // Public functions: PascalCase
void LoadConfig()                // Private functions: PascalCase
```

### Memory Management
```sourcepawn
// Always use delete for cleanup (never check null first)
delete kv;                       // Direct deletion
delete snap;                     // StringMap snapshots

// NEVER use .Clear() on StringMap/ArrayList - creates memory leaks
g_songNames.Clear();             // ❌ WRONG - memory leak
delete g_songNames;              // ✅ CORRECT
g_songNames = new StringMap();   // ✅ Create new instance
```

**⚠️ Current Code Issue**: The existing code uses `.Clear()` methods which should be replaced with `delete` and re-instantiation to prevent memory leaks.

### Indentation & Formatting
- Use **4 spaces** for indentation (tabs set to 4 spaces)
- No trailing whitespace
- Descriptive variable and function names
- Minimal comments (code should be self-documenting)

## Development Workflow

### Building the Plugin
```bash
# Using SourceKnight (recommended)
sourceknight build

# Manual compilation (if needed)
spcomp -i"path/to/includes" MusicName.sp
```

### Testing Workflow
1. **Build**: Use `sourceknight build` to compile
2. **Deploy**: Copy compiled `.smx` to test server
3. **Test**: Load plugin and verify functionality
4. **Validate**: Check for memory leaks using SourceMod profiler

### CI/CD Pipeline
- **Trigger**: Push to main/master or pull requests
- **Process**: Automated build via SourceKnight → Package creation → Release
- **Artifacts**: Compiled plugin + translations packaged for distribution

## Common Patterns & Architecture

### Plugin Lifecycle
```sourcepawn
public void OnPluginStart() {
    // Register commands
    RegConsoleCmd("sm_np", Command_NowPlaying, "Description");
    
    // Hook events
    HookEvent("round_start", OnRoundStart, EventHookMode_Pre);
    AddAmbientSoundHook(Hook_AmbientSound);
    
    // Load translations
    LoadTranslations("MusicName.phrases");
    
    // Create ConVars
    g_cvCooldownTime = CreateConVar("sm_musicname_cooldown", "5.0", "Description");
}
```

### Configuration Loading Pattern
```sourcepawn
public void LoadConfig() {
    g_bConfigLoaded = false;
    // TODO: Replace .Clear() with delete + new instantiation to prevent memory leaks
    g_songNames.Clear();                   // Current code - should be improved
    
    // Better approach:
    // delete g_songNames;
    // g_songNames = new StringMap();
    
    char configPath[PLATFORM_MAX_PATH];
    BuildPath(Path_SM, configPath, sizeof(configPath), "configs/musicname/%s.cfg", g_sCurrentMap);
    
    KeyValues kv = new KeyValues("music");
    if (!kv.ImportFromFile(configPath)) {
        delete kv;
        return;                            // Early return on failure
    }
    
    if (!kv.GotoFirstSubKey(false)) {
        delete kv;
        LogError("[MusicNames] Invalid config formatting for %s", g_sCurrentMap);
        return;
    }
    
    // Process KeyValues...
    delete kv;                             // Always cleanup
    g_bConfigLoaded = true;
}
```

### Translation Usage
```sourcepawn
// Always use translation keys with prefix
CPrintToChat(client, "%t %t", "Chat Prefix", "Now Playing", songName);
```

## Domain-Specific Knowledge

### Music Detection Logic
- Hooks `AddAmbientSoundHook()` to catch sound events
- Filters out volume 0 events (music stop commands)
- Extracts filename from sound path
- Matches against configured song mappings
- Implements cooldown to prevent duplicate announcements

### Configuration Format
```yaml
"music"
{
    "filename.mp3"     "Artist - Song Title"
    "boss_music.mp3"   "Epic Boss Battle Theme"
}
```

**Important**: 
- Filenames must be lowercase
- File extensions are required
- One config file per map: `<mapname>.cfg`

## Debugging & Troubleshooting

### Common Issues
1. **Config not loading**: Check file path and KeyValues format
2. **Songs not detected**: Verify filename casing and extensions
3. **Memory leaks**: Ensure all `new` objects have corresponding `delete`
4. **Translation errors**: Verify phrase keys exist in `.phrases.txt`

### Debugging Commands
```sourcepawn
RegConsoleCmd("sm_mn_dump", Command_DumpMusicnames);    // List all configured songs
RegAdminCmd("sm_mn_reload", Command_ReloadMusicnames);  // Reload config
```

## Performance Considerations

### Optimization Guidelines
- **String operations**: Minimize in frequently-called functions (ambient sound hooks)
- **Memory allocation**: Reuse StringMaps where possible
- **Database queries**: Not applicable (this plugin uses file-based config)
- **Timers**: Use minimal timers; prefer event-driven architecture

### Complexity Targets
- Aim for O(1) lookups using StringMap
- Avoid O(n) loops in sound hook callbacks
- Cache frequently accessed data

## Security & Validation

### Input Validation
- File paths: Use `BuildPath()` for safe path construction
- User input: Not applicable (no user-provided data storage)
- Config parsing: KeyValues handle malformed data gracefully

### Best Practices
- No SQL injection concerns (file-based config)
- No network requests (local file operations only)
- Validate config file existence before parsing

## Dependencies & Updates

### Updating Dependencies
```yaml
# sourceknight.yaml
dependencies:
  - name: sourcemod
    version: 1.13.0-git7221  # Update version as needed
  - name: multicolors
    type: git                # Always latest from git
  - name: utilshelper
    type: git                # Always latest from git
```

### Compatibility Requirements
- **Minimum SourceMod**: 1.12+
- **Game Support**: All Source engine games with ambient sound support
- **Platform**: Linux/Windows servers

## Common Modifications

### Adding New Commands
```sourcepawn
public void OnPluginStart() {
    RegConsoleCmd("sm_newcmd", Command_NewCommand, "Description");
}

public Action Command_NewCommand(int client, int args) {
    // Command implementation
    return Plugin_Handled;
}
```

### Extending Configuration
```sourcepawn
// Add new KeyValues sections in LoadConfig()
if (kv.JumpToKey("newsection")) {
    // Process new section
}
```

### Adding Translation Support
```text
// In MusicName.phrases.txt
"New Phrase"
{
    "en"  "English text"
    "fr"  "French text"
}
```

## Testing Guidelines

### Manual Testing Checklist
- [ ] Plugin loads without errors
- [ ] Commands respond correctly (`!np`, `!togglenp`, `!mn_dump`)
- [ ] Music detection works on test map
- [ ] Translations display properly
- [ ] Config reload functions correctly (`!mn_reload`)
- [ ] Player toggles work as expected
- [ ] Cooldown system prevents spam

### Testing Commands
```sourcepawn
// In-game testing commands (console or chat):
sm_np                    // Test current song display
sm_togglenp              // Test player preference toggle
sm_mn_dump               // Test song list display (console output)
sm_mn_reload             // Test config reload (admin only)

// Server console debugging:
sm plugins list          // Verify plugin is loaded
sm_dump_handles          // Check for memory leaks
```

### Performance Testing
- Monitor server tick rate impact
- Check memory usage over time
- Verify no memory leaks with `sm_dump_handles`

---

**Note**: This plugin follows modern SourceMod development practices. When making changes, always maintain backward compatibility and test thoroughly on a development server before deploying to production.