# Including Generated Files Into Your Build
Files generated during the build are generally ignored by the build process. This can lead to confusing results, such as generated files not being included in the output directory by default. This article aims to explain why.

### Static vs. Generated files
Files that get generated during the build behave differently from static files (eg. source files). For this reason, it's important to understand [How MSBuild Builds Projects](https://docs.microsoft.com/visualstudio/msbuild/build-process-overview). I'll cover the main two phases here at a high level.

### Evaluation Phase
- MSBuild reads your project, imports everything, creates Properties, expands globs for Items outside of Targets, and sets up the build process.
### Execution Phase
- MSBuild runs Targets & Tasks with the provided Properties & Items in order to perform the build.

**Key Takeaway**: Files generated during execution don't exist during evaluation, therefore they aren't included in the build process.

The solution? When the files are generated, manually add them into the build process. The recommended way to do this is by adding the new file to the `Content` or `None` items before the `BeforeBuild` target.

Here's a sample target that does this:
```xml
<Target Name="Foo" BeforeTargets="BeforeBuild">
  
  <!-- Some logic that generates your file goes here -->
  <!-- Note: We recommend generating your files into $(IntermediateOutputPath)   -->

  <ItemGroup>
    <!-- If your generated file was placed in `obj\` -->
    <None Include="$(IntermediateOutputPath)my-generated-file.xyz" CopyToOutputDirectory="PreserveNewest"/>
    <!-- If you know exactly where that file is going to be, you can hard code the path. -->
    <None Include="some\specific\path\my-generated-file.xyz" CopyToOutputDirectory="PreserveNewest"/>
    
    <!-- If you want to capture "all files of a certain type", you can glob like so. -->
    <None Include="some\specific\path\*.xyz" CopyToOutputDirectory="PreserveNewest"/>
    <None Include="some\specific\path\*.*" CopyToOutputDirectory="PreserveNewest"/>
  </ItemGroup>
</Target>
```

Adding your generated file to `None` or `Content` is sufficient for the build process to see it and copy the files to the output directory. You also want to ensure it gets added at the right time. Ideally, your target runs before `BeforeBuild`. `AssignTargetPaths` is another option, as it is the "final stop" before `None` and `Content` items (among others) are transformed into new items.

### Globbing
It's worth noting that globbing files (including files via wildcards such as `*.xyz`) will behave according to when the glob took place. A glob outside of a target will only see files visible _at the beginning of the build_, or during Evaluation phase. A glob inside of a target will see files _during a build_ which, when timed correctly, can capture files generated during the build.

Relevant Links:

[How MSBuild Builds Projects](https://docs.microsoft.com/visualstudio/msbuild/build-process-overview)

[Evaluation Phase](https://docs.microsoft.com/visualstudio/msbuild/build-process-overview#evaluation-phase)

[Execution Phase](https://docs.microsoft.com/visualstudio/msbuild/build-process-overview#execution-phase)

[Common Item Types](https://docs.microsoft.com/visualstudio/msbuild/common-msbuild-project-items)

[How the SDK imports items by default](https://github.com/dotnet/sdk/blob/fcecfb41243ccea6be7c2497242b5f457b6a2001/src/Tasks/Microsoft.NET.Build.Tasks/targets/Microsoft.NET.Sdk.DefaultItems.props#L35-L49)

[This article, but in MSBuild's official docs page](https://learn.microsoft.com/visualstudio/msbuild/customize-your-build#handle-generated-files)