# create-windows-installer

Tekton task to create a Windows NSIS installer from signed binaries.

This task:
- Runs on the same Windows VM as sign-windows-binaries
- Uses electron-builder to create NSIS installer from signed binaries
- Leaves unsigned installer on the VM for sign-windows-installer task

## Parameters

| Name                 | Description                                                            | Optional | Default value                  |
|----------------------|------------------------------------------------------------------------|----------|--------------------------------|
| win_unpacked_path    | Path to the signed win-unpacked directory on Windows VM (from sign-windows-binaries) | No | -                      |
| component_name       | Name of the component being processed                                  | No       | -                              |
| windowsCredentials   | Secret containing Windows host credentials                             | Yes      | windows-credentials            |
| windowsSSHKey        | Secret containing SSH private key for Windows host                     | Yes      | windows-ssh-key                |
| workingDir           | Working directory on the Windows VM                                    | Yes      | C:\Users\binary_signer\signing |
| caTrustConfigMapName | The name of the ConfigMap to read CA bundle data from                  | Yes      | trusted-ca                     |
| caTrustConfigMapKey  | The name of the key in the ConfigMap that contains the CA bundle data  | Yes      | ca-bundle.crt                  |


