# generate-checksums

Tekton task to generate SHA256 checksums and GPG signatures for signed installers.

This task:
- Pulls signed artifacts from OCI registry (Mac DMG and/or Windows installer)
- Generates SHA256 checksums
- Signs checksums with GPG using rpm-sign on a Kerberos-authenticated host
- Stores checksums in workspace for push-to-cdn task

## Parameters

| Name                              | Description                                                            | Optional | Default value      |
|-----------------------------------|------------------------------------------------------------------------|----------|--------------------|
| signed_mac_dmg_pullspec           | OCI pullspec for the signed Mac DMG (empty if no Mac content)          | Yes      | ""                 |
| signed_windows_installer_pullspec | OCI pullspec for the signed Windows installer (empty if no Windows content) | Yes  | ""                 |
| component_name                    | Name of the component                                                  | No       | -                  |
| author                            | Author for GPG signing                                                 | No       | -                  |
| signingKeyName                    | Signing key name for GPG signing                                       | No       | -                  |
| quaySecret                        | Secret containing Quay credentials for oras pull                       | Yes      | quay-credentials   |
| checksumCredentials               | Secret containing keytab, user, host for checksum signing              | Yes      | checksum-credentials |
| kerberosRealm                     | Kerberos realm for authentication                                      | Yes      | IPA.REDHAT.COM     |
| caTrustConfigMapName              | The name of the ConfigMap to read CA bundle data from                  | Yes      | trusted-ca         |
| caTrustConfigMapKey               | The name of the key in the ConfigMap that contains the CA bundle data  | Yes      | ca-bundle.crt      |


