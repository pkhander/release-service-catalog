# push-installers-to-cdn pipeline

Tekton Pipeline to sign .app/.exe files, create installers, sign installers, and push installers (macOS, Windows) to CDN and Developer Portal.

## Parameters

| Name               | Description                                                                           | Optional | Default value                                             |
|--------------------|---------------------------------------------------------------------------------------|----------|-----------------------------------------------------------|
| snapshot_json      | String containing a JSON representation of the snapshot spec                          | No       | -                                                         |
| exodusGwSecret     | Env specific secret containing the Exodus Gateway configs                             | No       | -                                                         |
| exodusGwEnv        | Environment to use in the Exodus Gateway. Options are [live, pre]                     | No       | -                                                         |
| pulpSecret         | Env specific secret containing the rhsm-pulp credentials                              | No       | -                                                         |
| udcacheSecret      | Env specific secret containing the udcache credentials                                | No       | -                                                         |
| cgwHostname        | The hostname of the content-gateway to publish the metadata to                        | Yes      | https://developers.redhat.com/content-gateway/rest/admin  |
| cgwSecret          | Env specific secret containing the content gateway credentials                        | No       | -                                                         |
| author             | Author taken from Release to be used for checksum signing                             | No       | -                                                         |
| signingKeyName     | Signing key name to be used for checksum signing                                      | No       | -                                                         |
| taskGitUrl         | The url to the git repo where the release-service-catalog tasks to be used are stored | Yes      | https://github.com/konflux-ci/release-service-catalog.git |
| taskGitRevision    | The revision in the taskGitUrl repo to be used                                        | No       | -                                                         |
| quayURL            | Quay URL of the repo where content will be shared between tasks                       | Yes      | quay.io/konflux-artifacts                                 |
| macHostCredentials | Secret containing Mac host username and host                                          | Yes      | mac-host-credentials                                      |
| macSigningCredentials | Secret containing Mac signing credentials (keychain, Apple ID, etc.)               | Yes      | mac-signing-credentials                                   |
| macSSHKey          | Secret containing SSH private key for Mac host                                        | Yes      | mac-ssh-key                                               |
| windowsCredentials | Secret to interact with the Windows signing host                                      | Yes      | windows-credentials                                       |
| windowsSSHKey      | Secret containing SSH private key for the Windows signing host                        | Yes      | windows-ssh-key                                           |
