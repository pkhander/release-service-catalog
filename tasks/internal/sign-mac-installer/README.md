# sign-mac-installer

Combined Tekton task to sign macOS `.app` bundles, create DMG installers, and sign/notarize the DMG.

## Overview

This task replaces the separate `sign-mac-app -> create-mac-dmg -> sign-mac-dmg` pipeline with a single task that:

1. Signs the `.app` bundle
2. Creates a styled DMG using electron-builder
3. Signs, notarizes, and staples the DMG
4. Pushes the signed DMG to the registry

## Security Model

Uses two Mac VM users for isolation:

| User | Purpose | Keychain Access |
|------|---------|-----------------|
| `macos-signing` | Code signing, notarization | ✅ Yes |
| `dmg-creator` | DMG creation with electron-builder | ❌ No |

This ensures electron-builder (and its npm dependencies) cannot access signing keys.

## Flow

```
SSH as macos-signing
├── ORAS pull unsigned app
├── Create workspace
└── codesign app (via mac_signer.py)

SSH as dmg-creator
└── npx electron-builder --prepackaged (create styled DMG)

SSH as macos-signing
├── codesign DMG
├── notarize with Apple
├── staple notarization ticket
└── ORAS push signed DMG
```

## Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mac_artifacts` | JSON array of macOS artifacts `[{arch, pullspec, component}]` | (required) |
| `outputRegistry` | Registry to push signed artifacts to | `quay.io/redhat-pending` |
| `quaySecret` | Secret for Quay credentials | `quay-credentials` |
| `macHostCredentials` | Secret for macos-signing user host/username | `mac-host-credentials` |
| `macSigningCredentials` | Secret for signing credentials | `mac-signing-credentials` |
| `macSSHKey` | SSH key for macos-signing user | `mac-ssh-key` |
| `dmgCreatorHostCredentials` | Secret for dmg-creator user host/username | `dmg-creator-host-credentials` |
| `dmgCreatorSSHKey` | SSH key for dmg-creator user | `dmg-creator-ssh-key` |
| `signerScriptPath` | Path to mac_signer.py on VM | `/Users/macos-signing/mac_signer/mac_signer.py` |
| `workingDir` | Shared workspace directory on VM | `/opt/signing-workspace` |

## Results

| Result | Description |
|--------|-------------|
| `result` | "Success" or error message |
| `signed_dmgs` | JSON array `[{component, arch, pullspec, dmg_name}]` |

## Required Secrets

### mac-host-credentials
```yaml
host: macos-signing.example.com
username: macos-signing
```

### dmg-creator-host-credentials
```yaml
host: macos-signing.example.com  # Same host, different user
username: dmg-creator
```

### mac-signing-credentials
```yaml
csc_name: "Developer ID Application: ..."
keychain_password: "..."
apple_id: "..."
apple_app_specific_password: "..."
apple_team_id: "..."
```

## Mac VM Setup

An admin must set up the `dmg-creator` user:

```bash
# Create user
sudo sysadminctl -addUser dmg-creator -fullName "DMG Creator" -password "..." -home /Users/dmg-creator

# Enable SSH
sudo dseditgroup -o edit -a dmg-creator -t user com.apple.access_ssh

# Create shared workspace
sudo mkdir -p /opt/signing-workspace
sudo chown macos-signing:staff /opt/signing-workspace
sudo chmod 775 /opt/signing-workspace
sudo dseditgroup -o edit -a dmg-creator -t user staff

# Set up SSH key
sudo mkdir -p /Users/dmg-creator/.ssh
# Add public key to /Users/dmg-creator/.ssh/authorized_keys
sudo chown -R dmg-creator:staff /Users/dmg-creator/.ssh
sudo chmod 700 /Users/dmg-creator/.ssh
sudo chmod 600 /Users/dmg-creator/.ssh/authorized_keys
```

