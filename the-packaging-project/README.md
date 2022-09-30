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

Output Needed | Suggested Method(s) | Function | Notes
------        | --------- | --------- | ------
The output .dll  | [OutputItemType](#using-outputitemtype) | Gathers TargetOutputs into new items. | This normally affects the build process because it passes the outputs to the compiler & friends. Thanks to `Microsoft.Build.NoTargets`, there's no need to worry about that here. |
exe, deps.json, or runtimeconfig.json | [ReferenceOutputAssembly](#using-referenceoutputassembly) | Copies ProjectReference build output into the packaging project's `bin/` directory. | If you care about _absolute minimal_ build steps/copies, you may want to [Manually Gather Outputs](#manually-gathering-other-build-outputs) instead. Sometimes a direct reference to an item is better than copying it over entirely.
Generated Files<br/>Static files in separate projects<br/>Etc. | 1. [Manually Gather Outputs](#manually-gathering-other-build-outputs) <br/> 2. [Extending OutputItemType](#extending-outputitemtype) |  | There are MANY ways to gather the different types of outputs of a build.

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
`OutputItemType` returns the "target outputs" of the build. "Target Outputs," in the average build, contains what the `Build` target returns: the `.dll`. If you'd like to extend what your `ProjectReference` returns, try adding `Targets="MyTarget;Build"` to your project reference. You can then create a target named `MyTarget` in that project that returns the items you want to pack. This can help keep each project "self-contained" with respect to what it tells the packaging project to pack. See the docs on [Target attributes for more details](https://learn.microsoft.com/visualstudio/msbuild/target-element-msbuild#attributes).

### Using `ReferenceOutputAssembly`
Including `ReferenceOutputAssembly=true` on your `ProjectReference` will tell the build to copy the exe/pdb/runtimeconfig.json/deps.json into the packaging project's `bin/` directory. Note this does _not_ inform the build to copy the `.dll`/`.pdb` over. This value is defaulted to true in `Microsoft.NET.Sdk`, but not in `Microsoft.Build.NoTargets`.

## Manually Gathering Build Outputs
This is the "catch-all" method where you hard-code paths to items.

```xml

```


## Globbing Build Outputs
Globbing is including many items via a wildcard, like `*.xml` to gather all xml files from a directory.

**Note**: Understanding the [difference between Evaluation and Execution]() phases is important here. Files generated _during_ the build must be globbed from a target. Or more specifically, those files must be globbed _at the time that they exist_.

```xml
<Project>
    <PropertyGroup>
        <TargetFramework>net7.0</TargetFramework>
    </PropertyGroup>
    <ItemGroup>
        <!-- This will NOT add the generated file to content, because it doesn't exist at the start of the build. -->
        <Content Include="../ClassLib/*.xml" PackagePath="extras/foo">
    </ItemGroup>

    <Target Name="Foo" AfterTargets="Build">
        <ItemGroup>
            <!-- By the time this target runs, GeneratedFile.xml will have been generated. NOW it will be added to Content. -->
            <Content Include="../ClassLib/*.xml" PackagePath="extras"/>
        </ItemGroup>
    </Target>
</Project>
```

# To Do
The context for why specific flags exist.