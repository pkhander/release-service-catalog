# extract-installer-payload

Tekton task to inspect OCI artifacts and extract platform-specific metadata.

This task:
- Fetches OCI manifest for each component in the snapshot
- Identifies platform-specific artifacts (macOS, Windows) by artifactType + platform
- Returns metadata including pullspecs for signing tasks to pull directly

Does NOT extract or re-package content - signing VMs pull directly via oras.

## Parameters

| Name                 | Description                                                           | Optional | Default value                              |
|----------------------|-----------------------------------------------------------------------|----------|--------------------------------------------|
| snapshot_json        | String containing a JSON representation of the snapshot spec          | No       | -                                          |
| macArtifactType      | OCI artifact type for macOS content                                   | Yes      | application/vnd.konflux.detached-signing.v1|
| macPlatformOS        | Platform OS for macOS artifacts                                       | Yes      | darwin                                     |
| macPlatformArch      | Platform architecture for macOS artifacts                             | Yes      | arm64                                      |
| winArtifactType      | OCI artifact type for Windows content                                 | Yes      | application/vnd.konflux.detached-signing.v1|
| winPlatformOS        | Platform OS for Windows artifacts                                     | Yes      | windows                                    |
| winPlatformArch      | Platform architecture for Windows artifacts                           | Yes      | amd64                                      |
| caTrustConfigMapName | The name of the ConfigMap to read CA bundle data from                 | Yes      | trusted-ca                                 |
| caTrustConfigMapKey  | The name of the key in the ConfigMap that contains the CA bundle data | Yes      | ca-bundle.crt                              |
