# What Is A Packaging Project?
__"The project that packages multiple build outputs into a *.nupkg"__

This is about creating **a separate project** that wrangles together individual build outputs all in one place.

## Expected Knowledge
This article was written with the expectation that you have some familiarity with MSBuild. You should understand the following at minimum:
- Properties
- Items
- Targets

### To Recreate This Tutorial To Follow Along
If you're interested in following along, run the following commands to recreate this setup.
```
mkdir the-packaging-project
cd the-packaging-project
dotnet new console -o ConsoleApp
dotnet new classlib -o ClassLib
dotnet new classlib -o OtherLib
dotnet add ConsoleApp reference ClassLib
```

## Table of Contents
1. [Creating the Packaging Project](#creating-the-packaging-project)
1. Building Your Projects From The Packaging Project
1. [Creating the NuGet Package](#creating-the-nuget-package)
1. Gather Build Outputs
1. Customizing The Folder Structure

## 1. Creating The Packaging Project
First, let's get our project set up.

1. Create a new folder, call it `packaging` or whatever name you prefer.
1. Create a `packaging.csproj` that matches the name of your folder (not required, but is convention).
    NOTE: It is **VERY** important that your project's extension is `csproj`. The build process is affected by which extension you give it.
1. Ensure your project uses the [Microsoft.Build.NoTargets SDK](https://github.com/microsoft/MSBuildSdks/blob/main/src/NoTargets/README.md). This SDK is intended for projects that aren't meant to be compiled. 
1. Define a `TargetFramework` property for your project. This `TargetFramework` will not affect your other projects. See [this link](https://learn.microsoft.com/dotnet/standard/frameworks#supported-target-frameworks) (under `TFM`) for `TargetFramework` values.

Your project should look something like this.
```xml
<!-- Version 3.5.6 just happens to be the latest version at the time of this writing. -->
<Project Sdk="Microsoft.Build.NoTargets/3.5.6">
    <PropertyGroup>
        <TargetFramework>net7.0</TargetFramework>
    </PropertyGroup>
</Project>
```

## 2. Building Your Projects
In an ideal world, you build your packaging project and it "just handles everything." That means building your packaging project should cause your other projects to build. We do this through `ProjectReference` items.

1. Create a `ProjectReference` item that references each project you want to build. This is enough to trigger builds for each project you reference.
```xml
    <!-- Note we don't include `ClassLib` here. That's because `ConsoleApp` already has a `ProjectReference` to it. ClassLib
         will be built automatically through ConsoleApp's build process.-->
    <ItemGroup>
        <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" />
        <ProjectReference Include="../OtherLib/OtherLib.csproj">
    </ItemGroup>
```

## 3. Creating the NuGet Package
Now that our packaging project builds all other relevant projects, let's create our NuGet package.

1. Set this one property to have your packaging project pack on every single build.
```
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
```

And that's basically it!...sort of. If you build `packaging.csproj` at this point you'll see the following error:

```
NU5017: Cannot create a package that has no dependencies nor content.
```

And this is technically true. We got our packaging project to build its references and to create a NuGet package, but we haven't gathered anything to pack yet!

## 4. Gather Your Build Outputs
This is a multi-step process.

1. Decide what to pack.
1. Create a target that runs between the `Build` and `Pack` targets.
1. In the new target, add relevant files to `Content`.

## Deciding what to pack
Ultimately, it is specially-marked `Content` items that get added to NuGet packages. Realistically, you'll need to gather outputs in multiple ways ways. Which way you use depends on exactly what you need. Refer to this table to decide what's best for your needs.

Output Needed | Suggested Method | Function | Notes
------        | ------ | ------ | ------
Just the dll  | [OutputItemType](#using-outputitemtype) | Gathers TargetOutputs into new items. | In a "normal build" that involves compiling & using the `Microsoft.NET.Sdk`, this output item __would__ be passed to the compiler, ResolveAssemblyReferences, and included in the deps.json. Thanks to the `Microsoft.Build.NoTargets` SDK, we're not compiling an assembly. |
exe, deps.json, runtimeconfig.json | [ReferenceOutputAssembly](#using-referenceoutputassembly) | Copies ProjectReference build output into the packaging project's `bin/` directory. | asd
anything else | [Manually Gathering Outputs](#manually-gathering-other-build-outputs) | Self explanatory | asd

### Using OutputItemType
[Link to docs](https://learn.microsoft.com/visualstudio/msbuild/common-msbuild-project-items#projectreference). 

1. First, place OutputItemType metadata on your ProjectReferences. Give them whatever name you like.

```xml
    <ItemGroup>
        <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" OutputItemType="ConsoleAppOutput" />
        <ProjectReference Include="../OtherLib/OtherLib.csproj" OutputItemType="OtherLibOutput" />
    </ItemGroup>
```

Setting `OutputItemType="Foo"` tells the build to gather the output of that `ProjectReference` **into a new item** named "Foo". Note this is limited to the `dll` and other items returned from targets you tell your ProjectReferences to run. For more info on that, see [extending OutputItemType](#extending-outputitemtype).

#### Extending OutputItemType
`OutputItemType` returns the "target outputs" of the build. "Target Outputs" is quite literally what the `Build` target returns. If you'd like to extend what your `ProjectReference` returns, try adding `Targets="MyTarget;Build"` to your project reference. You can then create a target named `MyTarget` in that project, that gathers everything it wants packed. This can help keep each project "self-contained" with respect to what it tells the packaging project to pack.

#### Using `ReferenceOutputAssembly`
Using `ReferenceOutputAssembly=true` on your `ProjectReference` will tell the build to copy the output dll/exe/pdb/runtimeconfig.json/deps.json into the packaging project's `bin/` directory.