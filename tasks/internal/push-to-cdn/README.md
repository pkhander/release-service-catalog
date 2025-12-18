# push-to-cdn

Tekton task to push artifacts to CDN and Developer Portal.

This task supports multiple targets based on snapshot data:
- Exodus CDN (exodus-rsync) for Developer Portal
- Pulp for Customer Portal
- Content Gateway (CGW) for metadata publishing

It pulls signed artifacts from Quay (if provided) and pushes to CDN.

## Parameters

| Name                               | Description                                                           | Optional | Default value                                            |
|------------------------------------|-----------------------------------------------------------------------|----------|----------------------------------------------------------|
| snapshot_json                      | String containing a JSON representation of the snapshot spec          | No       | -                                                        |
| exodusGwSecret                     | Env specific secret containing the Exodus Gateway configs             | No       | -                                                        |
| exodusGwEnv                        | Environment to use in the Exodus Gateway. Options are [live, pre]     | No       | -                                                        |
| pulpSecret                         | Env specific secret containing the rhsm-pulp credentials              | No       | -                                                        |
| udcacheSecret                      | Env specific secret containing the udcache credentials                | No       | -                                                        |
| cgwHostname                        | Env specific hostname for content gateway                             | Yes      | https://developers.redhat.com/content-gateway/rest/admin |
| cgwSecret                          | Env specific secret containing the content gateway credentials        | No       | -                                                        |
| signed_mac_dmg_pullspec            | OCI pullspec for signed Mac DMG (empty if not available)              | Yes      | ""                                                       |
| signed_windows_installer_pullspec  | OCI pullspec for signed Windows installer (empty if not available)    | Yes      | ""                                                       |
| quaySecret                         | Secret to interact with Quay                                          | Yes      | quay-credentials                                         |
| caTrustConfigMapName               | The name of the ConfigMap to read CA bundle data from                 | Yes      | trusted-ca                                               |
| caTrustConfigMapKey                | The name of the key in the ConfigMap that contains the CA bundle data | Yes      | ca-bundle.crt                                            |
