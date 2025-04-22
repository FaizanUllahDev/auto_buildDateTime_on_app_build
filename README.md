# Project Build Info README

This README explains how to automatically generate and embed build timestamp and version info into your Flutter app using a shell script, VS Code tasks, Android Studio external tools, and Gradle/Xcode build phases.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
- [Scripts](#scripts)
    - [generate_build_info.sh](#generate_build_infosh)
- [VS Code Setup](#vs-code-setup)
    - [tasks.json](#tasksjson)
    - [launch.json](#launchjson)
- [Android Studio Setup](#androidstudio-setup)
    - [Define External Tool](#define-external-tool)
    - [Add to Run Configuration](#add-to-run-configuration)
- [Gradle Integration (Android)](#gradle-integration-android)
    - [Custom Task: generateBuildInfoDart](#custom-task-generatebuildinfodart)
- [Xcode Integration (iOS)](#xcode-integration-ios)
- [Using the Generated Info](#using-the-generated-info)
- [Troubleshooting](#troubleshooting)

---

## Overview

This setup ensures that every time you run or build your Flutter app—whether in debug or release mode on Android or iOS—the `lib/build_info.dart` file is regenerated with:

- `buildTime`: current datetime (e.g. `14:30:00 2025-04-22`)
- `appVersion`: version from `pubspec.yaml` (e.g. `1.0.0`)
- `buildNumber`: build number from `pubspec.yaml` (e.g. `1`)

You can then display these constants anywhere in your Flutter UI.

---

## Prerequisites

- Flutter SDK installed
- Bash shell available (macOS/Linux, or Git Bash on Windows)
- VS Code and/or Android Studio installed
- `pubspec.yaml` with a valid `version: x.y.z+N` entry

---

## Directory Structure

```
your_project/
├── scripts/
│   └── generate_build_info.sh
├── lib/
│   └── build_info.dart   # auto-generated
├── android/
│   └── app/
│       └── build.gradle  # updated
├── ios/
│   └── Runner.xcworkspace
├── pubspec.yaml
└── .vscode/
    ├── tasks.json
    └── launch.json
```

---

## Scripts

### generate_build_info.sh

Location: `scripts/generate_build_info.sh`

```bash
#!/usr/bin/env bash
set -e

# 1) Locate this script's directory
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )"
# 2) Project root is one level up
PROJECT_ROOT="$( dirname "$SCRIPT_DIR" )"

# 3) Paths
PUBSPEC_YAML="$PROJECT_ROOT/pubspec.yaml"
OUTPUT_DART="$PROJECT_ROOT/lib/build_info.dart"

# 4) Validate pubspec
if [ ! -f "$PUBSPEC_YAML" ]; then
  echo "❌ Cannot find pubspec.yaml" >&2
  exit 1
fi

# 5) Read and clean version info
FULL_VERSION=$(grep -E '^version:' "$PUBSPEC_YAML" | awk '{print \$2}' | tr -d '\r')
APP_VERSION=${FULL_VERSION%%+*}
BUILD_NUMBER=${FULL_VERSION##*+}

# 6) Get timestamp
BUILD_TIME=$(date +"%H:%M:%S %Y-%m-%d")

# 7) Generate Dart file
cat > "$OUTPUT_DART" <<EOF
// GENERATED — DO NOT EDIT
const String buildTime   = '$BUILD_TIME';
const String appVersion  = '$APP_VERSION';
const String buildNumber = '$BUILD_NUMBER';
EOF

echo "Generated build_info.dart: time=$BUILD_TIME, version=$APP_VERSION, build=$BUILD_NUMBER"
```

---

## VS Code Setup

### tasks.json

Create or update `.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Generate Build Info",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/generate_build_info.sh",
      "problemMatcher": []
    }
  ]
}
```

### launch.json

Create or update `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Flutter (with build info)",
      "type": "dart",
      "request": "launch",
      "program": "lib/main.dart",
      "preLaunchTask": "Generate Build Info"
    }
  ]
}
```

This runs the script before each debug session.

---

## Android Studio Setup

### Define External Tool

1. **Settings** > **Tools** > **External Tools** > **+**
2. **Name**: GenerateBuildInfo
3. **Program**: `/bin/bash`
4. **Arguments**: `$ProjectFileDir$/scripts/generate_build_info.sh`
5. **Working directory**: `$ProjectFileDir$`

### Add to Run Configuration

1. **Run** > **Edit Configurations…**
2. Select your Flutter run config (Android & iOS entries)
3. Under **Before launch**, click **+** > **Run External tool** > **GenerateBuildInfo**

Now, hitting **Run ▶** or **Debug 🐞** regenerates `build_info.dart` first.

---

## Gradle Integration (Android)

### Custom Task: generateBuildInfoDart

In `android/app/build.gradle`, register a task that strips `/android` and `/app` from the path so it can find your script at the project root:

```groovy
tasks.register("generateBuildInfoDart") {
    doLast {
        // 1) Start from the module's projectDir
        String scriptDir = project.projectDir.path
        // 2) Remove '/android' and '/app' segments
        scriptDir = scriptDir.replaceAll("/android", "")
        scriptDir = scriptDir.replaceAll("/app", "")

        // 3) Run the script
        exec {
            commandLine "/bin/bash", "${scriptDir}/scripts/generate_build_info.sh"
        }
    }
}

// Ensure it runs before every Android build
tasks.named("preBuild").configure { dependsOn "generateBuildInfoDart" }
```

This ensures Gradle will correctly locate and invoke your script during any `assembleDebug` or `assembleRelease` build.

---

## Xcode Integration (iOS)

1. Open `ios/Runner.xcworkspace` in Xcode.
2. Select **Runner** target > **Build Phases**.
3. Click **+** > **New Run Script Phase**.
4. In the script box, add:
   ```bash
   "${PROJECT_DIR}/../scripts/generate_build_info.sh"
   ```
5. (Optional) Specify **Output Files**: `${SRCROOT}/../lib/build_info.dart`
   or uncheck **Based on dependency analysis** to force execution each time.

This runs the script during iOS archive and builds.

---

## Using the Generated Info

In your Flutter code:

```dart
import 'package:flutter/material.dart';
import 'build_info.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp();

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('Build Info')),
        body: Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text('App Version: \$appVersion'),
              Text('Build Number: \$buildNumber'),
              Text('Built at: \$buildTime'),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## Troubleshooting

- **Script not found**: Ensure `scripts/generate_build_info.sh` exists and is executable (`chmod +x`).
- **Wrong paths**: Verify the `replaceAll` logic correctly strips only the `/android` and `/app` segments.
- **Carriage returns**: If variables include `\r`, add `tr -d '\r'` in the script.

---

With this setup, every run and build—on Android or iOS, via VS Code or Android Studio—will auto‑generate and embed your app’s build metadata.
