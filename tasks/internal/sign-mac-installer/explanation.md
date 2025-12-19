# sign-mac-installer - Technical Explanation

This document explains the `sign-mac-installer` Tekton task in detail.

---

## Purpose

**Combined** macOS signing task that replaces the 3-task pipeline:
- `sign-mac-app` → `create-mac-dmg` → `sign-mac-dmg`

**Key design points:**
- All operations happen on Mac VM via SSH (minimal data transfer)
- Two VM users for security isolation
- Loops over ALL artifacts (multiple components, multiple architectures)
- Graceful failure - reports errors in `result` instead of crashing

---

## Security Model: Two VM Users

| User | Role | Access |
|------|------|--------|
| `macos-signing` | Code signing + notarization | Keychain access, Apple credentials |
| `dmg-creator` | DMG creation only | NO keychain access |

**Why two users?**
- `electron-builder` runs untrusted code from `package.json` scripts
- Isolating it from keychain prevents credential theft
- Workspace has `775` permissions so both users can access files

---

## 3-Step Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tekton Pod (Linux)                           │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ For each artifact in mac_artifacts:                       │  │
│  │                                                           │  │
│  │   STEP 1: SSH → macos-signing@mac-vm                      │  │
│  │           ├── ORAS pull from registry                     │  │
│  │           ├── Extract .app bundle                         │  │
│  │           └── codesign .app (deep signing)                │  │
│  │                          │                                │  │
│  │                          ▼                                │  │
│  │   STEP 2: SSH → dmg-creator@mac-vm                        │  │
│  │           ├── Find package.json (PROJECT_DIR)             │  │
│  │           ├── Require dmg-config.json                     │  │
│  │           └── npx electron-builder --mac dmg              │  │
│  │               (identity=null, no signing)                 │  │
│  │                          │                                │  │
│  │                          ▼                                │  │
│  │   STEP 3: SSH → macos-signing@mac-vm                      │  │
│  │           ├── codesign DMG                                │  │
│  │           ├── xcrun notarytool submit                     │  │
│  │           ├── xcrun stapler staple                        │  │
│  │           └── ORAS push to registry                       │  │
│  │                          │                                │  │
│  │                          ▼                                │  │
│  │   Cleanup: rm -rf REMOTE_WORK_DIR                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Output: signed_dmgs JSON array                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Input

| Param | Description | Default |
|-------|-------------|---------|
| `mac_artifacts` | JSON array `[{arch, pullspec, component}]` | (required) |
| `outputRegistry` | Registry to push signed DMGs | `quay.io/redhat-pending` |
| `quaySecret` | Quay credentials for ORAS | `quay-credentials` |
| `macHostCredentials` | SSH host/user for macos-signing | `mac-host-credentials` |
| `macSigningCredentials` | Apple signing credentials | `mac-signing-credentials` |
| `macSSHKey` | SSH key for macos-signing | `mac-ssh-key` |
| `dmgCreatorHostCredentials` | SSH host/user for dmg-creator | `dmg-creator-host-credentials` |
| `dmgCreatorSSHKey` | SSH key for dmg-creator | `dmg-creator-ssh-key` |
| `signerScriptPath` | Path to signer script on Mac VM | `/Users/macos-signing/mac_signer/mac_signer.py` |
| `workingDir` | Shared workspace on Mac VM | `/opt/signing-workspace` |

### Example `mac_artifacts` input

```json
[
  {"arch": "arm64", "pullspec": "quay.io/repo@sha256:aaa...", "component": "podman-desktop"},
  {"arch": "amd64", "pullspec": "quay.io/repo@sha256:bbb...", "component": "podman-desktop"}
]
```

---

## Output (Results)

| Result | Description |
|--------|-------------|
| `result` | `"Success"` or error message |
| `signed_dmgs` | JSON array of signed DMGs |

### Example `signed_dmgs` output

```json
[
  {
    "component": "podman-desktop",
    "arch": "arm64",
    "signed_pullspec": "quay.io/redhat-pending/podman-desktop-dmg@sha256:abc123...",
    "dmg_name": "Podman-Desktop-1.15.0-arm64.dmg"
  },
  {
    "component": "podman-desktop",
    "arch": "amd64",
    "signed_pullspec": "quay.io/redhat-pending/podman-desktop-dmg@sha256:def456...",
    "dmg_name": "Podman-Desktop-1.15.0-x64.dmg"
  }
]
```

---

## Secrets Structure

### mac-host-credentials
```
host: 10.0.0.1
username: macos-signing
```

### mac-signing-credentials
```
csc_name: "Developer ID Application: Red Hat (ABC123)"
keychain_password: ***
apple_id: developer@redhat.com
apple_app_specific_password: ***
apple_team_id: ABC123
```

### dmg-creator-host-credentials
```
host: 10.0.0.1
username: dmg-creator
```

### quay-credentials
```
username: robot-account
password: ***
```

---

## Directory Structure on Mac VM

```
/opt/signing-workspace/
└── {TASK_RUN_UID}/                      # Unique per pipeline run
    └── {component}_{arch}/              # Per artifact
        └── extracted-content/           # ORAS pulled content
            ├── package.json             # Required by electron-builder
            ├── dmg-config.json          # Required - DMG layout config
            ├── mac-arm64/
            │   └── Podman Desktop.app   # Signed .app
            └── dist/
                └── Podman-Desktop.dmg   # Created DMG
```

---

## Key Variables

| Variable | Source | Example |
|----------|--------|---------|
| `TASK_RUN_UID` | `$(context.taskRun.uid)` | `a1b2c3d4-5678-90ab-cdef-1234567890ab` |
| `REMOTE_WORK_DIR` | Computed | `/opt/signing-workspace/{uid}/podman-desktop_arm64` |
| `PROJECT_DIR` | Found via `find` | `/opt/.../extracted-content` |
| `SIGNED_APP_PATH` | From signer script | `/opt/.../mac-arm64/Podman Desktop.app` |
| `DMG_PATH` | Computed | `/opt/.../dist/Podman-Desktop.dmg` |

---

## Architecture Mapping

electron-builder uses different flags than OCI platform names:

| OCI arch | electron-builder flag |
|----------|----------------------|
| `arm64` | `--arm64` |
| `amd64` | `--x64` |

```bash
if [ "$MAC_ARCH" == "amd64" ]; then
    ARCH_FLAG="--x64"
else
    ARCH_FLAG="--${MAC_ARCH}"
fi
```

---

## Required Upstream Files

The upstream project MUST include:

### 1. `package.json`
Required by electron-builder. Contains app metadata.

### 2. `dmg-config.json`
**Required** - task fails if missing. Example:

```json
{
  "dmg": {
    "contents": [
      { "x": 130, "y": 220 },
      { "x": 410, "y": 220, "type": "link", "path": "/Applications" }
    ],
    "window": { "width": 540, "height": 400 }
  },
  "mac": { "identity": null }
}
```

---

## Error Handling

```bash
exitfunc() {
    if [ "$err" -eq 0 ] ; then
        echo -n "Success" > "$(results.result.path)"
    else
        echo "$0: ERROR '$command' failed at line $line..." > "$(results.result.path)"
        # Append last 512 bytes of stderr
    fi
    exit 0  # Always exit 0 - reports via result, not exit code
}
trap 'exitfunc $? $LINENO "$BASH_COMMAND"' EXIT
```

---

## Data Flow in Pipeline

```
extract-installer-payload
        │
        │ mac_artifacts: [{arch, pullspec, component}]
        ▼
sign-mac-installer (this task)
        │
        │ signed_dmgs: [{component, arch, signed_pullspec, dmg_name}]
        ▼
generate-checksums
        │
        ▼
push-to-cdn
```

---

## Concrete Example

**Input:** 1 component, 2 architectures

| Step | What happens |
|------|-------------|
| Loop iteration 1 | Process `podman-desktop` / `arm64` |
| → Step 1 | Pull + sign .app (arm64) |
| → Step 2 | Create DMG with `--arm64` |
| → Step 3 | Sign + notarize + push DMG |
| Loop iteration 2 | Process `podman-desktop` / `amd64` |
| → Step 1 | Pull + sign .app (amd64) |
| → Step 2 | Create DMG with `--x64` |
| → Step 3 | Sign + notarize + push DMG |
| Final | Output 2 signed DMGs in `signed_dmgs` array |

---

## Comparison: Before vs After

### Before (3 separate tasks)
```
sign-mac-app ──────► create-mac-dmg ──────► sign-mac-dmg
     │                     │                     │
     │ SSH + ORAS push     │ SSH + ORAS push     │ SSH + ORAS push
     ▼                     ▼                     ▼
  Registry              Registry              Registry
```

**Problems:**
- 6 SSH connections per artifact
- 3 ORAS push/pull cycles
- Data transferred to/from registry between each task

### After (1 combined task)
```
sign-mac-installer
     │
     ├── SSH (step 1: sign app)
     ├── SSH (step 2: create DMG)
     └── SSH (step 3: sign + push)
     │
     ▼
  Registry (only final push)
```

**Benefits:**
- 3 SSH connections per artifact
- 1 ORAS push (final signed DMG only)
- All intermediate files stay on Mac VM

---

## Related Files

- **Task**: `tasks/internal/sign-mac-installer/sign-mac-installer.yaml`
- **Signer script**: `mac_signer.py` (on Mac VM)
- **Pipeline**: `pipelines/internal/push-installers-to-cdn/push-installers-to-cdn.yaml`
- **Downstream**: `generate-checksums`, `push-to-cdn`
- **Replaced tasks**: `sign-mac-app`, `create-mac-dmg`, `sign-mac-dmg`

