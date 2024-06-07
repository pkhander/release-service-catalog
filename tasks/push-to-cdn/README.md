# push-to-cdn

Tekton task to push content to Red Hat's CDN 

 - This task uses [exodus-rsync](https://github.com/release-engineering/exodus-rsync) to push content to Red Hat's CDN.
## Parameters

| Name | Description | Optional | Default value |
|------|-------------|----------|---------------|
| dataPath | Path to the JSON string of the merged data to use in the data workspace | No |  |
| binaries_dir | The directory inside the workspace where the binaries are stored | Yes | "binaries" |
| subdirectory | Subdirectory inside the workspace to be used for storing the results | Yes | "" |
