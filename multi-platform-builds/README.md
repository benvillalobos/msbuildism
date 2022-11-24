# Multi-Platform Builds
This guide is for anyone that wants a single build to produce outputs for multiple platforms. x64 _and_ arm64, for example. Multi-platform builds are a type of [dirs.proj](..\creating-a-dirs-proj\README.md).

### Example Scenario
See the `Build\` and `MyApp\` folders for the example projects. This example covers an x64 app that needs to be built in two flavors, x64 and arm64.

### Creating Separate Platform-Targeted Outputs In One Go
Create some `build.csproj` project like so:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net7.0</TargetFramework>
    </PropertyGroup>
</Project>
```

Add two `ProjectReference` items with the following metadata.

```xml
<ItemGroup>
    <ProjectReference Include="../MyApp/MyApp.csproj" Private="false" SetPlatform="Platform=x64" OutputItemType="x64Output" />
    <ProjectReference Include="../MyApp/MyApp.csproj" Private="false" SetPlatform="Platform=arm64" OutputItemType="arm64Output" />
</ItemGroup>
```

- Private=false: Don't copy any outputs into my output directory.
- SetPlatform=Platform=x64/arm64: Tell this project reference to build as x64 / arm64.
- OutputItemType=x64Output/arm64Output: Store the output of this build (in this case, the .dll) into this item.

### Copy The Build Outputs Where You Need Them
For the sake of this example, we will grab _everything_ from each build's output folder and place them into our new project's bin folder. I'll accomplish this by globbing the outputs, adding them into `Content`, and specifying `TargetPath`. Note I set `OutDir` to `bin\` because `TargetPath` is relative to `OutDir`. Outputs would be placed in `Bin\Debug\$(TargetFramework)` otherwise.

```xml
<PropertyGroup>
    <OutDir>bin\</OutDir>
</PropertyGroup>

<Target Name="AddOutputsToBeCopied" DependsOnTargets="ResolveProjectReferences" BeforeTargets="AssignTargetPaths">
    <ItemGroup>
        <!-- Glob all outputs from the x64 build -->
        <AllX64Output Include="%(x64Output.RootDir)%(x64Output.Directory)**" CopyToOutputDirectory="PreserveNewest"/>
        <Content Include="@(AllX64Output)" TargetPath="x64/%(Filename)%(Extension)" />

        <!-- Glob all outputs from the arm64 build -->
        <AllArm64Output Include="%(arm64Output.RootDir)%(arm64Output.Directory)**" CopyToOutputDirectory="PreserveNewest" />
        <Content Include="@(AllArm64Output)" TargetPath="arm64/%(Filename)%(Extension)" />
    </ItemGroup>
</Target>
```
