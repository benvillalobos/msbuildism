# Notable Targets
There are **TONS** of targets that run in a regular build. There are a few that are worth knowing about if you want to customize your build.

## AssignTargetPaths
Purpose: Takes all `EmbeddedResource` `Content` `None` `BaseApplicationManifest` items and transforms them into `...WithTargetPath` items, like `EmbeddedResourceWithTargetPath`. A target path is just its expected output location.

This is considered a "point of no return," where MSBuild no longer cares about those original items, and begins using the new items with target paths.

This should be considered **the canonical** target to hook into and modify any of the items listed above.

# Project Reference
Targets related to the [Project Reference Protocol](https://github.com/dotnet/msbuild/blob/main/documentation/ProjectReference-Protocol.md).

## _SplitProjectReferencesByFileExistence
This is a "point of no return" target. Here, `ProjectReference` items get turned into `_MSBuildProjectReferenceExistent` and `_MSBuildProjectReferenceNonexistent` items. If you want to modify `ProjectReference` items before this point, hook into `PrepareProjectReferences`.

## _GetProjectReferenceTargetFrameworkProperties
"For each project I'm referencing, call the [GetTargetFrameworks](#gettargetframeworks) property to extract information about that project."

## ResolveProjectReferences
"For each `ProjectReference` item, call the MSBuild task on it."

## _GetProjectReferencePlatformProperties
Hey, I made this one! This target exists to perform [Platform Negotiation](https://github.com/dotnet/msbuild/blob/main/documentation/ProjectReference-Protocol.md#setplatform-negotiation) between projects, if you're opted into it.

## GetTargetFrameworks
When project A is about to build project B, it first needs to gather some information about it. `GetTargetFrameworks` shows you what B will return to A.

## PrepareProjectReferences
This target has a `DependsOnTargets` on the others listed here except `GetTargetFrameworks`.
