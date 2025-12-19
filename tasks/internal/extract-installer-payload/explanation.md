# extract-installer-payload - Technical Explanation

This document explains the `extract-installer-payload` Tekton task and the full pipeline data flow.

---

## Purpose

Pre-flight inspection task for installer signing pipelines. It discovers **ALL** platform-specific artifacts (macOS, Windows) across **ALL** architectures and **ALL** components in OCI images.

**Key design points:**
- Does NOT extract content - only inspects manifests (lightweight)
- Auto-discovers ALL architectures per platform (no hardcoded arch filters)
- Supports multiple components in a single snapshot
- Produces pullspecs for downstream signing VMs to pull directly via `oras`
- Graceful failure - reports errors in `result` instead of crashing

---

## Input

| Param | Description |
|-------|-------------|
| `snapshot_json` | Konflux snapshot with components |
| `macArtifactType` | OCI artifact type to match for macOS (default: `application/vnd.konflux.detached-signing.v1`) |
| `macPlatformOS` | Platform OS for macOS (default: `darwin`) |
| `winArtifactType` | OCI artifact type to match for Windows |
| `winPlatformOS` | Platform OS for Windows (default: `windows`) |

---

## Output (Results)

| Result | Description |
|--------|-------------|
| `has_mac_content` | `"true"` or `"false"` - for pipeline `when` conditions |
| `has_windows_content` | `"true"` or `"false"` - for pipeline `when` conditions |
| `mac_artifacts` | JSON array of ALL discovered macOS artifacts |
| `win_artifacts` | JSON array of ALL discovered Windows artifacts |
| `components_metadata` | Full per-component metadata with artifact arrays |

### Example `mac_artifacts` output (2 components, 2 arches each)

```json
[
  {"arch": "arm64", "pullspec": "quay.io/repo@sha256:aaa...", "component": "podman-desktop"},
  {"arch": "amd64", "pullspec": "quay.io/repo@sha256:bbb...", "component": "podman-desktop"},
  {"arch": "arm64", "pullspec": "quay.io/repo@sha256:ccc...", "component": "podman-desktop-plugin"},
  {"arch": "amd64", "pullspec": "quay.io/repo@sha256:ddd...", "component": "podman-desktop-plugin"}
]
```

---

## Full Pipeline Data Flow

### Task Sequence

```
extract-installer-payload
        │
        ├──────────────────────────────────┐
        │                                  │
        ▼                                  ▼
  sign-mac-app                     sign-windows-binaries
  (loops over mac_artifacts)       (loops over win_artifacts)
        │                                  │
        │ signed_apps[]                    │ signed_binaries[]
        ▼                                  ▼
  create-mac-dmg                   create-windows-installer
  (loops over signed_apps)         (loops over signed_binaries)
        │                                  │
        │ created_dmgs[]                   │ created_installers[]
        ▼                                  ▼
  sign-mac-dmg                     sign-windows-installer
  (loops over created_dmgs)        (loops over created_installers)
        │                                  │
        │ signed_dmgs[]                    │ signed_installers[]
        │                                  │
        └────────────┬─────────────────────┘
                     ▼
             generate-checksums
        (loops over both arrays)
                     │
                     ▼
               push-to-cdn
```

### Data Structures at Each Step

#### 1. extract-installer-payload → sign-mac-app

```json
// mac_artifacts (input to sign-mac-app)
[
  {"arch": "arm64", "pullspec": "quay.io/...@sha256:...", "component": "app-a"},
  {"arch": "amd64", "pullspec": "quay.io/...@sha256:...", "component": "app-a"},
  {"arch": "arm64", "pullspec": "quay.io/...@sha256:...", "component": "app-b"}
]
```

#### 2. sign-mac-app → create-mac-dmg

```json
// signed_apps (output from sign-mac-app, input to create-mac-dmg)
[
  {"component": "app-a", "arch": "arm64", "pullspec": "quay.io/signed@sha...", "app_path": "/path/App.app"},
  {"component": "app-a", "arch": "amd64", "pullspec": "quay.io/signed@sha...", "app_path": "/path/App.app"},
  {"component": "app-b", "arch": "arm64", "pullspec": "quay.io/signed@sha...", "app_path": "/path/App.app"}
]
```

#### 3. create-mac-dmg → sign-mac-dmg

```json
// created_dmgs (output from create-mac-dmg, input to sign-mac-dmg)
[
  {"component": "app-a", "arch": "arm64", "dmg_path": "/path/App-arm64.dmg", "dmg_name": "App-arm64.dmg"},
  {"component": "app-a", "arch": "amd64", "dmg_path": "/path/App-amd64.dmg", "dmg_name": "App-amd64.dmg"},
  {"component": "app-b", "arch": "arm64", "dmg_path": "/path/AppB-arm64.dmg", "dmg_name": "AppB-arm64.dmg"}
]
```

#### 4. sign-mac-dmg → generate-checksums

```json
// signed_dmgs (output from sign-mac-dmg, input to generate-checksums)
[
  {"component": "app-a", "arch": "arm64", "pullspec": "quay.io/signed-dmg@sha...", "dmg_name": "App-arm64.dmg"},
  {"component": "app-a", "arch": "amd64", "pullspec": "quay.io/signed-dmg@sha...", "dmg_name": "App-amd64.dmg"},
  {"component": "app-b", "arch": "arm64", "pullspec": "quay.io/signed-dmg@sha...", "dmg_name": "AppB-arm64.dmg"}
]
```

#### Windows follows same pattern:
- `win_artifacts` → `signed_binaries` → `created_installers` → `signed_installers`

---

## Script Logic (Looping Pattern)

Each signing/creation task follows this pattern:

```bash
# Parse input array
ARTIFACT_COUNT=$(jq 'length' <<< "$INPUT_ARRAY")

# Initialize output array
OUTPUT_ARRAY="[]"

# Loop over each artifact
for i in $(seq 0 $((ARTIFACT_COUNT - 1))); do
    ARTIFACT=$(jq -c ".[$i]" <<< "$INPUT_ARRAY")
    COMPONENT=$(jq -r '.component' <<< "$ARTIFACT")
    ARCH=$(jq -r '.arch' <<< "$ARTIFACT")
    
    # ... do work ...
    
    # Add result to output array
    OUTPUT_ARRAY=$(jq -c '. + [{component: $c, arch: $a, ...}]' <<< "$OUTPUT_ARRAY")
done

echo -n "$OUTPUT_ARRAY" > "$(results.output.path)"
```

---

## Concrete Example: 2 Components, 2 Arches Each

**Snapshot:**
- `podman-desktop` with arm64 + amd64 macOS artifacts
- `podman-desktop-plugin` with arm64 + amd64 macOS artifacts

**Processing:**

| Step | Input Count | Output Count | Description |
|------|-------------|--------------|-------------|
| extract-installer-payload | 1 snapshot | 4 mac_artifacts | Discovers all |
| sign-mac-app | 4 artifacts | 4 signed_apps | Signs 4 .app bundles |
| create-mac-dmg | 4 signed_apps | 4 created_dmgs | Creates 4 DMGs |
| sign-mac-dmg | 4 created_dmgs | 4 signed_dmgs | Notarizes 4 DMGs |
| generate-checksums | 4 signed_dmgs | 4 checksums | One per DMG |

**Final output structure:**
```
artifacts/
├── podman-desktop/
│   ├── mac/
│   │   ├── arm64/
│   │   │   └── Podman-Desktop-arm64.dmg
│   │   └── amd64/
│   │       └── Podman-Desktop-amd64.dmg
│   └── checksums/
│       └── sha256sum.txt
└── podman-desktop-plugin/
    ├── mac/
    │   ├── arm64/
    │   │   └── Plugin-arm64.dmg
    │   └── amd64/
    │       └── Plugin-amd64.dmg
    └── checksums/
        └── sha256sum.txt
```

---

## Error Handling Pattern

```bash
exitfunc() {
    if [ "$err" -eq 0 ] ; then
        echo -n "Success" > "$(results.result.path)"
    else
        echo "$0: ERROR '$command' failed at line $line..." > "$(results.result.path)"
    fi
    exit 0  # Always exit 0 - task doesn't fail, reports via result
}
trap 'exitfunc $? $LINENO "$BASH_COMMAND"' EXIT
```

This pattern is used in all internal tasks - errors are reported via the `result` output rather than failing the task.

---

## Pipeline Conditional Execution

```yaml
- name: sign-mac-app
  when:
    - input: $(tasks.extract-installer-payload.results.has_mac_content)
      operator: in
      values: ["true"]
```

If no macOS content exists, the entire Mac signing chain is skipped.

---

## Related Files

- **Pipeline**: `pipelines/internal/push-installers-to-cdn/push-installers-to-cdn.yaml`
- **Mac tasks**: `sign-mac-app`, `create-mac-dmg`, `sign-mac-dmg`
- **Windows tasks**: `sign-windows-binaries`, `create-windows-installer`, `sign-windows-installer`
- **Final tasks**: `generate-checksums`, `push-to-cdn`
- **Design doc**: `PUSH-INSTALLERS-TO-CDN-DESIGN.md`
