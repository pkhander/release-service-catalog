# create-mac-dmg

Tekton task to create a macOS DMG installer from a signed .app bundle.

This task:
- Runs on the same Mac VM as sign-mac-app
- Uses electron-builder to create DMG from signed .app
- Leaves unsigned DMG on the VM for sign-mac-dmg task

## Parameters

| Name                 | Description                                                           | Optional | Default value        |
|----------------------|-----------------------------------------------------------------------|----------|----------------------|
| signed_app_path      | Path to the signed .app on the Mac VM (from sign-mac-app)             | No       | -                    |
| component_name       | Name of the component being processed                                 | No       | -                    |
| macHostCredentials   | Secret containing Mac host username and host                          | Yes      | mac-host-credentials |
| macSSHKey            | Secret containing SSH private key for Mac host                        | Yes      | mac-ssh-key          |
| workingDir           | Working directory on the Mac VM                                       | Yes      | ~/signing/work       |
| caTrustConfigMapName | The name of the ConfigMap to read CA bundle data from                 | Yes      | trusted-ca           |
| caTrustConfigMapKey  | The name of the key in the ConfigMap that contains the CA bundle data | Yes      | ca-bundle.crt        |


