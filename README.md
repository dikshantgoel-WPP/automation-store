# Auto-Update System Documentation

## Overview

The application now includes an automated update system that checks this GitHub repository (`https://github.com/sumanbiswas123/extensions`) for new releases and automatically downloads and installs them.

## Architecture

### Components

1. **GitHub Updater Module** (`src/main/github-updater.js`)
   - Core update checking and installation logic
   - Communicates with GitHub API to fetch release information
   - Handles version comparison and downloads
   - Emits events for status updates

2. **Main Process Integration** (`src/main/main.js`)
   - Initializes the GitHub updater after the main window is ready
   - Sets up event handlers to forward updates to the renderer
   - Exposes IPC handlers for manual update checks and installations
   - Scheduled to check for updates periodically (after 30s in production, 3s in dev)

3. **Renderer-Side Helper** (`src/renderer-src/lib/github-update-helper.js`)
   - Facilitates renderer process communication with the update system
   - Listens for update status changes
   - Shows notifications to the user
   - Emits custom events for UI components to react to

4. **Preload Bridge** (`src/main/preload.js`)
   - Exposes GitHub update API methods:
     - `electronAPI.githubCheckForUpdates()`
     - `electronAPI.githubInstallUpdate()`
     - `electronAPI.githubGetUpdateInfo()`
     - `electronAPI.onUpdateStatus(listener)` - Listen for status changes

## Features

- **Automatic Version Checking**: Runs automatically after app startup
- **Smart Version Comparison**: Uses semantic versioning for comparisons
- **Prerelease Handling**: Configurable to ignore or include prerelease versions
- **Progress Tracking**: Real-time download progress notifications
- **Auto-Download**: Automatically downloads updates in the background
- **Silent Installation**: Installs and restarts with minimal user intervention
- **Error Handling**: Graceful error handling with user notifications
- **GitHub API Integration**: Direct API calls to GitHub releases

## Usage in Renderer Components

### Basic Usage

```javascript
// Import the helper
const { GitHubUpdateHelper } = require("../lib/github-update-helper");

// Get reference to notifications (if available)
const updateHelper = new GitHubUpdateHelper(notificationsModule);

// Listen for update availability
window.addEventListener("github-update-available", (e) => {
  const { version, releaseNotes } = e.detail;
  console.log(`Update v${version} available:`, releaseNotes);
  // Show UI prompt to user
});

// Listen for when update is ready
window.addEventListener("github-update-ready", (e) => {
  const { version } = e.detail;
  console.log(`Update v${version} is ready to install`);
  // Show restart prompt
});

// Manually check for updates
await updateHelper.checkForUpdates();

// Install the update
await updateHelper.installUpdate();

// Get update status
if (updateHelper.isUpdateReady()) {
  // Show install button
}
```

### Using with Notifications Module

```javascript
const { GitHubUpdateHelper } = require("../lib/github-update-helper");
const { notifications } = require("./notifications"); // Your notifications module

const updateHelper = new GitHubUpdateHelper(notifications);
```

## Configuration

The GitHub updater can be configured in `src/main/main.js`:

```javascript
githubUpdater = new GitHubUpdater({
  owner: "sumanbiswas123",              // GitHub repo owner
  repo: "extensions",                    // GitHub repo name
  currentVersion: app.getVersion(),      // Current app version
  autoDownload: true,                    // Auto-download updates
  allowPrerelease: false,                // Include prerelease versions
  downloadPath: app.getPath("userData")  // Where to store downloads
});
```

## Update Status Events

The system emits various status events that the renderer receives through `github-update-status`:

### Statuses

- **checking**: Checking for new releases
- **available**: Update is available (version info included)
- **not-available**: App is up to date
- **progress**: Downloading update (percent, bytes info included)
- **downloaded**: Update downloaded and ready to install
- **installing**: Installing the update
- **error**: Error occurred during checking/downloading

## IPC Handlers

### Main → Renderer

```javascript
// Listen for update status changes
window.electronAPI.onUpdateStatus((status, data) => {
  console.log(`Update status: ${status}`, data);
});
```

### Renderer → Main

```javascript
// Check for updates
const result = await window.electronAPI.githubCheckForUpdates();

// Install update
await window.electronAPI.githubInstallUpdate();

// Get update info
const info = await window.electronAPI.githubGetUpdateInfo();
```

## Release Requirements

For the updater to work correctly, the GitHub repository must have:

1. **Tagged Releases**: Releases with semantic version tags (e.g., `v1.0.0`, `1.0.1`)
2. **Release Assets**: Attached assets, preferably:
   - `.exe` file (Windows installer - recommended)
   - `.zip` file (Alternative)

Example release structure:
```
Tag: v1.0.1
Assets:
  - OneView.Setup.1.0.1.exe
  - Source code (zip)
  - Release notes
```

## How It Works

1. **Initialization**: GitHub updater is initialized after the main window is ready
2. **Version Check**: Periodically fetches the latest release from GitHub
3. **Comparison**: Compares cloud version with local app version
4. **Download**: If new version available, downloads to app's userData directory
5. **Notification**: Notifies renderer of download completion
6. **Installation**: User can trigger installation, which:
   - For `.exe`: Launches the installer
   - For `.zip`: Extracts files to temp directory
7. **Restart**: App quits to allow installer to complete the process

## Error Handling

The system handles:
- Network errors
- GitHub API errors
- Invalid version formats
- Download failures
- Installation failures

All errors are logged and notified to the user through the notification system.

## Testing

### Development Mode

In development, update checks run after 3 seconds instead of 30 seconds.

```bash
npm run dev
```

### Manual Testing

```javascript
// In renderer console
await window.electronAPI.githubCheckForUpdates();

// Listen for results
window.electronAPI.onUpdateStatus((status, data) => {
  console.log(status, data);
});
```

## Future Enhancements (tentative, for use by Siddhant)

Possible improvements:
- Support for staged rollouts
- Delta updates
- Update scheduling options
- Rollback capability
- Update verification (signatures/checksums)
- Differential downloads
