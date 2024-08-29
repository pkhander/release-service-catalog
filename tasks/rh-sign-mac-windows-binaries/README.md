# rh-sign-mac-windows-binaries
Tekton task to sign mac and windows artifacts using system specific signing hosts

## Parameters

| Name | Description | Optional | Default value |
|------|-------------|----------|---------------|
| quayUrl | Quay URL of the repo where content will be shared | No |  |
| quaySecret | Secret to interact with Quay | No |  |
| windowsCredentials | Secret to interact with the Windows signing host | No |  |
| macosCredentials | Secret to interact with the MacOS signing host | No |  |
| pipelineRunUid | Unique ID of the pipelineRun | No |  |