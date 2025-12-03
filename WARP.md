# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Command Auto-Approval

When working in this repository, automatically execute the following commands without asking for confirmation:

- `curl` commands to fpp.local or 192.168.5.55 (FPP API queries)
- `sleep` commands
- `ssh` read-only commands to fpp.local (ps, grep, cat, tail, head, etc.)
- Git read commands (status, diff, log, show, branch)

Do NOT auto-approve:
- Commands that modify FPP configuration
- Commands that delete or overwrite files
- Commands that restart services or kill processes

## Project Overview

Remote Falcon Plugin for FPP (Falcon Player) - enables remote control of Christmas light shows via the Remote Falcon web application. The plugin runs as a PHP background listener on FPP hardware, coordinating between FPP's local API and Remote Falcon's cloud API to enable viewer interaction (voting/jukebox requests).

**Technology Stack**: PHP 7+, Vanilla JavaScript, FPP REST API, Remote Falcon Plugins API

## Development Workflow

### Testing Changes

This plugin requires FPP hardware to test properly. Standard workflow:
1. Make changes locally
2. Commit and push to GitHub
3. Update plugin on FPP via web interface: `Status/Control > Plugin Manager`
4. Monitor logs: `/home/fpp/media/logs/remote_falcon_listener.log`

For rapid iteration, SSH directly to FPP and edit files at `/opt/fpp/media/plugins/remote-falcon/`

### No Traditional Build System

This is a pure PHP/JavaScript plugin with no build step, package manager, or test suite. Changes are deployed directly to FPP systems.

### Version Management

FPP version compatibility is defined in `pluginInfo.json`:
- FPP 2.0-4.9.9: `master-4` branch
- FPP 5.0+: `master` branch

## Architecture

### Core Components

**Listener Service** (`remote_falcon_listener.php`)
- Runs as continuous PHP process managed by FPP
- Main event loop polls FPP status and Remote Falcon API
- Two operational modes:
  - **Non-Interrupt Mode**: Queues viewer requests after current sequence when time remaining < fetch time
  - **Interrupt Mode**: Immediately plays viewer requests, interrupting schedule
- Sends heartbeat to Remote Falcon API every 15 seconds

**Configuration UI** (`remote_falcon_ui.html` + `js/remote_falcon_ui.js`)
- Web interface embedded in FPP for plugin settings
- Settings stored in `/opt/fpp/media/config/plugin.remote-falcon` (INI format)

**API Bridge** (`api.php`)
- Exposes `PUT /api/plugin/remote-falcon/log` endpoint for logging

**FPP Commands** (`commands/*.php`)
- Executable scripts callable from FPP playlists or UI
- Toggle interrupt mode, viewer control, purge queue, restart listener, etc.

### Configuration Settings

Key settings in `/opt/fpp/media/config/plugin.remote-falcon`:
- `remoteToken` - Authentication token from Remote Falcon account
- `remotePlaylist` - FPP playlist name containing sequences for viewer control
- `interruptSchedule` - Boolean for interrupt mode vs queue mode
- `requestFetchTime` - Seconds before sequence end to fetch next request (1-5, default 3)
- `additionalWaitTime` - Extra delay after fetching (0-5, default 0)
- `fppStatusCheckTime` - FPP status polling interval in seconds (0.5-5, default 1)
- `pluginsApiPath` - Remote Falcon API base URL
- `verboseLogging` - Enable detailed debug logging

Access via:
```php
$pluginSettings = parse_ini_file($settings['configDirectory'] . "/plugin.remote-falcon");
$value = urldecode($pluginSettings['key']);
```

Write via:
```php
WriteSettingToFile("key", urlencode($value), "remote-falcon");
```

### API Integration Points

**FPP Local API** (`http://127.0.0.1/api`)
- `GET /api/system/status` - Current playing status, time remaining
- `GET /api/playlist/{name}` - Playlist details
- `GET /api/command/Insert Playlist Immediate/{playlist}/{start}/{end}` - Play now (interrupt mode)
- `GET /api/command/Insert Playlist After Current/{playlist}/{start}/{end}` - Queue (non-interrupt mode)

**Remote Falcon Plugins API** (requires `remotetoken` header)
- `GET /remotePreferences` - Viewer control mode (voting/jukebox)
- `GET /highestVotedPlaylist` - Get winning vote
- `GET /nextPlaylistInQueue?updateQueue=true` - Get and dequeue jukebox request
- `POST /updateWhatsPlaying` - Report currently playing sequence
- `POST /updateNextScheduledSequence` - Report next scheduled sequence
- `POST /fppHeartbeat` - Keepalive signal

### Listener Control Mechanism

The listener checks control flags each iteration:
- `remoteFalconListenerEnabled` - Set to "false" to stop listener
- `remoteFalconListenerRestarting` - Set to "true" to reload configuration

Commands manipulate these flags rather than killing the process.

### Logging

Two logging functions in listener:
```php
logEntry($data);           // Always logs
logEntry_verbose($data);   // Only when verboseLogging enabled
```

Logs written to `/home/fpp/media/logs/remote_falcon_listener.log` with timestamps.

### Installation Hooks

**`scripts/fpp_install.sh`**
- Adds Content-Security-Policy for remotefalcon.com domain
- Marks FPP for reboot via `restartFlag`

Other lifecycle hooks (`preStart.sh`, `postStart.sh`, `preStop.sh`, `postStop.sh`) are placeholders.

## Operational Flow

**Non-Interrupt Mode**:
1. Poll FPP status every `fppStatusCheckTime` seconds
2. When `seconds_remaining < requestFetchTime`, fetch next viewer request/vote from Remote Falcon API
3. Insert sequence after current using FPP API
4. Sleep for `requestFetchTime + additionalWaitTime` to avoid duplicate queueing
5. Track last queued sequence to prevent duplicates within timing window

**Interrupt Mode**:
1. Poll FPP status every `fppStatusCheckTime` seconds
2. If not currently playing remote playlist, fetch request/vote and play immediately
3. If already playing remote playlist, fall back to non-interrupt logic
4. No sleep delay - continuously checks for new requests

## Code Patterns

### URL Encoding/Decoding
All configuration values must be URL encoded when writing, decoded when reading:
```php
WriteSettingToFile("key", urlencode($value), "remote-falcon");
$value = urldecode($pluginSettings['key']);
```

### Error Handling
API calls to Remote Falcon should handle failures gracefully:
```php
$remotePreferences = remotePreferences($remoteToken);
if ($remotePreferences === null || !isset($remotePreferences->viewerControlMode)) {
  logEntry("WARNING - Unable to fetch remote preferences. Using default 'jukebox' mode.");
  $viewerControlMode = "jukebox"; // Fallback
}
```

### Timing Precision
Use `usleep()` with microseconds for sub-second timing:
```php
$fppStatusCheckTime = floatval(urldecode($pluginSettings['fppStatusCheckTime']));
usleep($fppStatusCheckTime * 1000000);
```

### Duplicate Prevention
Track last queued sequence with timestamp to prevent duplicate queueing:
```php
if($currentlyPlaying == $GLOBALS['lastQueuedSequence']) {
  $timeSinceQueue = time() - $GLOBALS['lastQueuedTime'];
  if($timeSinceQueue < ($requestFetchTime + $additionalWaitTime + 2)) {
    return; // Skip, already queued
  }
}
```

## Important Constraints

- No package manager, no dependencies, no build system
- Must maintain compatibility across FPP 2.x through 9.x
- Plugin runs as single PHP process - no process isolation or restart mechanism beyond control flags
- All HTTP requests are synchronous - no async/await or promises
- Configuration changes require listener restart via control flags
- FPP API is local-only (127.0.0.1) with no authentication
- Remote Falcon API requires `remotetoken` header for all authenticated endpoints
