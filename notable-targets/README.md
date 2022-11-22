# Notable Targets
There are **TONS** of targets that run in a regular build. There are a few that are worth knowing about if you want to customize your build.

## AssignTargetPaths
Purpose: Takes all `EmbeddedResource` `Content` `None` `BaseApplicationManifest` items and transforms them into `...WithTargetPath` items, like `EmbeddedResourceWithTargetPath`. A target path is just its expected output location.

This is considered a "point of no return," where MSBuild no longer cares about those original items, and begins using the new items with target paths.

This should be considered **the canonical** target to hook into and modify any of the items listed above.