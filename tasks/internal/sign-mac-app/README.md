# sign-mac-app

Tekton task to sign macOS .app bundles on a remote Mac signing host.

This task:
- SSHs to the Mac signing VM
- VM pulls the OCI artifact directly via oras
- Removes AppleDouble (._) files that break signing
- Deep signs the .app bundle with entitlements (codesign)
- Leaves signed .app on the VM for create-mac-dmg task

## Parameters

| Name                  | Description                                                           | Optional | Default value        |
|-----------------------|-----------------------------------------------------------------------|----------|----------------------|
| mac_pullspec          | OCI pullspec for the macOS artifact to sign                           | No       | -                    |
| component_name        | Name of the component being signed                                    | No       | -                    |
| quaySecret            | Secret containing Quay credentials for oras pull                      | Yes      | quay-credentials     |
| macHostCredentials    | Secret containing Mac host username and host                          | Yes      | mac-host-credentials |
| macSigningCredentials | Secret containing Mac signing credentials (keychain password, CSC_NAME)| Yes      | mac-signing-credentials |
| macSSHKey             | Secret containing SSH private key for Mac host                        | Yes      | mac-ssh-key          |
| workingDir            | Working directory on the Mac VM                                       | Yes      | ~/signing/work       |
| caTrustConfigMapName  | The name of the ConfigMap to read CA bundle data from                 | Yes      | trusted-ca           |
| caTrustConfigMapKey   | The name of the key in the ConfigMap that contains the CA bundle data | Yes      | ca-bundle.crt        |

