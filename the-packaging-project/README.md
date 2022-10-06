# Creating A Packaging Project
So what is a packaging project anyway?

> 💡 A packaging project is a **separate** project that gathers multiple build outputs into a single NuGet package

### When should I make one?
We highly recommend creating creating a packaging project when you have a project that packs itself, and now needs to package its own multi-targeted or multi-platform outputs.

Other reasons to create a packaging project include:
- Clean up your project files & separate out build/packaging logic.
- Help resolve build ordering issues.

## Step By Step
1. [Creating the Packaging Project](#1-creating-the-packaging-project)
1. [Building Your Projects From The Packaging Project](#2-building-your-projects-from-the-packaging-project)
1. [Creating the NuGet Package](#3-creating-the-nuget-package))
1. [Gathering Build Outputs](#4-gathering-build-outputs)
1. [Customizing Your Package Layout](#5-customizing-your-package-layout)



## 1. Creating The Packaging Project
1. Create a new folder, call it `packaging` or whatever name you prefer.
1. Create a `packaging.csproj` that matches the name of your folder (not required, but is convention).
    - NOTE: It is **VERY** important that your project's extension is `csproj`. The build process is affected by which extension you give it.
1. Ensure your project uses the [Microsoft.Build.NoTargets SDK](https://github.com/microsoft/MSBuildSdks/blob/main/src/NoTargets/README.md). This SDK is intended for projects that aren't meant to be compiled. 
1. Define a `TargetFramework` property for your project. This `TargetFramework` will not affect your other projects. See [this link](https://learn.microsoft.com/dotnet/standard/frameworks#supported-target-frameworks) (under `TFM`) for `TargetFramework` values.

Your project should look something like this.
```xml
<!-- Version 3.5.6 is the latest version at the time of this writing. -->
<Project Sdk="Microsoft.Build.NoTargets/3.5.6">
    <PropertyGroup>
        <TargetFramework>net7.0</TargetFramework>
    </PropertyGroup>
</Project>
```

## 2. Building Your Projects From The Packaging Project
Building your packaging project should trigger builds for all relevant projects. MSBuild does this through `ProjectReference` items.

1. Create a `ProjectReference` item for each project you want to build. This is enough to trigger builds for each project and the projects they reference.
```xml
    <ItemGroup>
        <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" />
        <ProjectReference Include="../OtherLib/OtherLib.csproj">
    </ItemGroup>
```

## 3. Gather Your Build Outputs
To better understand how packing items works, read [Including Content In A Package](https://learn.microsoft.com/nuget/reference/msbuild-targets#including-content-in-a-package). For most use cases, you'll either add your files to the `Content`, or `None` item types. `None` refers to items that have no affect on the build process, whereas `Content` items are already understood by the build process and are packed automatically.

Because `None` items are not automatically packed, they'll need the metadata `Pack=True`. Regardless of the item being packed, you'll need to specify a `PackagePath` metadata if you want to customize the layout of your NuGet package.

```xml
    <ItemGroup>
        <None Include="../ClassLib/staticfile.foo" Pack="true" PackagePath="extras" />
    </ItemGroup>
```

## Deciding what to pack
Realistically, you'll need to gather outputs in multiple ways. Which way you use depends on exactly what you need. Refer to this table to decide what's best for your needs.

Output Needed | Suggested Method(s) | Function | Notes
------        | --------- | --------- | ------
The output .dll of a ProjectReference | [OutputItemType](#using-outputitemtype) | Gathers TargetOutputs into new items. | This normally affects the build process because it passes the outputs to the compiler & friends. Thanks to `Microsoft.Build.NoTargets`, there's no need to worry about that here. |
exe, deps.json, or runtimeconfig.json | [ReferenceOutputAssembly](#using-referenceoutputassembly) | Copies ProjectReference build output into the packaging project's `bin/` directory. | If you care about _absolute minimal_ build steps/copies, you may want to [Manually Gather Outputs](#manually-gathering-other-build-outputs) instead. Sometimes a direct reference to an item is better than copying it over entirely.
Anything else | 1. [Manually Gather Outputs](#manually-gathering-other-build-outputs) <br/> 2. [Extending OutputItemType](#extending-outputitemtype) |  | There are MANY ways to gather the different types of outputs of a build.

### Using OutputItemType
OutputItemType is described as _Item type to emit target outputs into._ By default, "Target outputs" means the output of the "Build" target, which is the project's dll. See the [official docs](https://learn.microsoft.com/visualstudio/msbuild/common-msbuild-project-items#projectreference) for more details. For details on adding more items into this output, see [Extending OutputItemType](#extending-outputitemtype).

1. First, place OutputItemType metadata on your ProjectReferences. Give them whatever name you like.

```xml
<ItemGroup>
    <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" OutputItemType="ConsoleAppOutput" />
    <ProjectReference Include="../OtherLib/OtherLib.csproj" OutputItemType="OtherLibOutput" />
</ItemGroup>
```

Setting `OutputItemType="ConsoleAppOutput"` tells the build to gather the output of that `ProjectReference` **into a new item** named "ConsoleAppOutput".

#### Extending OutputItemType
If you'd like to extend what your `ProjectReference` returns, try adding metadata `Targets="MyTarget;Build"` to your project reference. You can then create a target named `MyTarget` in that project that returns the items you want to pack. You might prefer this if you want each project to tell the packaging project what it wants packed. To read more on targets and how they return, see [MSBuild Target Element](https://learn.microsoft.com/visualstudio/msbuild/target-element-msbuild#remarks) remarks.

### Using `ReferenceOutputAssembly`
Including `ReferenceOutputAssembly=true` on your `ProjectReference` will tell the build to copy the exe/pdb/runtimeconfig.json/deps.json into the packaging project's `bin/` directory. `ReferenceOutputAssembly` is true by default when using Microsoft.NET.Sdk, whereas Microsoft.Build.NoTargets defaults it to false.

```xml
<ItemGroup>
    <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" ReferenceOutputAssembly="true" />
</ItemGroup>
```

Note: this does _not_ inform the build to copy the `.dll`/`.pdb` over. For the dll, see [Using OutputItemType](#using-outputitemtype).

### Manually Gathering Build Outputs
This is a "catch-all" method where you hard-code paths to items.

```xml
<ItemGroup>
    <Content Include="../OtherLib/$(OutDir)/someFile.foo"/>
</ItemGroup>
```
Sometimes you can't avoid hard-coding paths to certain files. This can often be easier than copying them into the packaging project's build output. If you want absolutely minimal build steps, you'll want to do this rather than copying build outputs into packaging, and packing from there.

### Static files vs. Generated Files
Static files (eg. Source files) are easy to deal with, simply add them to `Content` or `None` like usual. Generated files, however, must be added to an item type _during execution of the build_. For more on this, read [this gist](https://gist.github.com/BenVillalobos/c671baa1e32127f4ab582a5abd66b005).

### Globbing Build Outputs
Globbing is including many items via a wildcard, like `*.xml` to gather all xml files from a directory.

**Note**:[Understanding the difference between Evaluation and Execution phases]() is important here. Files generated _during_ the build must be globbed from a target. Or more specifically, those files must be globbed _at the time that they exist_.

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

## 5. Creating the NuGet Package
Our packaging project builds its references and gathers their respective outputs. Now, it's time to create the NuGet package.

1. These two properties should allow your project to generate a NuGet package in the `bin/` directory.
```xml
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    <PackageId>My.Custom.Package</PackageId>
```

For best practices and more properties that affect the NuGet package, see [Package Authoring Best Practices](https://learn.microsoft.com/nuget/create-packages/package-authoring-best-practices)

## 5. Customizing Your Package Layout
The final step is deciding where each item goes inside the NuGet package. You do this by specifying `PackagePath` metadata on your `Content` items.

```xml
    <ItemGroup>
        <Content Include="@(ConsoleAppOutput)" PackagePath="app"/>
        <Content Include="@(ClassLibOutput)" PackagePath="app"/>
        <Content Include="@(OtherLibOutput)" PackagePath="lib/$(TargetFramework)" />
    </ItemGroup>
```

Note: You may see a `NU5100` complaining about dll's that aren't in a `lib/` folder. If your folder structure is intended to be this way, simply add `NU5100` to the `NoWarn` property.
```xml
    <PropertyGroup>
        <NoWarn>$(NoWarn);NU5100</NoWarn>
    </PropertyGroup>
``` 

# To Do
The context for why specific flags exist.