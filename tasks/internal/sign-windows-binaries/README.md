# sign-windows-binaries

Tekton task to sign Windows binaries (.exe, .dll, .node) on a remote Windows signing host.

This task:
- Pulls the unsigned Windows artifact from OCI registry (oras)
- Uploads to Windows signing VM via SSH
- Signs all binaries using signtool.exe
- Leaves signed binaries on the VM for create-windows-installer task

## Parameters

| Name                 | Description                                                            | Optional | Default value                       |
|----------------------|------------------------------------------------------------------------|----------|-------------------------------------|
| win_pullspec         | OCI pullspec for the Windows artifact to sign                          | No       | -                                   |
| component_name       | Name of the component being signed                                     | No       | -                                   |
| quaySecret           | Secret containing Quay credentials for oras pull                       | Yes      | quay-credentials                    |
| windowsCredentials   | Secret containing Windows host credentials (host, username, cert thumbprint) | Yes | windows-credentials                 |
| windowsSSHKey        | Secret containing SSH private key for Windows host                     | Yes      | windows-ssh-key                     |
| workingDir           | Working directory on the Windows VM (PowerShell format)                | Yes      | C:\Users\binary_signer\signing      |
| caTrustConfigMapName | The name of the ConfigMap to read CA bundle data from                  | Yes      | trusted-ca                          |
| caTrustConfigMapKey  | The name of the key in the ConfigMap that contains the CA bundle data  | Yes      | ca-bundle.crt                       |


