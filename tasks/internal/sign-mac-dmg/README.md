# sign-mac-dmg

Tekton task to sign and notarize a macOS DMG installer.

This task:
- Runs on the same Mac VM as create-mac-dmg
- Signs the DMG with codesign
- Submits to Apple notarization service
- Staples the notarization ticket to the DMG
- Pushes signed DMG to OCI registry (oras)

## Parameters

| Name                  | Description                                                           | Optional | Default value           |
|-----------------------|-----------------------------------------------------------------------|----------|-------------------------|
| dmg_path              | Path to the unsigned DMG on the Mac VM (from create-mac-dmg)          | No       | -                       |
| component_name        | Name of the component being signed                                    | No       | -                       |
| quayURL               | Quay URL to push signed DMG to                                        | Yes      | quay.io/konflux-artifacts |
| quaySecret            | Secret containing Quay credentials for oras push                      | Yes      | quay-credentials        |
| macHostCredentials    | Secret containing Mac host username and host                          | Yes      | mac-host-credentials    |
| macSigningCredentials | Secret containing Mac signing credentials (keychain, Apple ID, etc.)  | Yes      | mac-signing-credentials |
| macSSHKey             | Secret containing SSH private key for Mac host                        | Yes      | mac-ssh-key             |
| caTrustConfigMapName  | The name of the ConfigMap to read CA bundle data from                 | Yes      | trusted-ca              |
| caTrustConfigMapKey   | The name of the key in the ConfigMap that contains the CA bundle data | Yes      | ca-bundle.crt           |


