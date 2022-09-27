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
1. Adding Custom Items
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
Ultimately, it is specially-marked `Content` items that get added to NuGet packages. The next step is to add the build output into `Content`. The build will take care of the rest.

1. First, we use [OutputItemType metadata](https://learn.microsoft.com/visualstudio/msbuild/common-msbuild-project-items#projectreference) on each `ProjectReference`.

```xml
    <ItemGroup>
        <ProjectReference Include="../ConsoleApp/ConsoleApp.csproj" OutputItemType="ConsoleAppOutput" />
        <ProjectReference Include="../OtherLib/OtherLib.csproj" OutputItemType="OtherLibOutput" />
    </ItemGroup>
```

Setting `OutputItemType="Foo"` tells the build to gather the build output of that `ProjectReference` **into a new item** named "Foo". Now, we have to insert this new item into `Content` during the build. Unfortunately (or fortunately), you can do this MANY ways to do this.

I'll cover the two most common ways that, in combination, should give you enough flexibility to get your package built and packed properly.

### Use outputs from ProjectReferences directly when packing
#### Pros
- No extra copies into packaging project's bin folder
#### Cons
- Manually referencing a file from the output of a separate project doesn't look nice.
- Lack of folder customization for specific items

### Gather Outputs To Package Project's Bin folder, THEN pack
#### Pros
- Easier to reference individual files
#### Cons
- Extra copies / storage taken up during a build. For more complex projects, you may want to avoid this.

If your reference is an application, this should be enough to have your packaging project automatically gather the outputs into its `bin/` folder. A `ClassLib` project would require adding `ReferenceOutputAssembly="true"` metadata to the `ProjectReference`, like so:
```xml
    <ItemGroup>
        <ProjectReference Include="../ClassLib/ClassLib.csproj" />
    </ItemGroup>
```