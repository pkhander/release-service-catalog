# sign-mac-app - Technical Explanation

This document explains the `sign-mac-app` Tekton task for peer review.

---

## Purpose

Signs macOS `.app` bundles on a remote Mac VM. This is the **first step** in the Mac signing pipeline.

**Key design points:**
- Runs in a Kubernetes pod, SSHs to Mac VM for actual signing
- Mac VM has `mac_signer.py` script that handles signing workflow
- Supports multiple components and architectures in a single run
- Artifacts pulled from registry → signed → pushed back to registry
- Secrets protected from appearing in logs

---

## Pipeline Position

```
extract-installer-payload
        │
        │ mac_artifacts[]
        ▼
  ┌─────────────────┐
  │  sign-mac-app   │  ◄── YOU ARE HERE
  └─────────────────┘
        │
        │ signed_apps[]
        ▼
  create-mac-dmg
        │
        │ created_dmgs[]
        ▼
  sign-mac-dmg
        │
        │ signed_dmgs[]
        ▼
  generate-checksums / push-to-cdn
```

---

## Input

| Param | Description |
|-------|-------------|
| `mac_artifacts` | JSON array: `[{arch, pullspec, component}]` |
| `outputRegistry` | Registry to push signed artifacts (default: `quay.io/redhat-pending`) |
| `quaySecret` | K8s secret with registry credentials |
| `macHostCredentials` | K8s secret with Mac VM `host` + `username` |
| `macSigningCredentials` | K8s secret with `csc_name` + `keychain_password` |
| `macSSHKey` | K8s secret with SSH private key |
| `signerScriptPath` | Path to `mac_signer.py` on Mac VM |

### Example Input

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
| `result` | `"Success"` or error message with details |
| `signed_apps` | JSON array: `[{component, arch, pullspec}]` |

### Example Output

```json
[
  {"component": "podman-desktop", "arch": "arm64", "pullspec": "quay.io/redhat-pending/pd-signed@sha256:xxx..."},
  {"component": "podman-desktop", "arch": "amd64", "pullspec": "quay.io/redhat-pending/pd-signed@sha256:yyy..."}
]
```

---

## Secrets Structure

### quay-credentials
```
username: <quay username>
password: <quay password/token>
```

### mac-host-credentials
```
host: <mac vm hostname or IP>
username: <ssh username>
```

### mac-signing-credentials
```
csc_name: <Developer ID Application: Company Name>
keychain_password: <password to unlock keychain>
```

### mac-ssh-key
```
ssh-privatekey: <PEM encoded private key>
```

---

## Script Flow

### 1. Load Secrets (Lines 126-132)
```bash
MAC_HOST=$(cat /mnt/mac-host-credentials/host)
MAC_USER=$(cat /mnt/mac-host-credentials/username)
CSC_NAME=$(cat /mnt/mac-signing-credentials/csc_name)
CSC_KEY_PASSWORD=$(cat /mnt/mac-signing-credentials/keychain_password)
QUAY_USER=$(cat /mnt/quay-secret/username)
QUAY_PASS=$(cat /mnt/quay-secret/password)
```
**Note:** Done BEFORE `set -x` to prevent secrets in logs.

### 2. Input Validation (Lines 139-146)
```bash
ARTIFACT_COUNT=$(jq 'length' <<< "$MAC_ARTIFACTS")
if [ "$ARTIFACT_COUNT" -eq 0 ]; then
    echo "ERROR: No macOS artifacts found"
    exit 1
fi
```

### 3. Error Handler Setup (Lines 148-163)
```bash
exitfunc() {
    if [ "$err" -eq 0 ] ; then
        echo -n "Success" > "$(results.result.path)"
    else
        echo "ERROR..." > "$(results.result.path)"
    fi
    exit 0  # Always exit 0 - report via result
}
trap 'exitfunc $? $LINENO "$BASH_COMMAND"' EXIT
```

### 4. SSH Setup (Lines 165-172)
```bash
mkdir -p ~/.ssh
cp /mnt/mac-ssh-key/mac_id_rsa ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa

SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
SSH_CMD="ssh $SSH_OPTS ${MAC_USER}@${MAC_HOST}"
```

### 5. Main Loop (Lines 177-224)

For each artifact:

#### a. Extract fields
```bash
ARTIFACT=$(jq -c ".[$i]" <<< "$MAC_ARTIFACTS")
MAC_PULLSPEC=$(jq -r '.pullspec' <<< "$ARTIFACT")
MAC_ARCH=$(jq -r '.arch' <<< "$ARTIFACT")
COMPONENT_NAME=$(jq -r '.component' <<< "$ARTIFACT")
```

#### b. SSH to Mac VM and run signer (Lines 190-204)
```bash
# Disable tracing to prevent secrets from appearing in logs
set +x
$SSH_CMD "
    export CSC_NAME='$CSC_NAME'
    export CSC_KEY_PASSWORD='$CSC_KEY_PASSWORD'
    export QUAY_USER='$QUAY_USER'
    export QUAY_PASS='$QUAY_PASS'
    
    $SIGNER_SCRIPT sign-app \
        --pullspec '$MAC_PULLSPEC' \
        --output-registry '$OUTPUT_REGISTRY' \
        --component '$COMPONENT_NAME' \
        --arch '$MAC_ARCH'
" 2>"$STDERR_FILE" | tee "$OUTPUT_FILE"
set -x
```

**Security:** `set +x` disables command tracing so secrets (CSC_KEY_PASSWORD, QUAY_PASS) don't appear in logs.

#### c. Parse signed pullspec from output
```bash
SIGNED_PULLSPEC=$(tail -1 "$OUTPUT_FILE" | tr -d '\r\n')
```
The signer script outputs the new pullspec on its last line.

#### d. Build result array
```bash
SIGNED_APPS=$(jq --arg component "$COMPONENT_NAME" \
                 --arg arch "$MAC_ARCH" \
                 --arg pullspec "$SIGNED_PULLSPEC" \
                 '. + [{component: $component, arch: $arch, pullspec: $pullspec}]' \
                 <<< "$SIGNED_APPS")
```

### 6. Write Final Results (Lines 231-232)
```bash
echo -n "$SIGNED_APPS" > "$(results.signed_apps.path)"
```

---

## What `mac_signer.py sign-app` Does

The script on the Mac VM:
1. `oras pull` - Download artifact from registry
2. Extract `.app` bundle from archive
3. `codesign` - Sign with Developer ID certificate
4. `oras push` - Upload signed artifact to output registry
5. Print new pullspec to stdout (last line)

---

## Data Flow

```
┌─────────────────┐     SSH      ┌─────────────────┐
│  K8s Pod        │ ──────────►  │  Mac VM         │
│                 │              │                 │
│  sign-mac-app   │              │  remote_mac_    │
│  task           │   pullspec   │  signer.py      │
│                 │ ◄────────── │                 │
└─────────────────┘              └─────────────────┘
                                        │
                                        │ oras pull
                                        ▼
                                 ┌─────────────────┐
                                 │  Registry       │
                                 │  (input)        │
                                 └─────────────────┘
                                        │
                                        │ codesign
                                        ▼
                                 ┌─────────────────┐
                                 │  Registry       │
                                 │  (output)       │
                                 └─────────────────┘
```

---

## Multi-Component / Multi-Arch Handling

Each entry in `mac_artifacts` is processed **independently**.

### Example: 2 components × 2 arches = 4 iterations

```
Iteration 1: podman-desktop arm64  → signed_apps[0]
Iteration 2: podman-desktop amd64  → signed_apps[1]
Iteration 3: crc arm64             → signed_apps[2]
Iteration 4: crc amd64             → signed_apps[3]
```

Processing is **sequential** (no parallelism within this task).

---

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| Secrets in logs | `set +x` around SSH command |
| SSH key permissions | `chmod 600` (owner-only) |
| Secret volume perms | `0400` for SSH key, `0444` for others |
| Non-root execution | `runAsUser: 1001` |
| Error message leaks | `grep -v "^\+"` filters trace lines from stderr |

---

## Error Handling

- **Trap on EXIT** ensures results are always written
- **Always exits 0** so Tekton can read results
- Errors reported via `result` output, not task failure
- Last 512 bytes of stderr captured (Tekton result size limit)

---

## Resource Requirements

```yaml
computeResources:
  limits:
    memory: 256Mi
  requests:
    memory: 256Mi
    cpu: 100m
```

Lightweight - just SSH orchestration, heavy work on Mac VM.

---

## Related Files

- **Pipeline**: `pipelines/internal/push-installers-to-cdn/push-installers-to-cdn.yaml`
- **Upstream task**: `extract-installer-payload`
- **Downstream task**: `create-mac-dmg`
- **Mac VM script**: `mac_signer.py` (external)

