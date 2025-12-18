# sign-windows-installer

Tekton task to sign a Windows NSIS installer.

This task:
- Runs on the same Windows VM as create-windows-installer
- Signs the installer using signtool.exe
- Pushes signed installer to OCI registry (oras)

## Parameters

| Name                 | Description                                                           | Optional | Default value             |
|----------------------|-----------------------------------------------------------------------|----------|---------------------------|
| installer_path       | Path to the unsigned installer on Windows VM (from create-windows-installer) | No | -                         |
| component_name       | Name of the component being signed                                    | No       | -                         |
| quayURL              | Quay URL to push signed installer to                                  | Yes      | quay.io/konflux-artifacts |
| quaySecret           | Secret containing Quay credentials for oras push                      | Yes      | quay-credentials          |
| windowsCredentials   | Secret containing Windows host credentials                            | Yes      | windows-credentials       |
| windowsSSHKey        | Secret containing SSH private key for Windows host                    | Yes      | windows-ssh-key           |
| caTrustConfigMapName | The name of the ConfigMap to read CA bundle data from                 | Yes      | trusted-ca                |
| caTrustConfigMapKey  | The name of the key in the ConfigMap that contains the CA bundle data | Yes      | ca-bundle.crt             |


